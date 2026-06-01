# Grafana Docs

Grafana를 처음 보는 사람이 설치부터 대시보드 운영까지 따라갈 수 있도록 문서를 묶어 둔 디렉터리다.

## 어디서 시작할까

| 순서 | 문서 | 용도 |
|------|------|------|
| 1 | `install.md` | Helm, systemd, Docker Compose 설치 방식 |
| 2 | `datasource-guide.md` | Prometheus/Mimir, Loki, Tempo 데이터소스 |
| 3 | `dashboard-guide.md`, `dashboard-design-guide.md` | 대시보드 생성과 설계 원칙 |
| 4 | `variable-guide.md`, `useful-dashboards.md` | 템플릿 변수와 자주 쓰는 대시보드 |
| 5 | `promql-guide.md`, `logql-guide.md`, `explore-guide.md` | 탐색과 쿼리 |
| 6 | `alerting-guide.md`, `provisioning-guide.md` | 알림과 GitOps 프로비저닝 |
| 7 | `slo-guide.md`, `oncall-runbook.md`, `performance-analysis-guide.md` | SLO와 장애 대응 |
| 8 | `rules/README.md` | 문서와 운영 규칙 |
| 9 | `agents/README.md` | AI 작업 지침 |
| 10 | `templates/README.md` | 문서 템플릿 |
| 11 | `../ops/README.md` | 실제 실행 자산과 운영 방법 |

## 문서 구조

| 구분 | 문서 |
|------|------|
| 설치/기초 | `install.md`, `datasource-guide.md` |
| 대시보드 | `dashboard-guide.md`, `dashboard-design-guide.md`, `variable-guide.md`, `useful-dashboards.md` |
| 쿼리/탐색 | `promql-guide.md`, `logql-guide.md`, `explore-guide.md` |
| 운영 | `alerting-guide.md`, `provisioning-guide.md`, `slo-guide.md`, `oncall-runbook.md` |
| 보조 자료 | `rules/`, `agents/`, `templates/` |

## 읽는 순서

1. `install.md`
2. `datasource-guide.md`
3. `dashboard-guide.md`
4. `dashboard-design-guide.md`
5. `variable-guide.md`
6. `explore-guide.md`
7. `alerting-guide.md`
8. `provisioning-guide.md`
9. `slo-guide.md`
10. `rules/README.md`
11. `agents/README.md`
12. `templates/README.md`

## 관련 경로

- `rules/`는 문서/운영 규칙
- `agents/`는 Claude 작업 지침
- `templates/`는 반복 문서 골격
- `../ops/`는 Grafana Helm values와 provisioning 예시
