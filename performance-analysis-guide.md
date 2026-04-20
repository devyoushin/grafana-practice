# 성능 분석 실전 가이드

Grafana에서 성능 문제의 원인을 찾는 실전 절차입니다. 증상에서 시작해서 병목 지점을 좁혀가는 방법을 다룹니다.

---

## 1. 성능 분석 기본 순서

```
1. 레이턴시 스파이크 시점 특정 (언제 시작됐나?)
   ↓
2. 영향 범위 확인 (특정 파드? 특정 엔드포인트? 전체?)
   ↓
3. 병목 유형 파악 (CPU? 메모리/GC? DB? 네트워크? 외부 API?)
   ↓
4. Trace로 스팬별 시간 확인 (어느 계층에서 지연 발생?)
   ↓
5. 로그에서 상세 원인 확인
```

---

## 2. 레이턴시 스파이크 분석

### 2-1. 언제 시작됐는지 파악

```promql
# P99 레이턴시 추이 (1분 단위)
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket{job="my-app"}[1m]))
)
```

→ 시간 범위를 이상 시점 포함하여 설정. 배포 Annotation과 겹치는지 확인.

### 2-2. 특정 엔드포인트에 집중되는지

```promql
# 엔드포인트별 P99 레이턴시
histogram_quantile(0.99,
  sum by (handler, le) (rate(http_request_duration_seconds_bucket{job="my-app"}[5m]))
)
```

→ 특정 handler의 레이턴시만 높으면 해당 로직에 집중.

### 2-3. 특정 파드에 집중되는지

```promql
# 파드별 평균 응답 시간
sum by (pod) (rate(http_request_duration_seconds_sum{job="my-app"}[5m]))
/
sum by (pod) (rate(http_request_duration_seconds_count{job="my-app"}[5m]))
```

→ 특정 파드만 느리면 그 파드의 노드 문제 또는 재시작 직후 워밍업 상태.

---

## 3. CPU 병목 분석

### CPU 사용률 및 Throttling 확인

```promql
# 파드별 CPU 사용률
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="default", container!=""}[5m]))

# CPU Throttling 비율 (이게 0이 아니면 CPU Limit이 너무 낮음)
sum by (pod, container) (
  rate(container_cpu_cfs_throttled_seconds_total{namespace="default"}[5m])
)
/
sum by (pod, container) (
  rate(container_cpu_cfs_periods_total{namespace="default"}[5m])
)
```

**Throttling이 높을 때 조치**:
```bash
# CPU Limit 상향 (임시)
kubectl set resources deployment my-app \
  --namespace default \
  --requests=cpu=500m \
  --limits=cpu=2000m

# 또는 values.yaml 수정 후 배포
```

### 노드 레벨 CPU 확인

```promql
# 노드별 CPU 사용률 (전체)
(1 - avg by (node, instance) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
)) * 100

# 노드별 iowait (디스크/네트워크 대기)
avg by (node, instance) (rate(node_cpu_seconds_total{mode="iowait"}[5m])) * 100

# iowait가 높으면 CPU 자체보다 I/O 병목
```

---

## 4. 메모리 / GC 분석

### 메모리 압박 확인

```promql
# 메모리 Limit 대비 사용률
container_memory_working_set_bytes{namespace="default", container!=""}
/
container_spec_memory_limit_bytes{namespace="default", container!=""}

# 메모리 사용량 추이 (누수 패턴: 지속 증가)
container_memory_working_set_bytes{pod=~"my-app.*", container="my-app"}
```

**누수 패턴 파악**:
- 재시작 없이 메모리가 계속 증가하면 누수 의심
- 재시작 후 낮아졌다가 다시 오르는 패턴 → 누수 + OOM Kill 반복

### JVM GC 분석 (Spring Boot / Java)

```promql
# GC 발생 빈도 (초당 횟수)
sum by (gc) (rate(jvm_gc_pause_seconds_count{app="my-app"}[5m]))

# GC 평균 시간
sum by (gc) (rate(jvm_gc_pause_seconds_sum{app="my-app"}[5m]))
/
sum by (gc) (rate(jvm_gc_pause_seconds_count{app="my-app"}[5m]))

# Heap 사용률 (Old Gen이 꽉 차면 Full GC 빈발)
jvm_memory_used_bytes{app="my-app", area="heap"}
/
jvm_memory_max_bytes{app="my-app", area="heap"}

# Full GC (STW) 발생 횟수 → 이게 많으면 레이턴시 스파이크 원인
rate(jvm_gc_pause_seconds_count{app="my-app", action="end of major GC"}[5m])
```

**GC 문제 징후**:
```
P99 레이턴시가 불규칙하게 스파이크 → Full GC STW
Heap Used가 Max에 근접 → GC 빈발 + 결국 OOM
GC 시간 합계 / 전체 시간 > 5% → GC overhead 과다
```

**로그에서 GC 관련 확인**:
```logql
{app="my-app"} |= "GC overhead" OR |= "java.lang.OutOfMemoryError"

# Heap dump 트리거 여부
{app="my-app"} |= "HeapDumpOnOutOfMemoryError"
```

---

## 5. 데이터베이스 병목 분석

### 응답 시간과 DB 쿼리 시간 비교

```promql
# 앱 전체 P99 레이턴시
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="my-app"}[5m]))

# DB 쿼리 P99 시간 (앱에서 DB 호출 시간을 메트릭으로 기록한 경우)
histogram_quantile(0.99, rate(db_query_duration_seconds_bucket{job="my-app"}[5m]))
```

앱 레이턴시 ≈ DB 쿼리 시간이면 → DB가 병목.

### PostgreSQL 지표 확인

```promql
# 활성 커넥션 수 / max_connections 비율
pg_stat_database_numbackends / pg_settings_max_connections

# 커넥션이 max에 가까우면 새 요청이 대기 → 앱 레이턴시 급증

# 슬로우 쿼리 수 (5초 이상)
rate(pg_stat_activity_max_tx_duration{state="active"}[5m]) > 5

# Lock 대기 중인 쿼리 수
pg_locks_count{mode="ExclusiveLock"} > 0

# 캐시 히트율 (낮으면 디스크 I/O 증가)
pg_stat_database_blks_hit
/ (pg_stat_database_blks_hit + pg_stat_database_blks_read)
```

**DB 병목 시 Loki 로그 확인**:
```logql
# 앱에서 DB 타임아웃 로그
{app="my-app"} |= "timeout" | json
  | message =~ "(?i)(query|database|connection).*timeout"

# 슬로우 쿼리 로그 (PostgreSQL log_min_duration_statement 설정 시)
{job="postgresql"} |= "duration:" | regexp `duration: (?P<ms>\d+\.\d+) ms`
  | ms > 1000
  | line_format `{{.ms}}ms: {{.query}}`
```

### Redis 병목 확인

```promql
# Redis 명령 처리 지연
redis_commands_duration_seconds_total / redis_commands_total

# 메모리 사용률 (eviction 발생 여부)
redis_memory_used_bytes / redis_memory_max_bytes

# eviction 발생 수 (급증하면 캐시 무효화 → DB 부하 증가)
rate(redis_evicted_keys_total[5m])

# Hit rate 저하
rate(redis_keyspace_hits_total[5m])
/ (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
```

---

## 6. 네트워크 병목 분석

```promql
# 파드 간 네트워크 에러 (패킷 드롭)
rate(container_network_transmit_errors_total{namespace="default"}[5m]) > 0
rate(container_network_receive_errors_total{namespace="default"}[5m]) > 0

# 노드 네트워크 대역폭 포화
rate(node_network_transmit_bytes_total{device!~"lo|veth.*"}[5m])

# TCP 연결 상태 (CLOSE_WAIT 누적은 커넥션 누수)
node_netstat_Tcp_CurrEstab     # 현재 ESTABLISHED 수
node_sockstat_TCP_tw           # TIME_WAIT 수 (너무 많으면 포트 고갈)
```

**Istio 사용 시 서비스 간 레이턴시**:
```promql
# 서비스 간 P99 레이턴시
histogram_quantile(0.99,
  sum by (source_app, destination_app, le) (
    rate(istio_request_duration_milliseconds_bucket[5m])
  )
) / 1000

# 어떤 서비스 호출에서 지연이 발생하는지 파악
```

---

## 7. Distributed Trace로 병목 계층 파악

가장 효율적인 방법입니다. 슬로우 트레이스를 찾아 span별 시간을 확인합니다.

```
Explore → Tempo → Search

Duration: > 1s           ← 1초 이상 걸린 트레이스
Service: my-app
Tags: http.status_code=200  ← 에러가 아닌데 느린 요청
```

트레이스를 열면:
```
[total: 1.2s]
  └── my-app: /api/orders  (50ms)
       ├── auth-service: validate_token  (10ms)   ← 정상
       ├── db: SELECT orders  (1100ms)             ← 병목!
       └── cache: get_user_profile  (5ms)          ← 정상
```

→ DB span이 1100ms → DB 쿼리 최적화 또는 인덱스 추가.

---

## 8. 병목 유형별 빠른 진단표

| 증상 | 확인 지표 | 원인 | 조치 |
|------|---------|------|------|
| 레이턴시 불규칙 스파이크 | JVM GC pause time | Full GC STW | Heap 크기 증가, GC 튜닝 |
| 레이턴시 지속적 상승 | DB 커넥션 수, lock 대기 | DB 커넥션 고갈 / 슬로우 쿼리 | 커넥션 풀 증가, 쿼리 최적화, 인덱스 추가 |
| CPU 높은데 레이턴시 높음 | CPU throttling 비율 | CPU Limit 너무 낮음 | Limit 상향 또는 코드 최적화 |
| CPU 낮은데 레이턴시 높음 | iowait, DB 응답 시간 | 디스크/네트워크 I/O 대기 | 스토리지 타입 변경(gp3), 캐싱 추가 |
| 특정 시간대만 느림 | 그 시간의 RPS | 트래픽 집중 | HPA 스케일아웃 튜닝 |
| 배포 직후 느림 | 신규 파드 레이턴시 | JVM 워밍업 / 캐시 cold start | Readiness Probe 조정, 워밍업 요청 추가 |
| Redis hit rate 급감 | redis evicted keys | 메모리 부족으로 eviction | maxmemory 상향, 덜 중요한 키 TTL 단축 |
| 외부 API 호출 느림 | Trace span 시간 | 외부 API 지연 | 타임아웃 설정, 서킷브레이커 적용 |

---

## 9. 실무 성능 분석 Explore 쿼리 세트

**Split View 구성** (메트릭 + 로그 동시에)

왼쪽 (Prometheus):
```promql
# 레이턴시 + CPU throttling 동시에
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="my-app"}[1m]))
```

오른쪽 (Loki):
```logql
# 동시에 느린 요청 로그 확인
{app="my-app"} | json | duration_ms > 1000
  | line_format `{{.pod}} {{.method}} {{.path}} {{.duration_ms}}ms`
```

→ 레이턴시 스파이크 구간에서 어떤 요청이 느렸는지 바로 확인.

---

## 참고

- [Brendan Gregg - USE Method](https://www.brendangregg.com/usemethod.html)
- [Google SRE Book - Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Grafana Tempo - Traces](https://grafana.com/docs/tempo/latest/)
