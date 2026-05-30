# grafana-practice

EKS + Grafana 11.x 기준으로 데이터소스, 대시보드, 변수, Explore, Alerting, Provisioning, SLO/온콜 운영을 정리한 개인 학습 문서입니다.

## 빠른 시작

- 처음 볼 문서: `docs/install.md`
- 전체 흐름: 설치 -> 데이터소스 -> 대시보드/변수 -> Explore/Alerting -> Provisioning -> SLO/운영
- AI 작업 지침: `CLAUDE.md`

## 구조

```text
grafana-practice/
├── README.md
├── CLAUDE.md
├── docs/
│   ├── README.md
│   ├── agents/
│   ├── rules/
│   ├── templates/
│   └── *.md
└── ops/
    ├── README.md
    └── config/     # Grafana Helm values와 provisioning 예시
```

## 학습 경로

| 단계 | 문서 |
|------|------|
| 설치 | `docs/install.md` |
| 핵심 개념 | `docs/datasource-guide.md`, `docs/dashboard-guide.md` |
| 대시보드 | `docs/dashboard-design-guide.md`, `docs/variable-guide.md`, `docs/useful-dashboards.md` |
| 쿼리/탐색 | `docs/promql-guide.md`, `docs/logql-guide.md`, `docs/explore-guide.md` |
| 운영 | `docs/alerting-guide.md`, `docs/provisioning-guide.md`, `docs/slo-guide.md` |
| 장애 대응 | `docs/performance-analysis-guide.md`, `docs/oncall-runbook.md`, `docs/practice.md` |

## 환경

| 항목 | 값 |
|------|-----|
| Platform | EKS |
| Grafana | 11.x |
| Namespace | `monitoring` |
| 주요 데이터소스 | Prometheus/Mimir, Loki, Tempo |
