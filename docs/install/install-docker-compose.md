# Grafana Docker Compose 설치

로컬 개발이나 provisioning 검증용으로 Grafana를 `docker compose`로 띄울 때 사용한다.

## 대상

- `grafana`

## 준비물

- `compose.yaml`
- `grafana.ini`
- `provisioning/` 디렉터리
- 데이터 저장용 volume

## 절차

1. `compose.yaml`에 이미지, 포트, volume을 정의한다.
2. provisioning 디렉터리를 bind mount 한다.
3. `docker compose up -d`로 올린다.
4. `docker compose logs -f grafana`와 `docker compose ps`를 확인한다.

## 확인 명령

```bash
docker compose ps
curl http://localhost:3000/api/health
```

## 운영 포인트

- 개발용은 로컬 계정과 단일 볼륨으로 충분하다.
- 운영 환경은 Helm 또는 systemd 문서를 따른다.
- 대시보드 provisioning 변경 시 컨테이너 재시작 없이 reload 가능한지 먼저 확인한다.
