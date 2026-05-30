# Explore 가이드

Explore는 대시보드 없이 데이터소스에 직접 쿼리를 날려 데이터를 탐색하는 임시 작업 공간입니다. 장애 분석, 로그 디버깅, 쿼리 개발에 사용합니다.

---

## Explore 접근

```
메뉴 → Explore (나침반 아이콘)
또는 단축키: G → E
```

---

## 1. PromQL (Prometheus / Mimir)

### 기본 사용

```
Data source: Prometheus
Query mode: Code (PromQL 직접 입력) 또는 Builder (GUI)
```

```promql
# 네임스페이스별 CPU 사용률 상위 10개
topk(10,
  sum by (namespace) (
    rate(container_cpu_usage_seconds_total{container!=""}[5m])
  )
)
```

### Metrics Browser

Explore 상단 **Metrics browser** 클릭 → 메트릭명 선택 → 레이블 필터 선택 → **Use query**

---

## 2. LogQL (Loki)

### 기본 사용

```
Data source: Loki
```

**Log Browser** 사용:
```
1. Label browser 클릭
2. namespace → default 선택
3. app → my-app 선택
4. Show logs 클릭
```

**LogQL 직접 입력**:
```logql
# 에러 로그만 필터
{namespace="default", app="my-app"} |= "ERROR"

# JSON 파싱 후 status_code 필터
{namespace="default"} | json | status_code >= 500

# 에러 로그 발생 추이 (Metric 쿼리)
sum(rate({namespace="default"} |= "ERROR" [5m])) by (app)
```

### Live tail

실시간으로 로그를 스트리밍합니다.

```
Explore → 쿼리 입력 → Live (▶) 버튼 클릭
```

### Log context

특정 로그 라인의 전후 맥락을 확인합니다.

```
로그 라인 우측 → Context 클릭 → 전후 10줄 표시
```

---

## 3. TraceQL (Tempo)

```
Data source: Tempo
Query type: Search 또는 TraceQL
```

**Search 모드** (GUI):
```
Service name: my-app
Span name: /api/users
Duration: > 500ms
```

**TraceQL 직접 입력**:
```
# 500ms 이상 걸린 my-app 스팬
{.service.name = "my-app" && duration > 500ms}

# 에러가 발생한 트레이스
{status = error}

# 특정 경로의 P95 레이턴시
quantile_over_time(0.95, {.http.route = "/api/users"}[1h])
```

---

## 4. Split View (분할 화면)

두 데이터소스를 나란히 놓고 상관 분석합니다.

```
Explore 우측 상단 → Split 버튼 클릭
```

**활용 예시**:
- 왼쪽: Prometheus (CPU 급등 시점 확인)
- 오른쪽: Loki (같은 시간대 에러 로그 확인)

```
왼쪽 쿼리:
rate(container_cpu_usage_seconds_total{pod=~"my-app.*"}[5m])

오른쪽 쿼리:
{namespace="default", app="my-app"} |= "ERROR"
```

→ 두 그래프의 **시간 범위가 연동**되어 함께 스크롤됩니다.

---

## 5. 데이터소스 간 이동 (Correlations)

### 메트릭 → 트레이스 (Exemplars)

Prometheus에 Exemplar가 활성화된 경우, 그래프의 다이아몬드(◆) 점을 클릭하면 해당 Trace ID로 Tempo Explore가 열립니다.

```
Prometheus 설정에서 Exemplars 활성화:
  Prometheus → Configuration → Exemplars → Enable
  Link datasource: Tempo
```

### 로그 → 트레이스 (Derived fields)

로그에 포함된 Trace ID를 클릭하면 Tempo로 이동합니다.

```
Configuration → Data Sources → Loki → Derived fields
  Name: TraceID
  Regex: traceID=(\w+)
  Internal link: Tempo
```

```logql
# trace_id가 포함된 로그
{namespace="default"} | json | trace_id != ""
```

→ 로그 라인에서 TraceID 링크 클릭 → Tempo에서 해당 트레이스 자동 표시

### 트레이스 → 로그

```
Configuration → Data Sources → Tempo → Trace to logs
  Data source: Loki
  Tags: service.name → app
  Map tag names: service.name → app
```

---

## 6. 유용한 Explore 단축키

| 단축키 | 동작 |
|--------|------|
| `Ctrl/Cmd + Enter` | 쿼리 실행 |
| `Shift + Enter` | 새 쿼리 라인 |
| `Ctrl/Cmd + H` | 쿼리 히스토리 |
| `Ctrl/Cmd + L` | 쿼리 지우기 |
| `G → E` | Explore로 이동 |

---

## 7. 쿼리 히스토리

이전에 실행했던 쿼리를 재사용합니다.

```
Explore → Query history (시계 아이콘) → 검색 또는 즐겨찾기
```

---

## 8. Explore에서 대시보드 패널로 추가

쿼리 개발 후 바로 대시보드에 추가할 수 있습니다.

```
Explore → 쿼리 실행 → Add to dashboard 버튼 (우측 상단)
→ 기존 대시보드 선택 또는 새 대시보드 생성
→ Open dashboard
```

---

## 참고

- [공식문서 - Explore](https://grafana.com/docs/grafana/latest/explore/)
- [공식문서 - Correlations](https://grafana.com/docs/grafana/latest/explore/correlations-editor/)
- [공식문서 - TraceQL](https://grafana.com/docs/tempo/latest/traceql/)
