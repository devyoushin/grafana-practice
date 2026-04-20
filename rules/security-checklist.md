# 보안 체크리스트 — grafana-practice

## Grafana 보안 설정

### 인증/인가
- [ ] 기본 admin 비밀번호 변경 (초기 설치 즉시)
- [ ] LDAP 또는 OAuth 연동 (개인 계정 공유 금지)
- [ ] 역할 기반 접근 제어 (Viewer/Editor/Admin 분리)
- [ ] Service Account 토큰으로 API 접근 (user/password 금지)
- [ ] 익명 접근 비활성화 (`auth.anonymous.enabled = false`)

### 데이터소스 보안
- [ ] 데이터소스 자격증명 암호화 저장 확인
- [ ] 데이터소스별 최소 권한 (read-only 원칙)
- [ ] 프록시 모드 사용 (브라우저에 자격증명 노출 금지)

### 네트워크 보안
- [ ] Grafana 포트(3000) 외부 직접 노출 금지 (Ingress/LB 경유)
- [ ] HTTPS 강제 (`protocol = https`)
- [ ] Content Security Policy 설정

### 플러그인 보안
- [ ] 공식 플러그인만 설치 (`allow_loading_unsigned_plugins` 금지)
- [ ] 불필요한 플러그인 제거
- [ ] 플러그인 정기 업데이트 확인

## 정기 보안 점검 (월별)
- [ ] 비활성 사용자 계정 비활성화
- [ ] Service Account 토큰 만료/순환
- [ ] 공유 대시보드 외부 공개 여부 확인
- [ ] Grafana 버전 보안 업데이트 확인
