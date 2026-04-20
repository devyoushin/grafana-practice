# [알림명] — Grafana 런북

> 심각도: Critical / Warning / Info
> 담당팀: | 최종 수정: YYYY-MM-DD

## 알림 요약
<!-- 이 알림이 무엇을 감지하는지 한 문장으로 -->

## 알림 조건
<!-- Grafana Alerting 조건 설명 -->

## 영향도
<!-- 이 알림이 발생했을 때 사용자/서비스에 미치는 영향 -->

## 즉시 확인 사항

```bash
# 1. Grafana 상태 확인
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana

# 2. Grafana 로그 확인
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana --tail=50

# 3. 데이터소스 연결 확인
curl http://grafana:3000/api/datasources
```

## 진단 단계

### 1단계: 알림 상태 확인
<!-- Grafana Alerting UI에서 확인할 사항 -->

### 2단계: 원인 분석
<!-- 일반적인 원인 목록 -->

### 3단계: 해결 조치
<!-- 단계별 해결 방법 -->

## 에스컬레이션
- 15분 내 해결 불가 시: 팀 리드 호출
- 서비스 영향 발생 시: 인시던트 선언

## 참고
- 관련 대시보드:
- 관련 문서:
