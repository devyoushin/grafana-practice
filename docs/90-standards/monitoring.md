# 모니터링 규칙 — grafana-practice

## Grafana 자체 모니터링

### 핵심 메트릭

```promql
# Grafana 프로세스 업타임
process_start_time_seconds{job="grafana"}

# 활성 알림 수
grafana_alerting_active_alerts

# 알림 평가 오류
rate(grafana_alerting_rule_evaluation_failures_total[5m])

# 데이터소스 쿼리 오류율
rate(grafana_datasource_request_failed_total[5m])

# HTTP 요청 지연 (p99)
histogram_quantile(0.99, rate(grafana_http_request_duration_seconds_bucket[5m]))
```

### 알림 규칙 (권장)

| 알림 | 조건 | 심각도 |
|------|------|--------|
| GrafanaDown | `up{job="grafana"} == 0` for 1m | critical |
| GrafanaAlertingFailure | `rate(grafana_alerting_rule_evaluation_failures_total[5m]) > 0` | warning |
| GrafanaDatasourceError | `rate(grafana_datasource_request_failed_total[5m]) > 0.1` | warning |

## SLO 정의 (Grafana 서비스)

| SLI | 목표 | 측정 방법 |
|-----|------|---------|
| 가용성 | 99.9% | `up{job="grafana"}` |
| 대시보드 로딩 p99 | < 5초 | `grafana_http_request_duration_seconds` |
| 알림 평가 성공률 | > 99% | `grafana_alerting_rule_evaluation_failures_total` |

## 대시보드 구성 (Grafana 자체 모니터링)
- **Overview**: 업타임, 활성 알림 수, 사용자 수
- **Alerting**: 알림 평가 현황, 발송 성공/실패
- **Datasources**: 데이터소스별 쿼리 성능
- **HTTP**: 요청 처리 현황 및 오류율
