# 실무 LogQL 가이드

실무에서 자주 쓰는 LogQL 패턴을 모아놓은 레퍼런스입니다.

---

## 1. 파싱 방법 선택

| 방법 | 대상 로그 형식 | 특징 |
|------|-------------|------|
| `\| json` | `{"level":"error","msg":"..."}` | JSON 키를 레이블로 자동 추출 |
| `\| logfmt` | `level=error msg="..."` | key=value 형식 자동 파싱 |
| `\| pattern` | 고정된 텍스트 구조 | 위치 기반 추출, 빠름 |
| `\| regexp` | 비정형 로그 | 정규식으로 추출 |
| `\| unpack` | Promtail JSON 래핑 | Promtail의 pack stage 언랩 |

---

## 2. JSON 파싱

```logql
# 기본 JSON 파싱 (모든 최상위 키가 레이블로 추출)
{app="my-app"} | json

# 특정 필드만 추출 (성능 좋음)
{app="my-app"} | json level, status_code, duration_ms

# 중첩 JSON 추출
{app="my-app"} | json
  | json request_method="request.method", request_path="request.path"

# 파싱 후 필터
{app="my-app"} | json | level = "error"
{app="my-app"} | json | status_code >= 500
{app="my-app"} | json | duration_ms > 1000

# 문자열 + 숫자 동시 필터
{app="my-app"} | json | level = "error" | duration_ms > 500
```

---

## 3. logfmt 파싱

```logql
# logfmt 기본 (ts=... level=... msg=...)
{app="my-app"} | logfmt

# 파싱 후 레이블 필터
{app="my-app"} | logfmt | level = "error"

# method와 path 추출
{app="my-app"} | logfmt | method = "POST" | path =~ "/api/.*"
```

---

## 4. Pattern 파싱 (고정 형식)

```logql
# Apache/Nginx 액세스 로그 파싱
# 예: 192.168.1.1 - - [14/Apr/2026:02:00:00] "GET /api/health HTTP/1.1" 200 512
{job="nginx"} | pattern `<ip> - - [<_>] "<method> <path> <_>" <status> <size>`
  | status >= 500

# 공백 구분 고정 형식
# 예: 2026-04-14 02:00:00 ERROR my-app Request failed
{app="my-app"} | pattern `<ts> <level> <app> <msg>`
  | level = "ERROR"
```

---

## 5. 정규식 추출

```logql
# 특정 값 추출
{app="my-app"} | regexp `user_id=(?P<user_id>\d+)`
  | user_id = "12345"

# 여러 필드 추출
{app="my-app"} | regexp `(?P<method>[A-Z]+) (?P<path>/\S+) .* (?P<status>\d{3})`
  | status >= "500"

# Trace ID 추출 (16진수)
{namespace="default"} | regexp `(?P<trace_id>[0-9a-f]{32})`
```

---

## 6. 레이블 가공

```logql
# 레이블 이름 변경
{app="my-app"} | json | label_format level=severity

# 레이블 값 변환 (소문자로)
{app="my-app"} | json | label_format level=`{{.level | ToLower}}`

# 레이블 조합으로 새 레이블 생성
{namespace="default"} | json
  | label_format service=`{{.namespace}}/{{.app}}`

# line_format으로 출력 형식 변경 (Explore에서 가독성 향상)
{app="my-app"} | json
  | line_format `[{{.level}}] {{.message}} ({{.duration_ms}}ms)`
```

---

## 7. 숫자 메트릭 추출 (unwrap)

로그의 숫자 필드를 추출하여 집계합니다.

```logql
# duration_ms의 평균 응답 시간
avg_over_time(
  {app="my-app"} | json | unwrap duration_ms [5m]
)

# P99 응답 시간 (로그에서 계산)
quantile_over_time(0.99,
  {app="my-app"} | json | unwrap duration_ms [5m]
) by (handler)

# 최대 응답 시간
max_over_time(
  {app="my-app"} | json | unwrap duration_ms [5m]
) by (pod)

# 바이트 단위 총 응답 크기 합계
sum_over_time(
  {job="nginx"} | pattern `<_> "<_> <_> <_>" <_> <size>`
  | unwrap size [5m]
)
```

---

## 8. 메트릭 쿼리 (집계)

로그에서 메트릭을 만들어 대시보드에 활용합니다.

```logql
# 초당 에러 로그 건수 (Time series 패널)
sum(rate({namespace="default"} |= "ERROR" [5m])) by (app)

# 서비스별 5분 에러 건수
sum by (app) (count_over_time({namespace="default"} |= "ERROR" [5m]))

# HTTP 상태코드별 비율 (JSON 파싱)
sum by (status_code) (
  rate({app="my-app"} | json | status_code != "" [5m])
)

# 에러율 (에러/전체)
sum(rate({app="my-app"} | json | level = "error" [5m]))
/
sum(rate({app="my-app"} | json [5m]))

# 슬로우 쿼리 비율 (1초 초과)
sum(rate({app="my-app"} | json | duration_ms > 1000 [5m]))
/
sum(rate({app="my-app"} | json [5m]))
```

---

## 9. 실무 장애 분석 쿼리

### 에러 패턴 파악

```logql
# 에러 메시지별 건수 (어떤 에러가 많은지)
topk(10,
  sum by (message) (
    count_over_time({namespace="default"} | json | level = "error" [1h])
  )
)

# 파드별 에러 건수 (어느 파드가 문제인지)
sum by (pod) (count_over_time({namespace="default"} |= "ERROR" [5m]))

# 에러가 특정 사용자에게 집중되는지
{app="my-app"} | json | level = "error"
  | line_format `user={{.user_id}} error={{.error}}`
```

### 특정 요청 추적

```logql
# 특정 사용자의 모든 요청 (user_id로)
{app="my-app"} | json | user_id = "12345"

# 특정 요청 ID로 전체 흐름 추적
{namespace="default"} | json | request_id = "abc-123-xyz"

# 특정 IP에서 오는 요청
{job="nginx"} | pattern `<ip> <_>`| ip = "1.2.3.4"
```

### 슬로우 요청 분석

```logql
# 2초 이상 걸린 요청 (느린 순 정렬)
{app="my-app"} | json | duration_ms > 2000
  | line_format `{{.duration_ms}}ms {{.method}} {{.path}}`

# 슬로우 요청의 공통 경로 파악
topk(5,
  sum by (path) (
    count_over_time({app="my-app"} | json | duration_ms > 1000 [1h])
  )
)
```

### 배포 전후 비교

```logql
# 현재 에러율
sum(rate({app="my-app"} |= "ERROR" [5m]))

# 1시간 전 에러율 (배포 전)
sum(rate({app="my-app"} |= "ERROR" [5m] offset 1h))
```

### DB 관련 로그 분석

```logql
# DB 관련 에러 전체
{namespace="default"} |~ "(?i)(database|db|sql|query|postgres|mysql)"
  |= "error" OR |= "failed" OR |= "timeout"

# 슬로우 쿼리 로그 (PostgreSQL이 앱 로그에 기록하는 경우)
{app="my-app"} |= "slow query" | regexp `(?P<duration>\d+)ms`
  | duration > 500

# 커넥션 풀 경고
{app="my-app"} |~ "(?i)(connection pool|hikari|pool.*exhausted|too many connections)"
```

---

## 10. 로그 볼륨 분석

과도한 로그를 생성하는 앱을 파악합니다.

```logql
# 앱별 로그 생성 속도 (초당 bytes)
sum by (app) (bytes_rate({namespace="default"}[5m]))

# 앱별 로그 라인 수 (초당)
sum by (app) (rate({namespace="default"}[5m]))

# 지난 1시간 앱별 총 로그 크기
sum by (app) (bytes_over_time({namespace="default"}[1h]))

# 특정 시간 이후 로그가 갑자기 늘어난 앱
sum by (app) (rate({namespace="default"}[5m]))
vs
sum by (app) (rate({namespace="default"}[5m] offset 1h))
```

---

## 11. 알림용 LogQL 패턴

```logql
# 에러 로그 급증 (초당 1건 초과)
sum(rate({namespace="default"} |= "ERROR" [5m])) > 1

# 특정 에러 키워드 등장 (즉시 알림)
count_over_time({namespace="default"} |= "OOMKilled" [5m]) > 0
count_over_time({namespace="default"} |= "database is down" [5m]) > 0

# 로그가 5분간 전혀 없는 경우 (앱 다운)
sum(rate({namespace="default", app="my-app"}[5m])) == 0

# P95 응답 시간 2초 초과
quantile_over_time(0.95,
  {app="my-app"} | json | unwrap duration_ms [5m]
) > 2000
```

---

## 12. 유용한 팁

### 라인 수 제한 우회 (대량 로그 분석)
```
Explore → Options → Max lines: 5000으로 올리기
또는 메트릭 쿼리로 전환 (count_over_time 사용)
```

### 로그 레벨 정규화
```logql
# WARN, Warning, warning을 모두 잡기
{app="my-app"} |~ "(?i)warn"

# 또는 json 파싱 후 대소문자 무시
{app="my-app"} | json | level =~ "(?i)error|err"
```

### 여러 앱 동시에 보기
```logql
# namespace 내 모든 앱
{namespace="default"} |= "ERROR" | json
  | line_format `[{{.app}}] {{.message}}`

# 특정 앱 그룹
{namespace="default", app=~"api|worker|scheduler"} |= "ERROR"
```

---

## 참고

- [LogQL 공식 문서](https://grafana.com/docs/loki/latest/query/)
- [LogQL Metric Queries](https://grafana.com/docs/loki/latest/query/metric_queries/)
- [LogQL Template Functions](https://grafana.com/docs/loki/latest/query/template_functions/)
