# 실무 PromQL 가이드

실무에서 자주 쓰는 PromQL 패턴을 모아놓은 레퍼런스입니다.

---

## 1. rate / irate / increase 차이

| 함수 | 특징 | 적합한 상황 |
|------|------|-----------|
| `rate()` | 범위 내 평균 증가율. 스파이크가 완만해짐 | 대시보드 그래프, 일반적인 RPS 표시 |
| `irate()` | 마지막 두 샘플의 순간 증가율. 스파이크가 그대로 | 빠른 변화를 민감하게 잡을 때 |
| `increase()` | 범위 내 총 증가량 | 특정 시간 동안 발생한 총 이벤트 수 |

```promql
# 5분 평균 RPS (부드러운 그래프)
rate(http_requests_total[5m])

# 순간 RPS (스파이크 민감)
irate(http_requests_total[5m])

# 1시간 동안 발생한 총 에러 수
increase(http_requests_total{status=~"5.."}[1h])
```

> **`$__rate_interval` 사용을 권장합니다.** Grafana가 현재 시간 범위에 맞게 자동 계산해줍니다.
>
> ```promql
> rate(http_requests_total[$__rate_interval])
> ```

---

## 2. Kubernetes 리소스

### CPU

```promql
# 파드별 CPU 사용량 (Cores)
sum by (pod) (
  rate(container_cpu_usage_seconds_total{
    namespace="$namespace",
    container!="",
    container!="POD"
  }[$__rate_interval])
)

# CPU Request 대비 사용률
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="$namespace", container!=""}[$__rate_interval]))
/
sum by (pod) (kube_pod_container_resource_requests{namespace="$namespace", resource="cpu", container!=""})

# 노드별 CPU 사용률 (%)
(
  1 - avg by (node) (
    rate(node_cpu_seconds_total{mode="idle"}[$__rate_interval])
  )
) * 100

# CPU Throttling 비율
sum by (pod, container) (
  rate(container_cpu_cfs_throttled_seconds_total{namespace="$namespace"}[$__rate_interval])
)
/
sum by (pod, container) (
  rate(container_cpu_cfs_periods_total{namespace="$namespace"}[$__rate_interval])
)
```

### 메모리

```promql
# 파드별 메모리 사용량 (working set)
sum by (pod) (
  container_memory_working_set_bytes{
    namespace="$namespace",
    container!="",
    container!="POD"
  }
)

# 메모리 Limit 대비 사용률
sum by (pod) (container_memory_working_set_bytes{namespace="$namespace", container!=""})
/
sum by (pod) (kube_pod_container_resource_limits{namespace="$namespace", resource="memory", container!=""})

# OOM Kill 발생 수
increase(kube_pod_container_status_last_terminated_reason{reason="OOMKilled", namespace="$namespace"}[1h])

# 노드별 메모리 사용률
(
  1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
) * 100
```

### 파드 상태

```promql
# 파드 재시작 수 (1시간)
sum by (pod, container) (
  increase(kube_pod_container_status_restarts_total{namespace="$namespace"}[1h])
)

# 현재 Pending 파드 목록
kube_pod_status_phase{namespace="$namespace", phase="Pending"} == 1

# CrashLoopBackOff 파드
kube_pod_container_status_waiting_reason{namespace="$namespace", reason="CrashLoopBackOff"} == 1

# 가용 레플리카 수 / 총 레플리카 수
kube_deployment_status_replicas_available{namespace="$namespace"}
/
kube_deployment_spec_replicas{namespace="$namespace"}

# 파드 Ready 비율
sum by (deployment) (kube_deployment_status_replicas_available{namespace="$namespace"})
/
sum by (deployment) (kube_deployment_spec_replicas{namespace="$namespace"})
```

---

## 3. HTTP 서비스 지표 (RED)

```promql
# ── Rate ──────────────────────────────────────────────────────
# 초당 요청 수 (전체)
sum(rate(http_requests_total{namespace="$namespace"}[$__rate_interval]))

# 서비스별 RPS
sum by (job) (rate(http_requests_total{namespace="$namespace"}[$__rate_interval]))

# 경로별 RPS
sum by (handler) (rate(http_requests_total{namespace="$namespace"}[$__rate_interval]))


# ── Errors ────────────────────────────────────────────────────
# 5xx 에러율
sum(rate(http_requests_total{namespace="$namespace", status=~"5.."}[$__rate_interval]))
/
sum(rate(http_requests_total{namespace="$namespace"}[$__rate_interval]))

# 4xx + 5xx 합산 에러율
sum(rate(http_requests_total{namespace="$namespace", status=~"[45].."}[$__rate_interval]))
/
sum(rate(http_requests_total{namespace="$namespace"}[$__rate_interval]))

# 서비스별 에러율 (테이블용)
sum by (job, status) (rate(http_requests_total{namespace="$namespace"}[$__rate_interval]))


# ── Duration ──────────────────────────────────────────────────
# P50 / P95 / P99 레이턴시
histogram_quantile(0.50, sum by (le) (rate(http_request_duration_seconds_bucket{namespace="$namespace"}[$__rate_interval])))
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{namespace="$namespace"}[$__rate_interval])))
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{namespace="$namespace"}[$__rate_interval])))

# 서비스별 P99 (여러 선으로)
histogram_quantile(0.99,
  sum by (job, le) (rate(http_request_duration_seconds_bucket{namespace="$namespace"}[$__rate_interval]))
)
```

---

## 4. 디스크 / 네트워크

```promql
# 노드 디스크 사용률
(
  1 - (node_filesystem_avail_bytes{mountpoint="/", fstype!="tmpfs"}
       / node_filesystem_size_bytes{mountpoint="/", fstype!="tmpfs"})
) * 100

# 파드별 디스크 Read/Write (초당 bytes)
sum by (pod) (rate(container_fs_reads_bytes_total{namespace="$namespace"}[$__rate_interval]))
sum by (pod) (rate(container_fs_writes_bytes_total{namespace="$namespace"}[$__rate_interval]))

# 노드 네트워크 수신/송신 (초당 bytes)
rate(node_network_receive_bytes_total{device!~"lo|veth.*"}[$__rate_interval])
rate(node_network_transmit_bytes_total{device!~"lo|veth.*"}[$__rate_interval])

# 파드 네트워크 수신/송신
sum by (pod) (rate(container_network_receive_bytes_total{namespace="$namespace"}[$__rate_interval]))
sum by (pod) (rate(container_network_transmit_bytes_total{namespace="$namespace"}[$__rate_interval]))
```

---

## 5. 집계 함수 활용

```promql
# 상위 5개 CPU 사용 파드
topk(5,
  sum by (pod) (
    rate(container_cpu_usage_seconds_total{namespace="$namespace", container!=""}[$__rate_interval])
  )
)

# 메모리 사용량 기준 정렬 (테이블 패널)
sort_desc(
  sum by (pod) (container_memory_working_set_bytes{namespace="$namespace", container!=""})
)

# 전체 클러스터 CPU 가용량 대비 사용률
sum(rate(container_cpu_usage_seconds_total{container!=""}[$__rate_interval]))
/
sum(kube_node_status_capacity{resource="cpu"})

# namespace별 메모리 합계
sum by (namespace) (container_memory_working_set_bytes{container!=""})
```

---

## 6. 시간 범위 활용

```promql
# 1일 전과 현재 RPS 비교
rate(http_requests_total[5m])
vs
rate(http_requests_total[5m] offset 1d)

# 전주 대비 현재 에러율
(rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]))
/
(rate(http_requests_total{status=~"5.."}[5m] offset 7d) / rate(http_requests_total[5m] offset 7d))
```

---

## 7. 알림용 PromQL 패턴

```promql
# ── 에러율 임계값 ──────────────────────────────────────────────
(
  sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum by (job) (rate(http_requests_total[5m]))
) > 0.05

# ── 파드 가용성 임계값 ─────────────────────────────────────────
(
  kube_deployment_status_replicas_available
  /
  kube_deployment_spec_replicas
) < 0.5

# ── 디스크 예측 (선형 회귀로 언제 가득 찰지 예측) ──────────────
predict_linear(
  node_filesystem_free_bytes{mountpoint="/"}[6h],
  24 * 3600   # 24시간 후 예측
) < 0

# ── Saturation: 큐 길이가 늘어나는지 ──────────────────────────
# worker 수 대비 처리 대기 중인 작업 수
sum(queue_jobs_waiting) / sum(kube_deployment_spec_replicas{deployment="worker"}) > 10
```

---

## 8. 자주 쓰는 레이블 조작

```promql
# 레이블 값 바꾸기 (label_replace)
label_replace(
  kube_pod_info{namespace="$namespace"},
  "short_name",      # 새 레이블 이름
  "$1",              # 캡처 그룹
  "pod",             # 원본 레이블
  "(.+)-[a-z0-9]+-[a-z0-9]+"  # 정규식 (deployment name 추출)
)

# 여러 메트릭 join (on 키로 결합)
kube_pod_info * on(pod, namespace) group_right()
kube_pod_labels{label_app="my-app"}

# 특정 레이블 제외 후 집계
sum without (pod, container, uid) (container_memory_working_set_bytes)
```

---

## 9. Recording Rules로 비싼 쿼리 최적화

대시보드 패널이 느릴 때 Recording Rules로 미리 계산합니다.

```yaml
# prometheus/recording-rules.yaml
groups:
  - name: recording.rules
    interval: 1m
    rules:
      # 네임스페이스별 CPU 사용률 (자주 쓰는 집계)
      - record: namespace:container_cpu_usage_seconds_total:sum_rate
        expr: |
          sum by (namespace) (
            rate(container_cpu_usage_seconds_total{container!=""}[5m])
          )

      # 서비스별 에러율
      - record: job:http_requests_error_rate:5m
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum by (job) (rate(http_requests_total[5m]))

      # P99 레이턴시 (histogram_quantile은 쿼리 비용이 높음)
      - record: job:http_request_duration_seconds:p99
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )
```

대시보드에서 Recording Rule 사용:
```promql
# 느린 쿼리 대신
namespace:container_cpu_usage_seconds_total:sum_rate{namespace="$namespace"}

# Recording Rule 네이밍 컨벤션: level:metric:operation
```

---

## 참고

- [PromQL 공식 문서](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Awesome Prometheus Alerts](https://samber.github.io/awesome-prometheus-alerts/)
- [Prometheus Histogram 심화](https://prometheus.io/docs/practices/histograms/)
