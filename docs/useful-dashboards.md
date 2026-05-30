# 실무 추천 대시보드 모음

커뮤니티에서 검증된 대시보드를 레이어별로 정리합니다. ID로 Grafana.com에서 바로 Import할 수 있습니다.

---

## 빠른 Import

```bash
GRAFANA_URL="http://localhost:3000"
DS_PROMETHEUS="Prometheus"
DS_LOKI="Loki"

import_dashboard() {
  local ID=$1
  local DS_NAME=${2:-$DS_PROMETHEUS}
  curl -s -X POST "${GRAFANA_URL}/api/dashboards/import" \
    -u admin:admin \
    -H "Content-Type: application/json" \
    -d "{
      \"gnetId\": ${ID},
      \"overwrite\": true,
      \"inputs\": [
        {\"name\": \"DS_PROMETHEUS\", \"type\": \"datasource\", \"pluginId\": \"prometheus\", \"value\": \"${DS_NAME}\"},
        {\"name\": \"DS_LOKI\", \"type\": \"datasource\", \"pluginId\": \"loki\", \"value\": \"${DS_LOKI}\"}
      ]
    }" | jq -r '.importedUrl'
}

# 사용 예
import_dashboard 1860           # Node Exporter Full
import_dashboard 15757          # Kubernetes Cluster
```

---

## 1. 인프라 레이어

### Node Exporter Full — ID: 1860

가장 많이 쓰이는 노드 모니터링 대시보드입니다.

**주요 패널**
- CPU 사용률, iowait, steal
- 메모리 사용량 (MemFree, Cached, Buffers)
- 디스크 Read/Write IOPS, throughput
- 네트워크 수신/송신 bandwidth
- 시스템 Load Average, Context Switch

**실무 활용**
```
언제 보는가:
  - 노드가 느려졌을 때 CPU/iowait/memory swap 여부 확인
  - 디스크 병목인지 확인 (iowait 높고 IOPS 포화 상태면 EBS 타입 변경 고려)
  - 새벽 배치 작업 후 리소스 사용 패턴 분석
```

---

### Node Exporter / Nodes — ID: 8919 (경량 버전)

1860보다 가볍고 핵심만 담은 버전. 노드가 많을 때 유용합니다.

---

## 2. Kubernetes 레이어

### Kubernetes / Compute Resources / Cluster — ID: 15757

클러스터 전체를 한눈에 봅니다.

**주요 패널**
- 전체 CPU Request/Limit/Usage 비교
- 전체 메모리 Request/Limit/Usage 비교
- namespace별 CPU/메모리 사용량 테이블
- 파드 수 추이

**실무 활용**
```
언제 보는가:
  - 클러스터 용량 계획 수립 시 (Request 대비 실 사용량 괴리 확인)
  - 노드 추가가 필요한지 판단할 때
  - 비정상적으로 많은 리소스를 쓰는 namespace 파악
```

---

### Kubernetes / Compute Resources / Namespace (Pods) — ID: 15758

namespace 내 파드별 리소스 현황입니다.

**주요 패널**
- 파드별 CPU Usage vs Request vs Limit
- 파드별 메모리 Usage vs Request vs Limit
- CPU Throttling 비율 (이게 높으면 Request를 올리거나 코드 최적화 필요)

**실무 활용**
```
언제 보는가:
  - 특정 namespace의 리소스 최적화 작업 시
  - HPA 튜닝 전 실제 사용 패턴 파악
  - OOM Kill 원인 분석 (메모리 사용량이 Limit 근처인지 확인)
```

---

### Kubernetes / Compute Resources / Pod — ID: 15759

개별 파드의 상세 리소스 현황입니다.

---

### Kubernetes / Compute Resources / Workload — ID: 15760

Deployment/StatefulSet/DaemonSet 단위 뷰입니다.

**실무 활용**
```
언제 보는가:
  - 배포 후 새 버전의 리소스 사용량이 이전과 다른지 비교
  - Deployment의 레플리카 수와 실제 가용 레플리카 불일치 확인
```

---

### Kubernetes / Networking / Cluster — ID: 15761

클러스터 전체 네트워크 트래픽입니다.

**주요 패널**
- namespace별 네트워크 in/out
- 파드별 네트워크 사용량

---

### Kubernetes / Persistent Volumes — ID: 13646

PVC 디스크 사용량을 모니터링합니다.

**실무 활용**
```
언제 보는가:
  - StatefulSet(DB, Kafka, ES) 운영 시 디스크 용량 모니터링 필수
  - PVC 용량 확장 타이밍 결정
  - 갑자기 디스크가 줄어드는 워크로드 파악
```

---

## 3. 애플리케이션 레이어

### Spring Boot 2.1 Statistics — ID: 11378

Spring Boot Actuator + Micrometer 기반 대시보드입니다.

**주요 패널**
- JVM Heap/Non-Heap 메모리
- GC 시간, GC 빈도
- 스레드 수 (BLOCKED, WAITING, RUNNABLE)
- HTTP 요청 수, 에러율, 레이턴시
- HikariCP 커넥션 풀 사용량

**실무 활용**
```
언제 보는가:
  - Java 앱 메모리 누수 의심 시 → Heap Used가 계속 우상향하는지 확인
  - GC Pause가 잦아 레이턴시 스파이크 발생 시
  - DB 커넥션 풀 고갈 (HikariCP pool usage → 100%면 커넥션 부족)
  - 스레드 고갈 (Blocked 스레드 증가)
```

---

### JVM (Micrometer) — ID: 4701

Spring Boot 외 Micrometer를 사용하는 모든 JVM 앱에 활용합니다.

---

### Go Runtime / Overview — ID: 13240

Go 애플리케이션 런타임 지표입니다.

**주요 패널**
- Goroutine 수 (급증하면 goroutine leak 의심)
- GC STW 시간
- 힙 사용량
- CPU 프로파일

---

### Node.js Application Dashboard — ID: 11159

**주요 패널**
- Event Loop Lag (이게 높으면 CPU 집약적 작업이 blocking)
- Active Handles/Requests
- Heap Used/External
- HTTP 요청 통계

---

## 4. 미들웨어 레이어

### NGINX Ingress Controller — ID: 9614

**주요 패널**
- 초당 요청 수 (RPS)
- 4xx / 5xx 에러 비율
- 요청 레이턴시 P50/P95/P99
- Ingress별 트래픽 분포
- Active Connections

**실무 활용**
```
언제 보는가:
  - 외부 트래픽이 실제로 들어오는지 확인 (배포 후 트래픽 전환 검증)
  - 특정 Ingress에서만 에러가 발생하는지 좁히기
  - 502/504 에러 발생 시 업스트림 연결 실패인지 확인
```

---

### PostgreSQL Database — ID: 9628

**주요 패널**
- 활성 Connection 수 / Max Connection 대비 사용률
- QPS (Query per second)
- 트랜잭션 Commit/Rollback 비율
- Table Bloat, Index Bloat
- Deadlock, Lock Waits
- 캐시 히트율 (낮으면 shared_buffers 튜닝 필요)
- WAL 생성량 (replication lag 원인 파악)

**실무 활용**
```
언제 보는가:
  - DB 느려졌을 때 Connection 포화, Lock Wait 여부 확인
  - 슬로우 쿼리 증가로 인한 Connection 고갈 패턴 파악
  - pg_stat_activity와 함께 보면 실시간 쿼리 파악 가능
```

---

### Redis / Overview — ID: 11835 (또는 763)

**주요 패널**
- Connected Clients
- Commands per second
- Hit rate (낮으면 캐시 효율 저하)
- Used Memory / Max Memory 비율 → eviction 발생 여부
- Keyspace (key 수 추이)
- Replication lag

**실무 활용**
```
언제 보는가:
  - OOM으로 Redis 재시작 후 hit rate 회복 추이 모니터링
  - eviction 발생 시 maxmemory-policy 정책 점검
  - 특정 시간대 commands 급증 → 애플리케이션 쿼리 패턴 분석
```

---

### Kafka Overview — ID: 7589

**주요 패널**
- Under Replicated Partitions (0이어야 정상)
- Consumer Lag (쌓이면 consumer가 따라가지 못함)
- Bytes In/Out (초당)
- Messages In/Out (초당)
- Active Controller (1이어야 정상)

**실무 활용**
```
언제 보는가:
  - Consumer Lag 급증 → consumer 스케일 아웃 또는 파티션 추가 검토
  - Under Replicated Partitions > 0 → ISR 이탈한 브로커 확인
  - 브로커 재시작 후 리밸런싱 완료 여부 확인
```

---

## 5. 관찰 가능성 스택 레이어

### Grafana / Overview — ID: 3590

Grafana 자체 모니터링입니다.

**주요 패널**
- 활성 사용자 수
- 대시보드 로드 시간
- 데이터소스 쿼리 지연

---

### Prometheus 2.0 Stats — ID: 3662

**주요 패널**
- TSDB 블록 크기, 시계열 수
- Scrape 성공/실패율
- Rule 평가 시간
- 쿼리 레이턴시

**실무 활용**
```
언제 보는가:
  - Prometheus OOM 직전 → 활성 시계열 수 확인, retention 조정
  - 느린 대시보드 → 쿼리 레이턴시 확인, Recording Rule 추가 검토
  - Scrape 실패 → 특정 Job의 타겟 접근 불가 파악
```

---

### Loki / Chunks — ID: 10655 (또는 13407)

Loki 운영 모니터링입니다.

---

## 6. 실무 커스텀 조합 권장

### 최소 설치 세트 (처음 EKS 모니터링 시작할 때)

```bash
# 1순위: 인프라 기본
import_dashboard 1860    # Node Exporter Full (노드 리소스)
import_dashboard 15757   # Kubernetes Cluster (클러스터 전체)
import_dashboard 15758   # Kubernetes Namespace (namespace별)
import_dashboard 15759   # Kubernetes Pod (파드별)

# 2순위: 네트워크
import_dashboard 9614    # NGINX Ingress Controller

# 3순위: 애플리케이션 (언어에 맞게 선택)
import_dashboard 11378   # Spring Boot (Java)
import_dashboard 13240   # Go Runtime
```

### 완전 설치 세트

```bash
# 인프라
import_dashboard 1860    # Node Exporter Full
import_dashboard 8919    # Node Exporter / Nodes (경량)
import_dashboard 15757   # K8s Cluster
import_dashboard 15758   # K8s Namespace Pods
import_dashboard 15759   # K8s Pod
import_dashboard 15760   # K8s Workload
import_dashboard 15761   # K8s Networking Cluster
import_dashboard 13646   # K8s Persistent Volumes

# 미들웨어
import_dashboard 9614    # NGINX Ingress
import_dashboard 9628    # PostgreSQL
import_dashboard 11835   # Redis

# 앱 (선택)
import_dashboard 11378   # Spring Boot
import_dashboard 4701    # JVM Micrometer
import_dashboard 13240   # Go Runtime

# Observability stack
import_dashboard 3662    # Prometheus Stats
import_dashboard 3590    # Grafana Overview
```

---

## 7. 대시보드 찾는 법

필요한 대시보드가 위 목록에 없을 때:

```
1. https://grafana.com/grafana/dashboards/ 에서 검색
2. 검색어: "redis kubernetes" / "nginx ingress" / "spring boot" 등
3. 필터: Downloads 내림차순 정렬 (많이 쓰인 게 검증된 것)
4. 리뷰 확인: Last updated가 최근 1~2년 이내인지 확인

주의사항:
  - 메트릭 이름이 다를 수 있음 (exporter 버전에 따라)
  - 가져온 후 패널 쿼리를 실제 환경 메트릭 이름에 맞게 수정
  - 변수($job, $namespace 등)가 실제 값과 맞는지 확인
```

---

## 참고

- [Grafana 공식 대시보드](https://grafana.com/grafana/dashboards/)
- [Awesome Prometheus Alerts](https://samber.github.io/awesome-prometheus-alerts/)
