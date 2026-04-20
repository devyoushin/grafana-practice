# grafana-practice — 프로젝트 가이드

## 프로젝트 설정
- 환경: EKS
- Grafana 버전: 11.x (Helm chart: grafana/grafana)
- 네임스페이스: monitoring
- 앱 이름: grafana

---

## 디렉토리 구조

```
grafana-practice/
├── CLAUDE.md                  # 이 파일 (자동 로드)
├── .claude/
│   ├── settings.json
│   └── commands/              # /new-doc, /new-runbook, /review-doc, /add-troubleshooting, /search-kb
├── agents/                    # doc-writer, dashboard-designer, alerting-advisor, troubleshooter
├── templates/                 # service-doc, runbook, incident-report
├── rules/                     # doc-writing, grafana-conventions, security-checklist, monitoring
├── grafana/                   # 대시보드 JSON, provisioning 설정
└── *-guide.md                 # 주제별 가이드 문서
```

---

## 커스텀 슬래시 명령어

| 명령어 | 설명 | 사용 예시 |
|--------|------|---------|
| `/new-doc` | 새 가이드 문서 생성 | `/new-doc grafana-oncall-integration` |
| `/new-runbook` | 새 런북 생성 | `/new-runbook 대시보드 패널 No Data 대응` |
| `/review-doc` | 문서 검토 | `/review-doc dashboard-design-guide.md` |
| `/add-troubleshooting` | 트러블슈팅 케이스 추가 | `/add-troubleshooting 데이터소스 연결 실패` |
| `/search-kb` | 지식베이스 검색 | `/search-kb Grafana 알림 라우팅` |

---

## 가이드 문서 목록

| 문서 | 주제 |
|------|------|
| `install.md` | Grafana 설치 (Helm + EKS) |
| `datasource-guide.md` | 데이터소스 설정 (Prometheus, Loki, Tempo) |
| `dashboard-guide.md` | 대시보드 기본 사용법 |
| `dashboard-design-guide.md` | 대시보드 설계 패턴 |
| `variable-guide.md` | 대시보드 변수(Variables) |
| `provisioning-guide.md` | 대시보드 프로비저닝 (ConfigMap) |
| `alerting-guide.md` | Grafana Unified Alerting |
| `explore-guide.md` | Explore 기능 활용 |
| `logql-guide.md` | LogQL (Explore + Loki) |
| `promql-guide.md` | PromQL (Explore + Prometheus) |
| `slo-guide.md` | SLO 대시보드 구성 |
| `performance-analysis-guide.md` | 성능 분석 대시보드 |
| `useful-dashboards.md` | 유용한 커뮤니티 대시보드 |
| `oncall-runbook.md` | OnCall 런북 |

---

## 핵심 명령어

```bash
# Grafana 상태 확인
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana

# Grafana 로그 확인
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana --tail=50

# 데이터소스 API 확인
curl -u admin:password http://grafana:3000/api/datasources

# 대시보드 목록
curl -u admin:password http://grafana:3000/api/search?type=dash-db
```
