새 Grafana 운영 런북을 생성합니다.

**사용법**: `/new-runbook <작업명>`  **예시**: `/new-runbook 대시보드 마이그레이션`

작업 유형: 대시보드 provisioning, 데이터소스 추가, 알람 채널 설정, 업그레이드

런북 포함 내용:
- 사전 체크리스트 (현재 대시보드/알람 상태)
- Grafana API 또는 kubectl 명령어
- provisioning ConfigMap 변경 방법
- 롤백 절차 (대시보드 버전 복구)
