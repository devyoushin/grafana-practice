# 대시보드 가이드

Grafana 대시보드는 여러 패널을 조합하여 메트릭·로그·트레이스를 하나의 화면에서 시각화합니다.

---

## 1. 대시보드 생성

```
Dashboards → New → New dashboard → Add visualization
```

---

## 2. 패널 타입

| 패널 타입 | 적합한 데이터 | 대표 사용 예 |
|---------|------------|------------|
| **Time series** | 시계열 메트릭 | CPU 사용률, 요청 수 추이 |
| **Stat** | 단일 최신값 | 현재 에러율, 응답 중인 인스턴스 수 |
| **Gauge** | 비율 (0~100%) | 메모리 사용률, 디스크 사용량 |
| **Bar chart** | 카테고리별 비교 | 서비스별 요청 수 비교 |
| **Table** | 다차원 데이터 | 파드별 CPU·메모리·재시작 수 |
| **Logs** | 로그 스트림 (Loki) | 에러 로그 실시간 표시 |
| **Heatmap** | 분포 변화 | 레이턴시 히스토그램 시간별 분포 |
| **Node Graph** | 토폴로지 | 서비스 의존성 맵 |
| **Text** | 마크다운 | 대시보드 설명, 런북 링크 |

---

## 3. 패널 편집

패널 제목 클릭 → **Edit** 또는 패널 우측 상단 `⋮` → Edit.

### 쿼리 탭

```
데이터소스 선택 → 쿼리 입력 → Legend 설정
```

**PromQL 예시** (Time series):
```promql
# Legend: {{pod}}
rate(container_cpu_usage_seconds_total{namespace="$namespace", pod=~"$pod"}[5m])
```

**LogQL 예시** (Logs 패널):
```logql
{namespace="$namespace", app="$app"} |= "$search"
```

### Transform 탭

쿼리 결과를 가공할 때 사용합니다.

| 변환 | 용도 |
|------|------|
| **Reduce** | 시계열 → 단일값 (last, mean, max) |
| **Merge** | 여러 쿼리 결과를 하나의 테이블로 합침 |
| **Filter by value** | 특정 임계값 이상/이하 행만 표시 |
| **Rename by regex** | 레이블 이름을 정규식으로 변환 |
| **Group by** | 레이블 기준으로 집계 |

### Override 탭

특정 시리즈에만 다른 색상, 단위, 축을 적용합니다.

```
Overrides → Add field override → Fields with name: error_rate
→ Add override property → Standard options > Color: Red
```

---

## 4. 주요 패널 설정

### Time series 패널

```
Options:
  Graph styles → Lines / Bars / Points
  Line width: 2
  Fill opacity: 10

Standard options:
  Unit: percent (0-100) / bytes / requests/sec
  Min: 0
  Decimals: 2

Thresholds:
  Base: green
  80 → yellow
  95 → red
```

### Stat 패널

```
Options:
  Calculation: Last (not null)   # 최신값
  Color mode: Background         # 배경색으로 상태 표시
  Graph mode: Area               # 미니 그래프 함께 표시

Standard options:
  Unit: short / percent / bytes
```

### Table 패널

```
쿼리에 instant=true 설정 (Time series 대신 즉시 스냅샷 조회)

Column 설정:
  - Override → 특정 컬럼에 색상 임계값 설정
  - Rename fields로 컬럼 이름 변경
```

---

## 5. 시간 범위 및 새로고침

대시보드 우측 상단:
- 시간 범위: `Last 1h`, `Last 6h`, `Last 24h`, `Custom range`
- 자동 새로고침: `5s`, `30s`, `1m`, `5m`, `Off`

> 프로덕션 모니터링 대시보드는 **30s** 새로고침을 권장합니다. 너무 짧으면 데이터소스에 부하가 생깁니다.

---

## 6. 어노테이션 (Annotations)

배포 이벤트 등 외부 이벤트를 그래프에 수직선으로 표시합니다.

```
Dashboard Settings → Annotations → Add annotation query

Name: Deployments
Data source: Prometheus
Query: changes(kube_deployment_status_replicas_updated{namespace="$namespace"}[2m]) > 0
```

---

## 7. 커뮤니티 대시보드 Import

```
Dashboards → Import → Grafana.com ID 입력 → Load
```

자주 사용하는 대시보드:

| 이름 | ID | 용도 |
|------|-----|------|
| Node Exporter Full | 1860 | 노드 리소스 전체 |
| Kubernetes / Compute Resources / Cluster | 15757 | 클러스터 전체 리소스 |
| Kubernetes / Compute Resources / Namespace | 15758 | 네임스페이스별 리소스 |
| Kubernetes / Compute Resources / Pod | 15759 | 파드별 리소스 |
| Loki Dashboard | 12611 | 로그 검색 UI |
| Mimir / Overview | 15869 | Mimir 전체 상태 |

```bash
# API로 커뮤니티 대시보드 import
GRAFANA_URL="http://localhost:3000"
DASHBOARD_ID=1860
DS_NAME="Prometheus"

curl -s -X POST "${GRAFANA_URL}/api/dashboards/import" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d "{
    \"gnetId\": ${DASHBOARD_ID},
    \"overwrite\": true,
    \"inputs\": [{
      \"name\": \"DS_PROMETHEUS\",
      \"type\": \"datasource\",
      \"pluginId\": \"prometheus\",
      \"value\": \"${DS_NAME}\"
    }]
  }" | jq '.importedUrl'
```

---

## 8. 대시보드 저장 및 공유

```bash
# 대시보드 JSON 내보내기 (백업용)
curl -s http://admin:admin@localhost:3000/api/dashboards/uid/<UID> | \
  jq '.dashboard' > my-dashboard.json

# 대시보드 JSON 가져오기
curl -s -X POST http://admin:admin@localhost:3000/api/dashboards/import \
  -H "Content-Type: application/json" \
  -d "{\"dashboard\": $(cat my-dashboard.json), \"overwrite\": true, \"folderId\": 0}"
```

> 대시보드를 Git으로 관리하려면 [provisioning-guide.md](./provisioning-guide.md)를 참고하세요.

---

## 참고

- [공식문서 - Panels](https://grafana.com/docs/grafana/latest/panels-visualizations/)
- [공식문서 - Dashboard](https://grafana.com/docs/grafana/latest/dashboards/)
- [Grafana 공식 대시보드 목록](https://grafana.com/grafana/dashboards/)
