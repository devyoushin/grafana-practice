# Provisioning as Code 가이드

Grafana Provisioning을 사용하면 데이터소스·대시보드·알림 설정을 YAML/JSON 파일로 선언적으로 관리할 수 있습니다. Helm 배포 시 자동 적용되므로 GitOps 환경에 적합합니다.

---

## Provisioning 개요

```
grafana/
└── provisioning/
    ├── datasources/
    │   └── datasources.yaml     ← 데이터소스 자동 등록
    ├── dashboards/
    │   └── dashboards.yaml      ← 대시보드 파일 위치 지정
    └── alerting/
        ├── alert-rules.yaml     ← Alert Rule
        └── contact-points.yaml  ← Contact Point
```

Grafana 파드 시작 시 이 파일들을 읽어 자동으로 적용합니다.

---

## 1. 데이터소스 Provisioning

```yaml
# ../ops/config/grafana/provisioning/datasources.yaml
apiVersion: 1

datasources:
  # Prometheus
  - name: Prometheus
    type: prometheus
    uid: prometheus
    url: http://prometheus.monitoring.svc.cluster.local:9090
    access: proxy
    httpMethod: POST
    isDefault: true
    jsonData:
      timeInterval: "15s"
      queryTimeout: "60s"
    editable: false

  # Mimir (멀티 테넌트)
  - name: Mimir
    type: prometheus
    uid: mimir
    url: http://mimir-nginx.monitoring.svc.cluster.local/prometheus
    access: proxy
    httpMethod: POST
    jsonData:
      prometheusType: Mimir
      httpHeaderName1: X-Scope-OrgID
      timeInterval: "15s"
    secureJsonData:
      httpHeaderValue1: default
    editable: false

  # Loki
  - name: Loki
    type: loki
    uid: loki
    url: http://loki.monitoring.svc.cluster.local:3100
    access: proxy
    jsonData:
      maxLines: 1000
      derivedFields:
        - name: TraceID
          matcherRegex: "traceID=(\\w+)"
          url: "$${__value.raw}"
          datasourceUid: tempo
    editable: false

  # Tempo
  - name: Tempo
    type: tempo
    uid: tempo
    url: http://tempo.monitoring.svc.cluster.local:3100
    access: proxy
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki
        filterByTraceID: true
        customQuery: false
      tracesToMetrics:
        datasourceUid: prometheus
      lokiSearch:
        datasourceUid: loki
    editable: false
```

---

## 2. 대시보드 Provisioning

### 단계 1: 대시보드 프로바이더 설정

```yaml
# ../ops/config/grafana/provisioning/dashboards.yaml
apiVersion: 1

providers:
  - name: default
    type: file
    disableDeletion: false       # false면 파일 삭제 시 Grafana에서도 삭제
    updateIntervalSeconds: 30    # 30초마다 변경 감지
    allowUiUpdates: false        # UI에서 수정 불가 (코드 단일 소스 원칙)
    options:
      path: /var/lib/grafana/dashboards/default
      foldersFromFilesStructure: true  # 폴더 구조 → Grafana 폴더로 자동 매핑
```

### 단계 2: 대시보드 JSON 파일 준비

```bash
# 기존 대시보드 JSON 내보내기
curl -s http://admin:admin@localhost:3000/api/dashboards/uid/<UID> | \
  jq '.dashboard | del(.id, .version)' \
  > ../ops/config/grafana/provisioning/dashboards/my-dashboard.json
```

> `id`와 `version` 필드를 제거해야 Provisioning 시 충돌이 없습니다.

---

## 3. Helm values로 Provisioning 설정

Grafana Helm Chart는 `datasources`와 `dashboardProviders` 값을 ConfigMap으로 자동 변환합니다.

```yaml
# ../ops/config/grafana/values.yaml (발췌)
grafana.ini:
  server:
    root_url: http://grafana.monitoring.svc.cluster.local

# 데이터소스 프로비저닝
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        uid: prometheus
        url: http://prometheus.monitoring.svc.cluster.local:9090
        isDefault: true
        jsonData:
          timeInterval: "15s"

      - name: Loki
        type: loki
        uid: loki
        url: http://loki.monitoring.svc.cluster.local:3100
        jsonData:
          maxLines: 1000

# 대시보드 프로바이더
dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
      - name: default
        orgId: 1
        folder: ''
        type: file
        disableDeletion: false
        updateIntervalSeconds: 30
        options:
          path: /var/lib/grafana/dashboards/default

# 대시보드 ConfigMap 또는 URL에서 자동 로드
dashboards:
  default:
    # Grafana.com 대시보드 자동 import
    node-exporter:
      gnetId: 1860
      revision: 37
      datasource: Prometheus

    kubernetes-cluster:
      gnetId: 15757
      revision: 37
      datasource: Prometheus

    # 로컬 파일 (ConfigMap)
    my-app:
      json: |
        { ... 대시보드 JSON 내용 ... }
```

---

## 4. Alert Rule Provisioning

```yaml
# ../ops/config/grafana/provisioning/alerting/alert-rules.yaml
apiVersion: 1

groups:
  - orgId: 1
    name: production-alerts
    folder: Alerts
    interval: 1m
    rules:
      - uid: high-error-rate
        title: HighErrorRate
        condition: C
        data:
          - refId: A
            datasourceUid: prometheus
            model:
              expr: |
                sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
                / sum by (job) (rate(http_requests_total[5m]))
              intervalMs: 1000
              maxDataPoints: 43200
          - refId: C
            datasourceUid: __expr__
            model:
              conditions:
                - evaluator:
                    params: [0.05]
                    type: gt
                  operator:
                    type: and
                  query:
                    params: [A]
                  reducer:
                    type: last
              type: threshold
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "에러율 5% 초과"
          description: "{{ $labels.job }} 에러율: {{ $values.A.Value | humanizePercentage }}"
```

---

## 5. Contact Point Provisioning

```yaml
# ../ops/config/grafana/provisioning/alerting/contact-points.yaml
apiVersion: 1

contactPoints:
  - orgId: 1
    name: slack-critical
    receivers:
      - uid: slack-critical-uid
        type: slack
        settings:
          url: ${SLACK_WEBHOOK_URL}    # 환경변수 참조
          channel: "#alerts-critical"
          title: "[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}"
          text: |
            {{ range .Alerts }}
            *요약:* {{ .Annotations.summary }}
            *상세:* {{ .Annotations.description }}
            {{ end }}
          sendResolved: true

  - orgId: 1
    name: email-oncall
    receivers:
      - uid: email-oncall-uid
        type: email
        settings:
          addresses: oncall@company.com

notificationPolicies:
  - orgId: 1
    receiver: slack-critical
    group_by: [alertname, namespace]
    group_wait: 30s
    group_interval: 5m
    repeat_interval: 12h
    routes:
      - receiver: slack-critical
        matchers:
          - severity=critical
```

---

## 6. Kubernetes ConfigMap으로 관리

```yaml
# k8s/grafana-dashboards-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
  labels:
    grafana_dashboard: "1"    # sidecar가 이 레이블을 감시
data:
  my-app-dashboard.json: |
    {
      "title": "My App Dashboard",
      "uid": "my-app",
      ...
    }
```

Helm values에서 sidecar 활성화:

```yaml
sidecar:
  dashboards:
    enabled: true
    label: grafana_dashboard
    labelValue: "1"
    folder: /tmp/dashboards
    searchNamespace: monitoring  # 이 네임스페이스의 ConfigMap만 감시
```

```bash
kubectl apply -f k8s/grafana-dashboards-configmap.yaml
# 30초 이내에 Grafana에 자동 반영됩니다
```

---

## 7. GitOps 워크플로우

```
Git Repository
    │
    ├── ../ops/config/grafana/provisioning/datasources.yaml
    ├── ../ops/config/grafana/provisioning/dashboards.yaml
    ├── ../ops/config/grafana/provisioning/dashboards/*.json
    └── ../ops/config/grafana/values.yaml
          │
          ▼ ArgoCD / Flux
    Helm release 자동 업데이트
          │
          ▼
    Grafana ConfigMap 갱신
          │
          ▼
    sidecar가 변경 감지 → 대시보드 자동 반영
```

대시보드 수정 워크플로우:
1. UI에서 대시보드 수정
2. `Share → Export → Save to file`로 JSON 내보내기
3. Git에 커밋
4. CI/CD로 ConfigMap 업데이트
5. Grafana에 자동 반영

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|----------|
| 데이터소스 프로비저닝 안됨 | YAML 문법 오류 | `kubectl logs grafana-xxx -n monitoring` 확인 |
| 대시보드 UI 수정 안됨 | `allowUiUpdates: false` | 파일을 수정해서 반영 (의도된 동작) |
| 대시보드 중복 생성 | UID 충돌 | `uid` 필드가 고유한지 확인 |
| sidecar 대시보드 반영 안됨 | ConfigMap 레이블 누락 | `grafana_dashboard: "1"` 레이블 확인 |
| Secret 참조 실패 | 환경변수 미설정 | `envFrom.secretRef` 설정 확인 |

---

## 참고

- [공식문서 - Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- [공식문서 - Dashboard provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards)
- [grafana/grafana Helm Chart](https://github.com/grafana/helm-charts/tree/main/charts/grafana)
