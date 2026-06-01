# Grafana systemd 설치

단일 VM이나 베어메탈에서 `grafana-server`를 systemd 서비스로 관리할 때 사용한다.

## 대상

- `grafana-server.service`

## 준비물

- Grafana 설치 패키지 또는 바이너리
- 설정 파일: `/etc/grafana/grafana.ini`
- provisioning 디렉터리: `/etc/grafana/provisioning/`
- 데이터 디렉터리: `/var/lib/grafana`

## 절차

1. 패키지를 설치한다.
2. `grafana.ini`와 provisioning 파일을 배치한다.
3. `systemctl daemon-reload` 후 `enable --now grafana-server`를 실행한다.
4. `journalctl -u grafana-server -f`로 로그를 본다.

## 확인 명령

```bash
systemctl status grafana-server
curl http://localhost:3000/api/health
```

## 운영 포인트

- 데이터소스와 대시보드는 provisioning으로 관리한다.
- 관리자 비밀번호는 환경 변수나 별도 secret 파일로 주입한다.
- 플러그인 설치가 필요하면 패키지 레벨 또는 시작 스크립트에서 반영한다.

