# 종합 실습

설치부터 대시보드 생성, 알림 설정까지 전체 흐름을 단계별로 실습합니다.

---

## 전제 조건

- EKS 클러스터 및 kubectl 설정 완료
- `monitoring` 네임스페이스에 Prometheus, Loki 설치되어 있음
- `default` 네임스페이스에 샘플 앱 배포되어 있음

---

## 1단계: Grafana 설치

```bash
# Helm 저장소 추가
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Admin 패스워드 Secret 생성
kubectl create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=grafana123 \
  --namespace monitoring

# Grafana 설치
helm install grafana grafana/grafana \
  --namespace monitoring \
  --values ../ops/config/grafana/values.yaml \
  --wait

# 설치 확인
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana
```

포트 포워딩:
```bash
kubectl port-forward svc/grafana -n monitoring 3000:80
```

`http://localhost:3000` 접속, `admin / grafana123`로 로그인.

---

## 2단계: 데이터소스 등록

### Prometheus 등록

```
Configuration → Data Sources → Add data source → Prometheus

URL: http://prometheus.monitoring.svc.cluster.local:9090
HTTP Method: POST
Save & Test → "Data source is working" 확인
```

### Loki 등록

```
Configuration → Data Sources → Add data source → Loki

URL: http://loki.monitoring.svc.cluster.local:3100
Save & Test → "Data source connected and labels found" 확인
```

---

## 3단계: Explore로 데이터 확인

### 메트릭 확인 (Prometheus)

```
Explore → Data source: Prometheus

쿼리:
up{namespace="monitoring"}
```

→ 각 서비스가 `1` (up) 상태인지 확인합니다.

### 로그 확인 (Loki)

```
Explore → Data source: Loki

쿼리:
{namespace="default"}
```

→ 앱 로그가 수집되는지 확인합니다.

---

## 4단계: 첫 번째 대시보드 만들기

### 4-1. 새 대시보드 생성

```
Dashboards → New → New dashboard → Add visualization
```

### 4-2. CPU 사용률 패널

```
Data source: Prometheus
Query:
  sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="default", container!=""}[5m]))
Legend: {{pod}}

Visualization: Time series
Title: Pod CPU Usage
Standard options → Unit: short (Cores)
```

### 4-3. 메모리 사용률 패널

```
Add panel → Add visualization

Query:
  sum by (pod) (container_memory_working_set_bytes{namespace="default", container!=""})
Legend: {{pod}}

Visualization: Time series
Title: Pod Memory Usage
Unit: bytes (IEC)
```

### 4-4. 에러 로그 수 패널

```
Add panel → Add visualization

Data source: Loki
Query:
  sum(count_over_time({namespace="default"} |= "ERROR" [5m]))

Visualization: Stat
Title: Error Logs (last 5m)
```

### 4-5. 에러 로그 패널

```
Add panel → Add visualization

Data source: Loki
Query:
  {namespace="default"} |= "ERROR"

Visualization: Logs
Title: Error Logs
Options → Deduplication: Exact
```

저장:
```
Save dashboard → Name: "My App Overview" → Save
```

---

## 5단계: 템플릿 변수 추가

```
Dashboard Settings → Variables → Add variable
```

**namespace 변수**:
```
Type: Query
Name: namespace
Data source: Prometheus
Query: label_values(kube_namespace_labels, namespace)
Refresh: On time range change
```

**pod 변수**:
```
Type: Query
Name: pod
Data source: Prometheus
Query: label_values(kube_pod_info{namespace="$namespace"}, pod)
Multi-value: true
Include All: true
Refresh: On time range change
```

패널 쿼리를 변수를 사용하도록 수정:
```promql
# Before
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="default", container!=""}[5m]))

# After
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="$namespace", pod=~"$pod", container!=""}[5m]))
```

---

## 6단계: 커뮤니티 대시보드 Import

```bash
# Node Exporter Full 대시보드 import
GRAFANA_URL="http://localhost:3000"

curl -s -X POST "${GRAFANA_URL}/api/dashboards/import" \
  -u admin:grafana123 \
  -H "Content-Type: application/json" \
  -d '{
    "gnetId": 1860,
    "overwrite": true,
    "inputs": [{"name": "DS_PROMETHEUS", "type": "datasource", "pluginId": "prometheus", "value": "Prometheus"}]
  }' | jq '.importedUrl'
```

또는 UI:
```
Dashboards → Import → 1860 → Load → Prometheus 선택 → Import
```

---

## 7단계: Alert Rule 생성

### 7-1. Contact Point 추가

```
Alerting → Contact points → Add contact point

Name: slack-test
Type: Slack
Webhook URL: https://hooks.slack.com/services/...
Channel: #alerts-test
Test → 테스트 메시지 전송 확인
```

### 7-2. Alert Rule 생성

```
Alerting → Alert rules → New alert rule

Rule name: HighErrorLogs
Folder: Practice

Query A (Loki):
  sum(count_over_time({namespace="default"} |= "ERROR" [5m]))

Expression C:
  Threshold: A IS ABOVE 5

Evaluate every: 1m
For: 2m

Labels:
  severity: warning

Annotations:
  summary: "에러 로그 급증"
  description: "최근 5분간 {{ $values.A.Value }}건의 에러 로그가 발생했습니다."

Save rule
```

### 7-3. Notification Policy 연결

```
Alerting → Notification policies → Root policy 편집
Default contact point: slack-test
```

---

## 8단계: Provisioning 설정

```bash
# 대시보드 JSON 내보내기
DASHBOARD_UID=$(curl -s http://admin:grafana123@localhost:3000/api/search?query=My+App+Overview | jq -r '.[0].uid')

curl -s http://admin:grafana123@localhost:3000/api/dashboards/uid/${DASHBOARD_UID} | \
  jq '.dashboard | del(.id, .version)' \
  > ../ops/config/grafana/provisioning/dashboards/my-app-overview.json

# Git 커밋
git add ../ops/config/grafana/provisioning/dashboards/my-app-overview.json
git commit -m "feat: add My App Overview dashboard"
```

---

## 실습 결과 확인

```bash
# 1. Grafana 파드 상태
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana

# 2. 데이터소스 목록
curl -s http://admin:grafana123@localhost:3000/api/datasources | \
  jq '.[] | {name: .name, type: .type}'

# 3. 대시보드 목록
curl -s http://admin:grafana123@localhost:3000/api/search | \
  jq '.[] | {title: .title, uid: .uid}'

# 4. 현재 Alert 상태
curl -s http://admin:grafana123@localhost:3000/api/alertmanager/grafana/api/v2/alerts | \
  jq '.[] | {alertname: .labels.alertname, status: .status.state}'
```

---

## 완료 체크리스트

- [ ] Grafana 설치 및 접속 확인
- [ ] Prometheus 데이터소스 등록
- [ ] Loki 데이터소스 등록
- [ ] Explore에서 메트릭/로그 조회 확인
- [ ] 대시보드 생성 (Time series, Stat, Logs 패널)
- [ ] 템플릿 변수 추가 (namespace, pod)
- [ ] 커뮤니티 대시보드 Import
- [ ] Contact Point 생성
- [ ] Alert Rule 생성 및 Pending 상태 확인
- [ ] 대시보드 JSON을 Git으로 관리
