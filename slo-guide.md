# SLO / Error Budget 대시보드 가이드

SLO(Service Level Objective)를 Grafana에서 정의하고, Error Budget 소진율을 모니터링하는 방법을 다룹니다.

---

## 1. SLO 개념 정리

```
SLI (Service Level Indicator)
  → 측정 지표: 가용성 99.9%, P99 레이턴시 < 500ms

SLO (Service Level Objective)
  → 목표: SLI가 30일 기준 99.9% 이상 유지

Error Budget
  → 허용 가능한 오류 여유분
  → 99.9% SLO = 0.1% 오류 허용 = 30일 × 43.2분 = 43.2분 다운타임 허용

Burn Rate
  → Error Budget이 소진되는 속도
  → Burn Rate 1 = 30일 SLO 기간 동안 정확히 Budget을 모두 소진
  → Burn Rate 2 = 15일 만에 Budget 소진 (2배 빠름)
```

---

## 2. SLI 정의 및 PromQL

### 가용성 SLI (성공 요청 비율)

```promql
# 최근 1시간 가용성
sum(rate(http_requests_total{status!~"5.."}[1h]))
/
sum(rate(http_requests_total[1h]))

# 최근 30일 가용성 (Recording Rule로 미리 계산 권장)
sum(increase(http_requests_total{status!~"5.."}[30d]))
/
sum(increase(http_requests_total[30d]))
```

### 레이턴시 SLI (P99 < 500ms 비율)

```promql
# 500ms 이하 요청 비율
sum(rate(http_request_duration_seconds_bucket{le="0.5"}[1h]))
/
sum(rate(http_request_duration_seconds_count[1h]))
```

### Error Budget 계산

```promql
# 남은 Error Budget (%)
# SLO: 99.9%

(
  sum(rate(http_requests_total{status!~"5.."}[30d]))
  /
  sum(rate(http_requests_total[30d]))
  - 0.999
) / (1 - 0.999)

# 해석:
#  1.0 = Budget 100% 남음 (에러 없음)
#  0.5 = Budget 50% 남음
#  0.0 = Budget 모두 소진
# -1.0 = Budget 초과 (SLO 위반)
```

---

## 3. Burn Rate 알림 (Google SRE 권장)

Burn Rate 알림은 **빠른 소진(Fast Burn)**과 **느린 소진(Slow Burn)** 두 가지를 조합합니다.

```
Fast Burn  (1시간 창): 즉각적인 심각한 장애 감지
Slow Burn  (6시간 창): 지속적인 낮은 수준 문제 감지
```

### Burn Rate 계산 공식

```promql
# Burn Rate = 에러율 / (1 - SLO)
# SLO가 99.9%이면 (1 - SLO) = 0.001

# 1시간 Burn Rate
(
  sum(rate(http_requests_total{status=~"5.."}[1h]))
  /
  sum(rate(http_requests_total[1h]))
) / (1 - 0.999)

# 6시간 Burn Rate
(
  sum(rate(http_requests_total{status=~"5.."}[6h]))
  /
  sum(rate(http_requests_total[6h]))
) / (1 - 0.999)
```

### Multi-Window Burn Rate Alert

```yaml
# Prometheus Alert Rule
groups:
  - name: slo.alerts
    rules:
      # Page: 1시간에 2% Error Budget 소진 (= 50배 빠름)
      - alert: ErrorBudgetBurnFast
        expr: |
          (
            (
              sum(rate(http_requests_total{status=~"5..", job="my-app"}[1h]))
              / sum(rate(http_requests_total{job="my-app"}[1h]))
            ) / (1 - 0.999) > 14.4    # 14.4 = 2% / (1h/30d)
          )
          and
          (
            (
              sum(rate(http_requests_total{status=~"5..", job="my-app"}[5m]))
              / sum(rate(http_requests_total{job="my-app"}[5m]))
            ) / (1 - 0.999) > 14.4
          )
        for: 0m
        labels:
          severity: critical
          slo: availability
        annotations:
          summary: "Error Budget 빠른 소진 — {{ $value | printf \"%.1f\" }}배 속도"
          description: |
            현재 Burn Rate가 {{ $value | printf \"%.1f\" }}x 입니다.
            이 속도가 유지되면 {{ (1 / $value * 30 * 24) | printf \"%.1f\" }}시간 내에 Error Budget이 모두 소진됩니다.

      # Ticket: 6시간에 5% Error Budget 소진 (= 6배 빠름)
      - alert: ErrorBudgetBurnSlow
        expr: |
          (
            (
              sum(rate(http_requests_total{status=~"5..", job="my-app"}[6h]))
              / sum(rate(http_requests_total{job="my-app"}[6h]))
            ) / (1 - 0.999) > 6
          )
          and
          (
            (
              sum(rate(http_requests_total{status=~"5..", job="my-app"}[30m]))
              / sum(rate(http_requests_total{job="my-app"}[30m]))
            ) / (1 - 0.999) > 6
          )
        for: 0m
        labels:
          severity: warning
          slo: availability
        annotations:
          summary: "Error Budget 느린 소진 — {{ $value | printf \"%.1f\" }}배 속도"
```

---

## 4. SLO 대시보드 구성

### 레이아웃

```
┌────────────────────────────────────────────────────────────────┐
│ Variables: [service ▼] [slo_target ▼]                        │
├───────────┬───────────┬───────────┬───────────────────────────┤
│ 현재 가용성│ Error     │ Burn Rate │ SLO Target                │
│ 99.97%    │ Budget    │ 0.3x      │ 99.9%                      │
│           │ 남음 73%  │           │                            │
├───────────┴───────────┴───────────┴───────────────────────────┤
│ Error Budget 소진 추이 (30일, 시계열)                          │
│ 100%┤▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░                │
│  0%┤                                                           │
├────────────────────────────────────────────────────────────────┤
│ 시간대별 에러율 (24h, 시계열)    │ Burn Rate 추이 (24h)        │
├──────────────────────────────────┼─────────────────────────────┤
│ 엔드포인트별 에러 건수 (테이블) │                              │
└────────────────────────────────────────────────────────────────┘
```

### 패널별 PromQL

```promql
# ─── Stat: 현재 가용성 (지난 30일) ────────────────────────────
sum(increase(http_requests_total{job="$service", status!~"5.."}[30d]))
/
sum(increase(http_requests_total{job="$service"}[30d]))

# 임계값 설정:
#   99.9% 이상 → green
#   99.0% ~ 99.9% → yellow
#   99.0% 미만 → red


# ─── Gauge: 남은 Error Budget (%) ─────────────────────────────
(
  (
    sum(increase(http_requests_total{job="$service", status!~"5.."}[30d]))
    / sum(increase(http_requests_total{job="$service"}[30d]))
    - 0.999
  ) / (1 - 0.999)
) * 100

# min: 0, max: 100
# 임계값: 50% → yellow, 20% → red


# ─── Time series: Error Budget 소진 추이 ──────────────────────
# 현재까지 누적 소진량 (%)
(1 -
  (
    sum(increase(http_requests_total{job="$service", status!~"5.."}[$__range]))
    / sum(increase(http_requests_total{job="$service"}[$__range]))
    - 0.999
  ) / (1 - 0.999)
) * 100


# ─── Time series: Burn Rate ────────────────────────────────────
# 1시간 Burn Rate
(
  sum(rate(http_requests_total{job="$service", status=~"5.."}[1h]))
  / sum(rate(http_requests_total{job="$service"}[1h]))
) / (1 - 0.999)

# 6시간 Burn Rate (같은 패널에 2번째 쿼리)
(
  sum(rate(http_requests_total{job="$service", status=~"5.."}[6h]))
  / sum(rate(http_requests_total{job="$service"}[6h]))
) / (1 - 0.999)

# 참조선 추가: Burn Rate = 1 (정상 소진 속도)


# ─── Table: 엔드포인트별 에러 건수 ────────────────────────────
topk(10,
  sum by (handler, status) (
    increase(http_requests_total{job="$service", status=~"5.."}[24h])
  )
)
```

---

## 5. 레이턴시 SLO 대시보드

```promql
# 현재 레이턴시 SLI (P99 < 500ms 요청 비율)
sum(rate(http_request_duration_seconds_bucket{job="$service", le="0.5"}[1h]))
/
sum(rate(http_request_duration_seconds_count{job="$service"}[1h]))

# 레이턴시 SLO Error Budget
(
  sum(rate(http_request_duration_seconds_bucket{job="$service", le="0.5"}[30d]))
  / sum(rate(http_request_duration_seconds_count{job="$service"}[30d]))
  - 0.999
) / (1 - 0.999)

# P50 / P95 / P99 추이
histogram_quantile(0.50, sum by (le) (rate(http_request_duration_seconds_bucket{job="$service"}[5m])))
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{job="$service"}[5m])))
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{job="$service"}[5m])))
```

---

## 6. SLO 목표 기준 참고

| SLO | 월간 허용 다운타임 | 연간 허용 다운타임 | 적합한 서비스 |
|-----|----------------|----------------|------------|
| 99.0% ("two nines") | 7.3시간 | 3.65일 | 내부 도구 |
| 99.5% | 3.7시간 | 1.83일 | 내부 서비스 |
| 99.9% ("three nines") | 43.8분 | 8.77시간 | 일반 프로덕션 |
| 99.95% | 21.9분 | 4.38시간 | 핵심 서비스 |
| 99.99% ("four nines") | 4.4분 | 52.6분 | 결제, 인증 |

---

## 7. Recording Rules로 SLO 계산 최적화

30일 창의 쿼리는 비용이 매우 큽니다. Recording Rule로 미리 계산합니다.

```yaml
groups:
  - name: slo.recording
    interval: 5m
    rules:
      # 5분 윈도우 에러율
      - record: job:http_requests_error_rate:5m
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          / sum by (job) (rate(http_requests_total[5m]))

      # 1시간 Burn Rate
      - record: job:error_budget_burn_rate:1h
        expr: |
          (
            sum by (job) (rate(http_requests_total{status=~"5.."}[1h]))
            / sum by (job) (rate(http_requests_total[1h]))
          ) / (1 - 0.999)

      # 6시간 Burn Rate
      - record: job:error_budget_burn_rate:6h
        expr: |
          (
            sum by (job) (rate(http_requests_total{status=~"5.."}[6h]))
            / sum by (job) (rate(http_requests_total[6h]))
          ) / (1 - 0.999)
```

---

## 참고

- [Google SRE Workbook - SLO Implementation](https://sre.google/workbook/implementing-slos/)
- [Google SRE Workbook - Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Prometheus SLO 계산](https://prometheus.io/docs/practices/instrumentation/)
