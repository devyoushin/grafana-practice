Grafana 트러블슈팅 케이스를 추가합니다.

**사용법**: `/add-troubleshooting <증상 설명>`  **예시**: `/add-troubleshooting 대시보드 패널 No Data`

형식:
```markdown
### <증상>
**원인**: <근본 원인>
**확인**:
\`\`\`bash
kubectl logs -n monitoring deployment/grafana
curl -u admin:<pw> http://grafana/api/health
\`\`\`
**해결**: <해결 방법>
```
`troubleshooting-guide.md` 또는 관련 가이드에 추가하세요.
