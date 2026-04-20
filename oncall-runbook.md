# 온콜 장애 대응 가이드

Grafana를 활용한 실전 장애 대응 절차입니다. 알림이 울렸을 때 어디서부터 시작해서 어떻게 원인을 좁혀가는지를 다룹니다.

---

## 1. 장애 대응 기본 절차

```
1. 알림 수신 (Slack / PagerDuty)
   ↓
2. Grafana Alerting → 어떤 rule이 firing인지 확인
   ↓
3. 해당 서비스의 대시보드 → 증상 확인 (무엇이 이상한가?)
   ↓
4. Explore → 메트릭 + 로그 + 트레이스 상관 분석 (왜 이상한가?)
   ↓
5. 원인 파악 → 조치 → 상태 확인
   ↓
6. 배포/조치 시점을 Annotation으로 기록
```

---

## 2. 증상별 체크리스트

### 증상 A: 5xx 에러율 급증

```
1. 에러 비율 확인
   Prometheus → rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])

2. 에러 시작 시점 파악 (배포와 겹치는지)
   대시보드 상단 Annotations 확인 (배포 이벤트가 에러 급증 직전?)

3. 어느 Pod에서 에러가 나는지
   sum by (pod) (rate(http_requests_total{status=~"5.."}[5m]))

4. 에러 로그 확인 (Loki)
   {namespace="default", app="my-app"} |= "ERROR" | json
   → status_code, error message, stack trace 확인

5. 에러가 특정 엔드포인트에 집중되는지
   sum by (handler, status) (rate(http_requests_total[5m]))

6. DB 연결 오류인지 확인
   {app="my-app"} |= "connection refused" OR |= "too many connections"

7. 업스트림 서비스 문제인지
   up{job=~"downstream-service.*"} == 0
```

**흔한 원인 → 조치**
| 원인 | 확인 방법 | 조치 |
|------|---------|------|
| 배포로 인한 버그 | Annotation 확인 | 이전 버전으로 롤백 |
| DB 커넥션 고갈 | HikariCP metrics 또는 pg_stat_activity | 커넥션 풀 크기 조정, 슬로우 쿼리 종료 |
| 메모리 부족 OOM | 파드 재시작 + `reason=OOMKilled` | 메모리 Limit 상향 또는 누수 수정 |
| 업스트림 서비스 장애 | upstream 서비스의 `up` 메트릭 | 서킷브레이커 fallback 확인 |

---

### 증상 B: 서비스 응답 없음 (타임아웃)

```
1. Pod이 Running인지 확인
   kube_pod_status_phase{namespace="default", phase="Running"}
   → 파드가 0이면 배포 문제 또는 OOM

2. Pod 재시작 여부
   increase(kube_pod_container_status_restarts_total{namespace="default"}[30m]) > 0

3. Ready 상태 파드 수 확인
   kube_deployment_status_replicas_ready / kube_deployment_spec_replicas

4. CrashLoopBackOff 파드
   kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}

5. 파드 로그 확인 (Loki)
   {namespace="default", app="my-app"} | json
   → 시작 실패 원인, 환경 변수 누락, 설정 오류 등

6. CPU/메모리 한도 확인
   container_cpu_cfs_throttled_seconds_total (CPU throttling)
   container_memory_working_set_bytes vs container_spec_memory_limit_bytes

7. Readiness Probe 실패 여부
   kube_pod_container_status_ready == 0
   → 로그에서 "readiness probe failed" 검색
```

---

### 증상 C: 응답 속도 저하 (레이턴시 급증)

```
1. P99 레이턴시 추이 확인 (언제부터 느려졌나)
   histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

2. 특정 엔드포인트에만 느린지
   sum by (handler) (rate(http_request_duration_seconds_sum[5m]))
   / sum by (handler) (rate(http_request_duration_seconds_count[5m]))

3. CPU 병목인지 확인
   rate(container_cpu_usage_seconds_total{pod=~"my-app.*"}[5m])
   → CPU throttling 있는지: container_cpu_cfs_throttled_seconds_total

4. 메모리 압박 → GC 지연인지
   # JVM의 경우
   jvm_gc_pause_seconds_sum → GC 시간이 갑자기 증가하는지
   jvm_memory_used_bytes{area="heap"} → Heap이 한계에 가까운지

5. DB 쿼리 슬로우인지 (로그에서 확인)
   {app="my-app"} |= "slow query" OR |= "query took"
   또는 pg_stat_activity에서 long-running query 확인

6. 외부 API 호출 지연인지 (Trace로 확인)
   Explore → Tempo → 느린 트레이스 선택 → span별 시간 확인
   → 어느 span이 가장 오래 걸렸는지 파악

7. 트래픽 급증으로 인한 포화 상태인지
   rate(http_requests_total[5m]) 의 갑작스러운 증가 확인
```

---

### 증상 D: 메모리 OOM / CrashLoopBackOff

```
1. OOM Kill 확인
   kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}

2. 메모리 사용 추이 확인 (재시작 전까지)
   container_memory_working_set_bytes{pod=~"my-app.*"}
   → 우상향하다가 갑자기 끊기면 OOM

3. 메모리 Limit 확인
   container_spec_memory_limit_bytes{pod=~"my-app.*"}
   → 사용량이 Limit의 90% 이상이면 OOM 임박

4. JVM Heap 분석 (Java 앱)
   jvm_memory_used_bytes{area="heap"} vs jvm_memory_max_bytes{area="heap"}
   → Heap이 Max에 붙어있으면 GC Full 후 OOM

5. 재시작 직전 로그 확인 (Loki)
   {pod="my-app-xxx"} | json
   → "OutOfMemoryError", "Cannot allocate memory" 등 확인

6. 누수 패턴 (메모리가 조금씩 계속 증가)
   container_memory_working_set_bytes[24h] → 지속적 우상향이면 누수
```

**즉시 조치**
```bash
# 메모리 Limit 임시 상향 (긴급 조치)
kubectl set resources deployment my-app \
  -n default \
  --limits=memory=2Gi

# 파드 재시작 (임시 방편)
kubectl rollout restart deployment/my-app -n default
```

---

### 증상 E: 디스크 가득 참

```
1. 노드별 디스크 사용률 확인
   (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100

2. 언제 가득 찰지 예측
   predict_linear(node_filesystem_free_bytes{mountpoint="/"}[6h], 24 * 3600) < 0
   → 24시간 내에 가득 찰 노드 감지

3. 어느 PVC가 큰지
   kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes

4. 어느 파드가 로컬 디스크를 많이 쓰는지
   container_fs_usage_bytes{namespace="default"}

5. 로그 디스크인지 (Loki) — ingestion rate 확인
   sum(rate({namespace="default"}[5m])) by (app)
   → 특정 앱이 로그를 폭발적으로 쏟고 있는지 확인
```

---

### 증상 F: 특정 노드 이상

```
1. 노드 상태 확인
   kube_node_status_condition{condition="Ready", status="true"} == 0
   → NotReady 노드 있는지

2. 노드 CPU/메모리 포화
   (1 - avg by (node) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) * 100
   node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100

3. 해당 노드에서 실행 중인 파드 목록
   kube_pod_info{node="ip-10-0-1-100.ap-northeast-2.compute.internal"}

4. 노드 네트워크 이상 (패킷 드롭)
   rate(node_network_receive_drop_total[5m]) > 0

5. 시스템 로그 확인 (노드 레벨)
   {__host__="ip-10-0-1-100"} | json
   또는 kubectl describe node <node-name>
```

---

## 3. 배포 이벤트 Annotation

배포 직후 이상이 발생했는지 빠르게 파악하려면 배포 시점을 그래프에 표시해야 합니다.

### Grafana Annotation API로 배포 기록

```bash
# CI/CD 파이프라인에서 배포 완료 시 호출
GRAFANA_URL="http://grafana.monitoring.svc.cluster.local"
GRAFANA_TOKEN="${GRAFANA_API_TOKEN}"

curl -s -X POST "${GRAFANA_URL}/api/annotations" \
  -H "Authorization: Bearer ${GRAFANA_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"dashboardUID\": \"my-app-overview\",
    \"time\": $(date +%s)000,
    \"tags\": [\"deployment\", \"my-app\"],
    \"text\": \"my-app v1.2.3 배포 완료 (by ${CI_USER})\"
  }"
```

### Prometheus 기반 Annotation (Grafana Dashboard 설정)

```
Dashboard Settings → Annotations → Add annotation query

Name: Deployments
Data source: Prometheus
Query:
  changes(kube_deployment_status_observed_generation{namespace="$namespace"}[2m]) > 0
Step: 60s
Color: blue
```

→ 배포가 있을 때마다 그래프에 **파란 수직선**이 자동으로 표시됩니다.

---

## 4. 장애 상황 공유 — Snapshot

팀원이나 경영진에게 현재 상황을 공유할 때 Snapshot을 활용합니다.

```
대시보드 상단 → Share (공유 아이콘) → Snapshot → Publish

→ 로그인 없이 접근 가능한 읽기 전용 링크 생성
→ 유효 기간 설정 가능 (1시간, 1일, 7일)
```

```bash
# Grafana API로 Snapshot 생성
curl -s -X POST http://admin:admin@localhost:3000/api/snapshots \
  -H "Content-Type: application/json" \
  -d '{
    "dashboard": { "id": null },
    "expires": 3600,
    "name": "2026-04-14 02:30 장애 상황"
  }' | jq '.url'
```

---

## 5. Explore Split View로 RCA 수행

루트 원인 분석(RCA) 시 메트릭과 로그를 동시에 봅니다.

```
Explore → Split 버튼 클릭

왼쪽 (Prometheus):
  histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{app="my-app"}[1m]))
  → 레이턴시 스파이크 시점 확인

오른쪽 (Loki):
  {namespace="default", app="my-app"} | json | level = "error"
  → 동일 시점의 에러 로그 확인
```

시간 범위를 맞추고 스파이크 시점의 로그를 확인합니다:
1. 그래프에서 이상 시점 드래그 → 해당 구간으로 zoom in
2. 로그에서 해당 시간대 에러 메시지 확인
3. Trace ID 발견 시 Tempo에서 트레이스 분석

---

## 6. 자주 쓰는 장애 분석 쿼리 북마크

Explore → Query history에 저장해두면 빠르게 재사용할 수 있습니다.

```promql
# 네임스페이스별 에러율
sum by (namespace, job) (rate(http_requests_total{status=~"5.."}[5m]))
/ sum by (namespace, job) (rate(http_requests_total[5m]))

# 재시작 중인 파드
rate(kube_pod_container_status_restarts_total[15m]) * 900 > 3

# CPU throttling 심한 파드
rate(container_cpu_cfs_throttled_seconds_total[5m])
/ rate(container_cpu_cfs_periods_total[5m]) > 0.25

# 메모리 Limit 90% 초과
container_memory_working_set_bytes{container!=""}
/ container_spec_memory_limit_bytes{container!=""} > 0.9

# PVC 85% 초과
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.85

# NotReady 노드
kube_node_status_condition{condition="Ready", status="true"} == 0
```

```logql
# 최근 5분 에러 로그 (앱 전체)
{namespace="default"} |= "ERROR" | json | line_format "{{.time}} [{{.level}}] {{.message}}"

# OOM kill 패턴
{namespace="default"} |= "OOMKilled" OR |= "OutOfMemoryError"

# DB 커넥션 관련 에러
{namespace="default"} |= "connection refused" OR |= "too many connections" OR |= "connection pool"

# 스택 트레이스 포함 에러 (Java)
{namespace="default"} |= "Exception" | json
```

---

## 참고

- [Google SRE Book - Incident Management](https://sre.google/sre-book/managing-incidents/)
- [Grafana OnCall](https://grafana.com/docs/oncall/latest/)
