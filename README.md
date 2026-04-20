# grafana-practice

A hands-on repository for learning Grafana on EKS.
- **Environment**: EKS / Grafana 11.x
- **Namespaces**: monitoring stack `monitoring`, app `default`

---

## Learning Path

```
1. Installation    → install.md
2. Core Concepts   → datasource-guide.md, dashboard-guide.md
3. Advanced
   ├── Variables    → variable-guide.md
   ├── Alerting     → alerting-guide.md
   ├── Explore      → explore-guide.md
   └── Provisioning → provisioning-guide.md
4. 실무
   ├── 대시보드 설계    → dashboard-design-guide.md
   ├── PromQL 레퍼런스  → promql-guide.md
   ├── LogQL 레퍼런스   → logql-guide.md
   ├── 추천 대시보드    → useful-dashboards.md
   ├── SLO/Error Budget → slo-guide.md
   ├── 성능 분석        → performance-analysis-guide.md
   └── 장애 대응 (온콜) → oncall-runbook.md
5. Hands-on        → practice.md
```

---

## Documents

### Installation
| File | Description |
|------|-------------|
| [install.md](./install.md) | Grafana를 Helm으로 EKS에 설치 |

### Core Concepts
| File | Description |
|------|-------------|
| [datasource-guide.md](./datasource-guide.md) | 데이터소스 설정 — Prometheus/Mimir, Loki, Tempo |
| [dashboard-guide.md](./dashboard-guide.md) | 대시보드 생성 — 패널, 시각화, 레이아웃 |

### Advanced
| File | Description |
|------|-------------|
| [variable-guide.md](./variable-guide.md) | 템플릿 변수 — 동적 대시보드 구성 |
| [alerting-guide.md](./alerting-guide.md) | Grafana Alerting — alert rules, contact points, notification policies |
| [explore-guide.md](./explore-guide.md) | Explore — PromQL/LogQL 실시간 쿼리 및 상관 분석 |
| [provisioning-guide.md](./provisioning-guide.md) | Provisioning as Code — 데이터소스·대시보드 GitOps 관리 |

### 실무
| File | Description |
|------|-------------|
| [dashboard-design-guide.md](./dashboard-design-guide.md) | 대시보드 설계 — USE/RED/Golden Signals, 레이아웃 원칙, 안티패턴 |
| [promql-guide.md](./promql-guide.md) | 실무 PromQL — Kubernetes, HTTP, 인프라 쿼리 모음 |
| [logql-guide.md](./logql-guide.md) | 실무 LogQL — 파싱, 집계, 장애 분석, 알림용 쿼리 모음 |
| [useful-dashboards.md](./useful-dashboards.md) | 추천 커뮤니티 대시보드 — 레이어별 ID 목록과 실무 활용 팁 |
| [slo-guide.md](./slo-guide.md) | SLO/Error Budget — Burn Rate 알림, SLO 대시보드 구성 |
| [performance-analysis-guide.md](./performance-analysis-guide.md) | 성능 병목 분석 — CPU/GC/DB/네트워크 원인 파악 절차 |
| [oncall-runbook.md](./oncall-runbook.md) | 장애 대응 — 증상별 체크리스트, RCA 절차, Annotation 활용 |

### Hands-on
| File | Description |
|------|-------------|
| [practice.md](./practice.md) | 종합 실습 (설치 → 데이터소스 → 대시보드 → 알림) |

---

## Manifest Structure

```
grafana/
├── values.yaml                          # Grafana Helm values
└── provisioning/
    ├── datasources.yaml                 # 데이터소스 자동 프로비저닝
    ├── dashboards.yaml                  # 대시보드 프로비저닝 설정
    └── dashboards/
        └── sample-dashboard.json        # 샘플 대시보드 JSON
```

---

## Key Concept Summary

Grafana는 **Prometheus(메트릭), Loki(로그), Tempo(트레이스)**를 하나의 UI에서 연결하는 통합 가시성 플랫폼입니다.

```
[Prometheus / Mimir]  → 메트릭 데이터소스  ─┐
[Loki]               → 로그 데이터소스    ──┤──▶ [Grafana]  → 대시보드 / 알림 / Explore
[Tempo]              → 트레이스 데이터소스 ─┘
```

> Grafana는 데이터를 **저장하지 않습니다**. 각 데이터소스에 쿼리를 위임하고 결과를 시각화합니다.
