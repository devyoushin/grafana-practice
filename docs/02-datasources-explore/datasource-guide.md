# 데이터소스 설정 가이드

Grafana는 데이터를 직접 저장하지 않습니다. Prometheus, Loki, Tempo 등 외부 시스템에 쿼리를 위임하고 결과를 시각화합니다.

---

## 데이터소스 개요

| 데이터소스 | 쿼리 언어 | 용도 |
|-----------|----------|------|
| Prometheus / Mimir | PromQL | 메트릭 (시계열) |
| Loki | LogQL | 로그 |
| Tempo | TraceQL | 분산 트레이스 |
| CloudWatch | — | AWS 메트릭 / 로그 |

---

## 1. Prometheus 데이터소스

### UI에서 추가

```
Configuration → Data Sources → Add data source → Prometheus
```

| 설정 항목 | 값 | 설명 |
|---------|---|------|
| Name | `Prometheus` | 데이터소스 이름 |
| URL | `http://prometheus.monitoring.svc.cluster.local:9090` | 클러스터 내부 주소 |
| HTTP Method | `POST` | 긴 쿼리 지원 |
| Scrape interval | `15s` | Prometheus scrape_interval과 일치시킬 것 |

**Save & Test** 후 `Data source is working` 확인.

---

### Mimir를 Prometheus 타입으로 추가

Mimir는 Prometheus 호환 API를 제공합니다. 추가 설정 항목:

| 설정 항목 | 값 |
|---------|---|
| URL | `http://mimir-nginx.monitoring.svc.cluster.local/prometheus` |
| Prometheus type | `Mimir` |
| Custom HTTP Header | `X-Scope-OrgID: default` |

```
Advanced settings → Custom HTTP Headers
Header: X-Scope-OrgID
Value: default
```

> `X-Scope-OrgID` 헤더는 Mimir의 **테넌트 ID**입니다. 단일 테넌트 환경에서는 `default`를 사용합니다.

---

## 2. Loki 데이터소스

### UI에서 추가

```
Configuration → Data Sources → Add data source → Loki
```

| 설정 항목 | 값 |
|---------|---|
| Name | `Loki` |
| URL | `http://loki.monitoring.svc.cluster.local:3100` |
| Maximum lines | `1000` |

**Derived fields** 설정 (로그에서 Trace ID 추출 → Tempo 연동):

```
Name: TraceID
Regex: (?:trace_id|traceID)=(\w+)
URL: ${__value.raw}
Internal link: Tempo (데이터소스 선택)
```

---

## 3. Tempo 데이터소스

### UI에서 추가

```
Configuration → Data Sources → Add data source → Tempo
```

| 설정 항목 | 값 |
|---------|---|
| Name | `Tempo` |
| URL | `http://tempo.monitoring.svc.cluster.local:3100` |

**Trace to logs** 설정 (트레이스에서 Loki 로그로 이동):

```
Data source: Loki
Tags: [{"key": "service.name", "value": "app"}]
```

**Trace to metrics** 설정 (트레이스에서 Prometheus 메트릭으로 이동):

```
Data source: Prometheus
```

---

## 4. 데이터소스 Health Check

```bash
# Grafana API로 모든 데이터소스 상태 확인
kubectl port-forward svc/grafana -n monitoring 3000:80 &

# API Key 또는 기본 인증 사용
curl -s http://admin:admin@localhost:3000/api/datasources | \
  jq '.[] | {name: .name, type: .type, url: .url}'

# 특정 데이터소스 Health Check
curl -s http://admin:admin@localhost:3000/api/datasources/name/Prometheus/health | jq
```

---

## 5. 데이터소스 간 연동 흐름

```
[Grafana Explore / Dashboard]
          │
          ├── PromQL ──▶ Prometheus / Mimir
          │                 │ Exemplars
          │                 ▼
          ├── TraceQL ──▶ Tempo ◀── Trace ID
          │                 │ Logs
          │                 ▼
          └── LogQL ───▶ Loki
```

- **Metrics → Traces**: Prometheus Exemplar에 포함된 Trace ID로 Tempo 조회
- **Traces → Logs**: Tempo 트레이스의 서비스명으로 Loki 로그 조회
- **Logs → Traces**: 로그에서 추출한 Trace ID로 Tempo 조회 (Derived fields)

---

## 참고

- [공식문서 - Data sources](https://grafana.com/docs/grafana/latest/datasources/)
- [공식문서 - Correlations](https://grafana.com/docs/grafana/latest/administration/correlations/)
