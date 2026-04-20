# Grafana Alerting 가이드

Grafana Unified Alerting은 Prometheus, Loki, 기타 데이터소스를 아울러 하나의 알림 시스템으로 통합 관리합니다.

---

## 전체 흐름

```
Alert Rule (조건 평가, 주기적 실행)
    │ 조건 만족 + for 시간 경과
    ▼
Alert Manager (Grafana 내장)
    │ 그루핑 → 라우팅 → Silence 필터링
    ▼
Contact Point
    │
    ▼
Slack / Email / PagerDuty / Webhook
```

### Alert 상태 주기

```
Normal ──(조건 만족)──▶ Pending ──(for 시간 경과)──▶ Firing
   ▲                                                     │
   └─────────────────(조건 해소)────────────────────────┘
                                                 ▼
                                              Resolved (발송)
```

- **Normal**: 조건 불만족
- **Pending**: 조건 만족, 하지만 `for` 시간 미충족 (일시적 스파이크 무시)
- **Firing**: `for` 시간 초과 → Contact Point로 전송

---

## 1. Alert Rule 생성

```
Alerting → Alert rules → New alert rule
```

### 단계 1: 규칙 이름 및 폴더

```
Rule name: HighErrorRate
Folder: Production Alerts
Group: api-service
```

### 단계 2: 쿼리 및 조건

**Prometheus 기반 알림** (5xx 에러율 5% 초과):

```promql
# Query A
sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
/
sum by (job) (rate(http_requests_total[5m]))
```

```
Expressions:
  Threshold: A IS ABOVE 0.05
```

**Loki 기반 알림** (에러 로그 10건 초과):

```logql
# Query A (Instant query)
sum(count_over_time({namespace="default"} |= "ERROR" [5m]))
```

```
Expressions:
  Threshold: A IS ABOVE 10
```

### 단계 3: Alert 평가 설정

```
Evaluate every: 1m
For: 5m           # Pending → Firing 전환 시간
```

### 단계 4: 레이블 및 어노테이션

```yaml
Labels:
  severity: critical
  team: backend

Annotations:
  summary: "{{ $labels.job }} 에러율 {{ $values.A.Value | humanizePercentage }} 초과"
  description: |
    {{ $labels.job }}의 5분 에러율이 5%를 초과했습니다.
    현재 에러율: {{ $values.A.Value | humanizePercentage }}
  runbook: "https://wiki.company.com/runbooks/high-error-rate"
```

---

## 2. 자주 사용하는 Alert Rule 예시

### 서비스 다운

```promql
up{job="my-app"} == 0
```
```
For: 2m, Severity: critical
```

### 높은 레이턴시

```promql
histogram_quantile(0.99,
  sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))
) > 1.0
```
```
For: 5m, Severity: warning
```

### 파드 재시작 반복

```promql
increase(kube_pod_container_status_restarts_total{namespace="default"}[15m]) > 3
```
```
For: 0m, Severity: critical
```

### 메모리 사용률 90% 초과

```promql
(
  container_memory_working_set_bytes{container!=""}
  /
  container_spec_memory_limit_bytes{container!="", container_spec_memory_limit_bytes != 0}
) > 0.9
```
```
For: 10m, Severity: warning
```

### Loki: 에러 로그 급증

```logql
sum(rate({namespace="default"} |= "ERROR" [5m])) > 1
```
```
For: 3m, Severity: warning
```

---

## 3. Contact Point 설정

```
Alerting → Contact points → Add contact point
```

### Slack

```yaml
Name: slack-critical
Type: Slack

Integration settings:
  Webhook URL: https://hooks.slack.com/services/XXXXX/YYYYY/ZZZZZ
  Channel: #alerts-critical
  Title: |
    [{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}
  Text: |
    {{ range .Alerts }}
    *요약:* {{ .Annotations.summary }}
    *상세:* {{ .Annotations.description }}
    {{ if .Annotations.runbook }}*런북:* {{ .Annotations.runbook }}{{ end }}
    {{ end }}
  Send resolved: true
```

### Email

```yaml
Name: email-oncall
Type: Email

Addresses: oncall@company.com
Subject: [{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}
Single email: false
```

### PagerDuty

```yaml
Name: pagerduty-critical
Type: PagerDuty

Integration key: <PD_INTEGRATION_KEY>
Severity: critical
```

---

## 4. Notification Policy (라우팅)

```
Alerting → Notification policies
```

라우팅 트리를 정의합니다. 루트 정책에서 시작하여 레이블 조건으로 분기합니다.

```
Root policy
  receiver: slack-default
  group_by: [alertname, namespace]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h

  └── severity=critical
        receiver: pagerduty-critical
        repeat_interval: 1h
        continue: true      ← 다음 매처도 계속 평가

  └── severity=critical
        receiver: slack-critical

  └── severity=warning
        receiver: slack-warning

  └── team=backend
        receiver: slack-backend
```

---

## 5. Silence (음소거)

유지보수 시 특정 알림을 일시적으로 억제합니다.

```
Alerting → Silences → Add silence
```

```yaml
Start: 2026-04-14 02:00
End: 2026-04-14 04:00

Matchers:
  - namespace = staging

Comment: 스테이징 환경 유지보수 (DB 마이그레이션)
```

```bash
# Grafana API로 Silence 생성
curl -s -X POST http://admin:admin@localhost:3000/api/alertmanager/grafana/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [{"name": "namespace", "value": "staging", "isRegex": false}],
    "startsAt": "2026-04-14T02:00:00Z",
    "endsAt": "2026-04-14T04:00:00Z",
    "comment": "스테이징 유지보수",
    "createdBy": "operator"
  }' | jq '.id'
```

---

## 6. 알림 상태 확인

```bash
kubectl port-forward svc/grafana -n monitoring 3000:80 &

# 현재 Firing 알림 목록
curl -s http://admin:admin@localhost:3000/api/alertmanager/grafana/api/v2/alerts?active=true | \
  jq '.[] | {name: .labels.alertname, status: .status.state, summary: .annotations.summary}'

# Alert Rule 목록
curl -s http://admin:admin@localhost:3000/api/v1/provisioning/alert-rules | \
  jq '.[] | {uid: .uid, title: .title, state: .isPaused}'
```

---

## 7. Alert Rule을 코드로 관리

```bash
# 기존 Alert Rule export
curl -s http://admin:admin@localhost:3000/api/v1/provisioning/alert-rules/export?download=true \
  > alert-rules-export.yaml

# Alert Rule import
curl -s -X PUT http://admin:admin@localhost:3000/api/v1/provisioning/alert-rules/import \
  -H "Content-Type: application/yaml" \
  --data-binary @alert-rules-export.yaml
```

> Provisioning 방식으로 Alert Rule을 코드로 관리하는 방법은 [provisioning-guide.md](./provisioning-guide.md)를 참고하세요.

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|----------|
| Pending에서 Firing으로 안됨 | `for` 시간 미충족 | 조건이 지속적으로 만족하는지 Explore에서 확인 |
| Slack 알림 안옴 | Webhook URL 만료 | Slack App에서 새 Webhook 발급 |
| 중복 알림 과다 | `repeat_interval` 짧음 | Notification Policy에서 `repeat_interval: 12h` 이상으로 설정 |
| 쿼리 오류로 알림 생성 안됨 | PromQL/LogQL 문법 오류 | Alert Rule 편집 → Preview 탭에서 쿼리 결과 확인 |
| No data 상태 | 메트릭 수집 안됨 | 데이터소스 Health Check 및 Explore에서 쿼리 직접 실행 |

---

## 참고

- [공식문서 - Grafana Alerting](https://grafana.com/docs/grafana/latest/alerting/)
- [공식문서 - Notification policies](https://grafana.com/docs/grafana/latest/alerting/fundamentals/notification-policies/)
- [공식문서 - Contact points](https://grafana.com/docs/grafana/latest/alerting/fundamentals/contact-points/)
