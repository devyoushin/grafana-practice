# Grafana 컨벤션 규칙

## 대시보드 네이밍
- 형식: `[도메인] [서비스명] Overview` 또는 `[도메인] [서비스명] Details`
- 예: `Kubernetes Pod Overview`, `Application API Details`
- 태그 필수: 도메인(kubernetes, app, infra), 환경(prod, stg, dev)

## 폴더 구조
```
Grafana/
├── Infrastructure/     # 인프라 메트릭
├── Kubernetes/         # K8s 클러스터
├── Applications/       # 서비스별 대시보드
├── Observability/      # 모니터링 시스템 자체
└── Alerting/           # 알림 상태
```

## 패널 배치 원칙
1. 최상단: 핵심 SLI (Error Rate, Latency, Availability) — Stat 패널
2. 중간: 시계열 트렌드 — Time series 패널
3. 하단: 상세 메트릭 및 로그 링크

## 변수(Variables) 규칙
- `datasource`: 항상 첫 번째 변수
- `namespace`, `job`, `instance`: 표준 필터 변수
- 변수명은 snake_case
- 다중 선택(Multi-value) 활성화 기본값

## 알림 규칙 컨벤션
- 알림명: `[서비스] [증상]` (한국어 가능)
- Pending 기간: warning은 5m, critical은 1m
- 모든 알림에 `runbook_url` 어노테이션 필수
- Contact point: severity 기반 자동 라우팅

## Provisioning 규칙
- 모든 대시보드는 ConfigMap으로 프로비저닝
- UID는 고정값 사용 (자동 생성 금지)
- `editable: false` (프로덕션 대시보드)
