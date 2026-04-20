새 Grafana 가이드 문서를 생성합니다.

**사용법**: `/new-doc <주제명>`  **예시**: `/new-doc tempo-tracing`

주제 분류: dashboard, datasource, alerting, provisioning, slo, variable, oncall

`<주제명>-guide.md` 생성 시 포함 내용:
- CLAUDE.md 환경 설정 반영 (EKS, Grafana 버전)
- JSON 패널/대시보드 예시 또는 UI 설정 단계
- PromQL/LogQL/TraceQL 쿼리 예시
- Grafana API 또는 provisioning YAML 자동화
- 트러블슈팅 섹션
