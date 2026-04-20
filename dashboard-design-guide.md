# 대시보드 설계 가이드

좋은 대시보드는 "지금 시스템이 정상인가?"를 **5초 안에** 판단할 수 있어야 합니다. 이 가이드는 실무에서 유용한 대시보드를 만드는 원칙과 방법을 다룹니다.

---

## 1. 관찰 가능성 방법론

### Golden Signals (구글 SRE 방법론)

서비스 건강 상태를 나타내는 가장 중요한 4가지 지표입니다.

| 지표 | 설명 | PromQL 예시 |
|------|------|------------|
| **Latency** | 요청 처리 시간. 특히 에러 레이턴시를 정상 레이턴시와 분리할 것 | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` |
| **Traffic** | 서비스에 들어오는 수요(RPS, connections) | `rate(http_requests_total[5m])` |
| **Errors** | 실패한 요청 비율 | `rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])` |
| **Saturation** | 리소스가 얼마나 가득 찼는가 (CPU, 메모리, 큐 길이) | `container_memory_working_set_bytes / container_spec_memory_limit_bytes` |

> 서비스 대시보드는 반드시 이 4가지를 최상단에 배치하세요.

---

### RED Method (마이크로서비스 지향)

서비스 레이어에서 사용합니다.

| 지표 | 설명 |
|------|------|
| **R**ate | 초당 처리 요청 수 |
| **E**rrors | 실패 요청 수 또는 비율 |
| **D**uration | 요청 처리 시간 (P50, P95, P99) |

---

### USE Method (인프라 리소스 지향)

노드, 디스크, 네트워크 등 리소스 레이어에서 사용합니다.

| 지표 | 설명 |
|------|------|
| **U**tilization | 리소스 사용률 (0~100%) |
| **S**aturation | 대기열 길이, throttle 발생 여부 |
| **E**rrors | 에러 발생 수 |

---

## 2. 대시보드 계층 구조

대시보드는 **넓고 얕게 시작해서 좁고 깊게** 드릴다운하도록 설계합니다.

```
Level 1: Fleet Overview (클러스터 전체)
    → 이상 징후가 있는 namespace/node를 빠르게 식별
    └── Level 2: Namespace / Node (범위 좁히기)
            → 문제 namespace의 workload 확인
            └── Level 3: Pod / Service (개별 workload)
                    → 특정 pod의 에러/레이턴시 확인
                    └── Level 4: Trace / Log (원인 분석)
                            → Explore에서 근본 원인 파악
```

### 계층별 대시보드 예시

| 레벨 | 대시보드 이름 | 핵심 패널 |
|------|-------------|----------|
| Fleet | Kubernetes Cluster Overview | 노드 수, 파드 수, CPU/메모리 전체 |
| Namespace | Namespace Resources | namespace별 CPU/Memory Request/Limit 대비 사용량 |
| Workload | Deployment Overview | 레플리카 수, CPU, 메모리, 재시작 수 |
| Service | Service RED | RPS, 에러율, P50/P95/P99 레이턴시 |

---

## 3. 좋은 대시보드의 구조

### 최상단: 요약 행 (Summary Row)

한 눈에 상태를 파악할 수 있는 Stat 패널 4~6개를 배치합니다.

```
[ 현재 RPS ] [ 에러율 ] [ P99 레이턴시 ] [ 가용 파드 수 ] [ CPU 사용률 ] [ 메모리 사용률 ]
```

### 중간: 핵심 시계열 그래프

시간 흐름에 따른 추이를 보여주는 Time series 패널.

```
[ RPS 추이 (시계열) ] [ 에러율 추이 (시계열) ]
[ P99 레이턴시 추이 ] [ 메모리 사용량 추이  ]
```

### 하단: 상세 정보

Table 패널로 개별 pod/instance 상태 나열.

```
[ 파드별 CPU / 메모리 / 재시작 수 테이블 ]
[ 에러 로그 패널 (Loki) ]
```

---

## 4. 패널 설계 원칙

### 단위(Unit) 반드시 설정

숫자만 표시하면 의미를 알 수 없습니다.

```
나쁜 예: 1073741824
좋은 예: 1.0 GiB

Field config → Standard options → Unit
  메모리: bytes (IEC)  → "1.0 GiB"
  CPU:    short        → "0.5 Cores"
  비율:   percent (0-1) 또는 percent (0-100)
  시간:   milliseconds 또는 duration (hh:mm:ss)
  RPS:    reqps (req/s)
```

### 임계값(Threshold) 색상 설정

패널에 임계값을 설정하면 상태가 색상으로 즉시 전달됩니다.

```
CPU 사용률:       0% → green  /  70% → yellow  /  90% → red
메모리 사용률:    0% → green  /  80% → yellow  /  95% → red
에러율:           0% → green  / 0.1% → yellow  /   1% → red
P99 레이턴시:   0ms → green  / 500ms → yellow  / 1000ms → red
재시작 수:        0 → green  /    1 → yellow  /     5 → red
```

### Legend 간결하게

```
나쁜 예: container_cpu_usage_seconds_total{container="my-app", namespace="default", pod="my-app-abc123-xyz"}
좋은 예: my-app-abc123-xyz

→ Legend format: {{pod}} 또는 {{pod}} / {{container}}
```

### No data 상태 처리

데이터가 없을 때 어떻게 표시할지 설정합니다.

```
Panel → Field config → No value: N/A 또는 0
```

---

## 5. 레이아웃 원칙

### 그리드 크기 기준

Grafana는 24컬럼 그리드를 사용합니다.

| 패널 너비 | 용도 |
|---------|------|
| 24 (전체) | 로그 패널, 전체 너비 테이블 |
| 12 (절반) | 주요 시계열 그래프 2개 나란히 |
| 8 (1/3) | 3개 나란히 비교 |
| 6 (1/4) | Stat 패널 4개 요약 행 |
| 4 (1/6) | Stat 패널 6개 요약 행 |

### 시각적 계층

```
높이 4~6: Stat 패널 (요약 행)
높이 8~10: Time series 패널 (추이)
높이 6~8: Table 패널 (상세)
높이 8~12: Log 패널 (로그)
```

---

## 6. 대시보드 안티패턴

| 안티패턴 | 문제 | 해결 방법 |
|---------|------|----------|
| 패널이 너무 많음 | 정보 과부하, 어디를 봐야 할지 모름 | 핵심 지표 10~15개로 제한, 나머지는 드릴다운용 별도 대시보드 |
| 단위 없는 숫자 | 의미 파악 불가 | 모든 패널에 Unit 설정 |
| 고정된 namespace/pod | 재사용 불가 | 템플릿 변수 사용 |
| 시간 범위 고정 | 유연성 없음 | 시간 범위를 변수로 두고 `$__rate_interval` 활용 |
| 알 수 없는 이름 | 유지보수 어려움 | 패널 제목과 Description 작성 |
| 색상만으로 구분 | 색맹 접근성 문제 | 색상 + 레이블 조합 사용 |
| 너무 짧은 새로고침 | 데이터소스 과부하 | 30s 이상 권장, 실시간 필요 시 Loki Live tail 활용 |

---

## 7. 실무 대시보드 설계 예시

### 서비스 RED 대시보드 레이아웃

```
┌─────────────────────────────────────────────────────────────────┐
│ Variables: [namespace ▼] [service ▼] [interval ▼]              │
├──────────┬──────────┬──────────┬──────────┬──────────┬──────────┤
│  RPS     │ Error    │ P50      │ P99      │ Pods     │ Restarts │
│  42 r/s  │ 0.2%     │ 45ms     │ 280ms    │ 3/3      │ 0        │
│  (stat)  │ (stat)   │ (stat)   │ (stat)   │ (stat)   │ (stat)   │
├──────────┴──────────┴──────────┴──────────┴──────────┴──────────┤
│              RPS 추이 (timeseries, 12col)       │  에러율 (12col) │
├─────────────────────────────────────────────────┼────────────────┤
│              P99 / P95 / P50 레이턴시 (12col)   │  CPU (12col)   │
├─────────────────────────────────────────────────┴────────────────┤
│              파드별 상태 테이블 (24col)                           │
│              pod | cpu | memory | restarts | age                 │
├──────────────────────────────────────────────────────────────────┤
│              에러 로그 (Loki, 24col)                              │
└──────────────────────────────────────────────────────────────────┘
```

### 인프라 USE 대시보드 레이아웃

```
┌─────────────────────────────────────────────────────────────────┐
│ Variables: [node ▼] [interval ▼]                               │
├──────────┬──────────┬──────────┬──────────────────────────────┤
│ CPU Util │ Mem Util │ Disk Util│ Net Util                       │
│ 35%      │ 68%      │ 42%      │ 120 MB/s                       │
├──────────┴──────────┴──────────┴──────────────────────────────┤
│    CPU 사용률 추이 (12col)      │   메모리 사용량 추이 (12col)   │
├────────────────────────────────┼────────────────────────────────┤
│    디스크 I/O (12col)           │   네트워크 I/O (12col)         │
├──────────────────────────────────────────────────────────────────┤
│    파드별 CPU/메모리 테이블 (24col)                              │
└──────────────────────────────────────────────────────────────────┘
```

---

## 8. 대시보드 링크 설정

대시보드 간 드릴다운 이동을 설정합니다.

```
Dashboard Settings → Links → Add link

Type: Dashboard
Title: Pod Detail →
URL: /d/pod-detail/pod-detail?var-namespace=${namespace}&var-pod=${__series.labels.pod}
```

패널 수준 링크:
```
Panel → Edit → Field config → Data links
Title: Explore logs
URL: /explore?left={"datasource":"loki","queries":[{"expr":"{namespace=\"${__field.labels.namespace}\",pod=\"${__field.labels.pod}\"}"}]}
```

---

## 참고

- [Google SRE Book - Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Grafana 공식 - Best practices for dashboards](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/)
- [USE Method](https://www.brendangregg.com/usemethod.html)
