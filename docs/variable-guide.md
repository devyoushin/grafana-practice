# 템플릿 변수 가이드

템플릿 변수를 사용하면 대시보드 상단에 드롭다운 필터를 추가하여 동적으로 쿼리를 변경할 수 있습니다.

---

## 1. 변수 타입

| 타입 | 설명 | 사용 예 |
|------|------|---------|
| **Query** | 데이터소스에서 값 목록을 동적으로 가져옴 | namespace, pod 목록 |
| **Custom** | 쉼표로 구분된 고정 값 목록 | `dev,staging,prod` |
| **Constant** | 상수 (대시보드에 표시 안됨) | 서비스 기본 URL |
| **Datasource** | 등록된 데이터소스 선택 | 멀티 클러스터 전환 |
| **Interval** | 시간 간격 | `1m,5m,10m,30m,1h` |
| **Text box** | 사용자가 직접 입력 | 검색어, 키워드 |
| **Ad hoc filters** | 레이블 기반 동적 필터 추가 | 자유 레이블 필터링 |

---

## 2. 변수 추가 방법

```
Dashboard Settings (⚙) → Variables → Add variable
```

---

## 3. Query 변수 (Prometheus)

### namespace 목록

```
Type: Query
Data source: Prometheus
Query: label_values(kube_namespace_labels, namespace)
Refresh: On time range change
Multi-value: true
Include All option: true, All value: .*
```

### pod 목록 (namespace 변수에 종속)

```
Type: Query
Data source: Prometheus
Query: label_values(kube_pod_info{namespace="$namespace"}, pod)
Refresh: On time range change
```

> `$namespace` 변수를 참조하면 namespace를 선택할 때마다 pod 목록이 자동으로 갱신됩니다.

### 컨테이너 목록 (pod 변수에 종속)

```
Query: label_values(kube_pod_container_info{namespace="$namespace", pod="$pod"}, container)
```

---

## 4. Query 변수 (Loki)

### namespace 목록 (로그 레이블에서)

```
Type: Query
Data source: Loki
Query: label_values(namespace)
```

### app 목록

```
Query: label_values({namespace="$namespace"}, app)
```

---

## 5. Interval 변수

긴 시간 범위에서 집계 단위를 동적으로 조정합니다.

```
Type: Interval
Name: interval
Values: 1m,5m,10m,30m,1h,6h,12h,24h
Auto option: Enable
Step count: 50
```

쿼리에서 사용:
```promql
rate(http_requests_total{namespace="$namespace"}[$__rate_interval])
```

> `$__rate_interval`은 Grafana가 현재 시간 범위에 맞게 자동으로 계산해주는 내장 변수입니다.

---

## 6. 변수 사용 예시

### 기본 사용

```promql
# $변수명 또는 ${변수명} 형태로 사용
container_memory_working_set_bytes{
  namespace="$namespace",
  pod=~"$pod"
}
```

### 멀티값 변수 (regex)

멀티값 선택 시 `pod` 변수가 `pod1|pod2|pod3` 형태로 확장됩니다.

```promql
# pod 변수가 멀티값일 때 =~ 사용
container_cpu_usage_seconds_total{pod=~"$pod"}
```

### LogQL에서 사용

```logql
{namespace="$namespace", app="$app"} |= "$search_keyword"
```

---

## 7. 내장 변수 (Built-in Variables)

| 변수 | 설명 | 예시 값 |
|------|------|---------|
| `$__from` | 시간 범위 시작 (ms) | `1712000000000` |
| `$__to` | 시간 범위 끝 (ms) | `1712003600000` |
| `$__interval` | 자동 계산된 쿼리 간격 | `1m` |
| `$__rate_interval` | rate() 함수에 적합한 간격 | `4m` |
| `$__range` | 현재 시간 범위 길이 | `6h` |
| `$__dashboard` | 현재 대시보드 이름 | `My Dashboard` |

```promql
# 시간 범위에 맞게 자동 조정
rate(http_requests_total[${__rate_interval}])

# 1시간 단위로 집계
sum_over_time(up[$__range])
```

---

## 8. 패널 반복 (Repeat)

하나의 패널을 변수 값 개수만큼 자동으로 반복 생성합니다.

```
Panel → Edit → Panel options → Repeat options
  Repeat by variable: namespace
  Max per row: 3
  Min width: 8
```

→ namespace가 `dev`, `staging`, `prod` 3개면 동일한 패널이 3개 나란히 생성됩니다.

---

## 9. 변수 체이닝 예시

```
namespace (Query, Prometheus) ← 전체 namespace 목록
    └── app (Query, Prometheus) ← 해당 namespace의 app 목록
            └── pod (Query, Prometheus) ← 해당 app의 pod 목록

대시보드 쿼리:
container_cpu_usage_seconds_total{
  namespace="$namespace",
  app="$app",
  pod=~"$pod"
}
```

---

## 참고

- [공식문서 - Variables](https://grafana.com/docs/grafana/latest/dashboards/variables/)
- [공식문서 - Variable syntax](https://grafana.com/docs/grafana/latest/dashboards/variables/variable-syntax/)
