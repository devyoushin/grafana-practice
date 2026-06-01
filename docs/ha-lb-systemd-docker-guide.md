# Grafana 일반 서버 HA와 LB 구성

컨테이너 오케스트레이션 없이 VM이나 베어메탈에서 Grafana를 운영할 때의 이중화와 LB 구성 방식이다. Docker Compose와 systemd 모두 프로세스 실행 방식만 다를 뿐, HA 설계 원칙은 같다.

## 기본 구조

```text
사용자
  |
LB
  |
+-------------+-------------+
|                           |
Grafana-1                  Grafana-2
|                           |
+-------------+-------------+
              |
     PostgreSQL / MySQL
              |
        datasource
              |
   Thanos Query / Mimir / Prometheus
```

## Grafana HA 특성

Grafana는 여러 인스턴스를 같은 외부 DB에 연결해서 이중화한다.

```text
LB -> Grafana-1
   -> Grafana-2

Grafana-1,2 -> PostgreSQL/MySQL
```

Grafana를 2대 이상 운영할 때 SQLite를 사용하면 안 된다. 대시보드, 사용자, 조직, 알림, datasource 설정을 여러 인스턴스가 공유해야 하므로 PostgreSQL이나 MySQL 같은 외부 DB를 사용한다.

## Grafana 공통 설정

모든 Grafana 인스턴스는 다음 설정을 동일하게 맞춘다.

```text
grafana.ini
provisioning datasource
provisioning dashboard
plugin 버전
secret_key
root_url/domain 설정
```

DB 설정 예시:

```ini
[database]
type = postgres
host = grafana-db.example.com:5432
name = grafana
user = grafana
password = $__file{/etc/grafana/db_password}
ssl_mode = require
```

`secret_key`가 인스턴스마다 다르면 암호화된 datasource password나 session 관련 동작에서 문제가 생길 수 있으므로 같은 값을 사용한다.

## Grafana 앞단 LB

Grafana 앞단 LB는 round-robin 구성이 가능하다.

```haproxy
frontend grafana_front
    bind *:80
    mode http
    default_backend grafana_back

backend grafana_back
    mode http
    balance roundrobin
    option httpchk GET /api/health
    server grafana1 10.0.1.11:3000 check
    server grafana2 10.0.1.12:3000 check
```

OAuth, SSO, reverse proxy 인증을 사용한다면 `root_url`, `domain`, `serve_from_sub_path`, `X-Forwarded-*` 헤더 처리를 함께 확인한다.

## Datasource HA

Grafana 자체는 LB 뒤에서 이중화할 수 있지만, datasource도 별도로 이중화해야 한다.

Prometheus를 datasource로 직접 연결하는 경우:

```text
Grafana
  |
Prometheus LB(active/passive 권장)
  |
+-------------+-------------+
|                           |
Prometheus-1                Prometheus-2
```

Prometheus는 로컬 TSDB를 사용하므로 두 인스턴스의 쿼리 결과가 완전히 동일하다고 보장할 수 없다. 그래서 Prometheus 앞단은 round-robin보다 active/passive가 안전하다.

운영 권장 구성:

```text
Grafana
  |
Thanos Query / Mimir
  |
+-------------+-------------+
|                           |
Prometheus-1                Prometheus-2
```

Grafana datasource는 Prometheus 개별 인스턴스보다 Thanos Query나 Mimir를 바라보는 편이 안정적이다.

## Docker로 배포할 때

Docker Compose를 사용해도 HA는 서버 단위로 나누어야 한다. 한 서버 안에서 Grafana 컨테이너를 여러 개 띄우는 것은 서버 장애를 막지 못한다.

```text
Server-1:
  grafana

Server-2:
  grafana

Server-3:
  postgresql/mysql
  haproxy
```

각 Grafana 컨테이너는 같은 `grafana.ini`, provisioning 파일, plugin 버전을 사용한다. 데이터는 컨테이너 로컬 볼륨이 아니라 외부 DB에 저장한다.

## systemd로 배포할 때

systemd는 각 서버에서 `grafana-server` 프로세스를 관리한다. HA 자체를 제공하지는 않는다.

```text
grafana-server.service
haproxy.service
postgresql.service 또는 외부 DB
```

역할 분리는 다음처럼 가져간다.

```text
systemd: 서비스 시작, 재시작, 로그 관리
LB: health check와 트래픽 분산
외부 DB: Grafana 상태 공유
provisioning: datasource/dashboard 설정 관리
```

## 전체 권장 구성

작은 규모:

```text
LB
  |
+-------------+-------------+
|                           |
Grafana-1                  Grafana-2
|                           |
+-------------+-------------+
              |
       PostgreSQL/MySQL

Grafana datasource
  |
Prometheus LB(active/passive)
  |
+-------------+-------------+
|                           |
Prometheus-1                Prometheus-2
```

운영 권장:

```text
LB
  |
+-------------+-------------+
|                           |
Grafana-1                  Grafana-2
|                           |
+-------------+-------------+
              |
       PostgreSQL/MySQL
              |
       Thanos Query / Mimir
              |
+-------------+-------------+
|                           |
Prometheus-1                Prometheus-2
```

## 핵심 요약

```text
Grafana HA
= 여러 Grafana 인스턴스 + 공유 DB + LB

Grafana DB
= 2대 이상이면 SQLite 금지, PostgreSQL/MySQL 사용

Grafana LB
= round-robin 가능, health check는 /api/health

Datasource HA
= Prometheus 직접 연결보다 Thanos Query/Mimir 권장

Docker/systemd
= 실행 방식 차이일 뿐, HA는 서버 분산과 LB health check로 구성
```
