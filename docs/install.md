# Grafana 설치 (Helm)

EKS 환경에서 `grafana/grafana` Helm Chart로 Grafana를 설치합니다.

---

## 사전 조건 확인

```bash
# kubectl 연결 확인
kubectl get nodes

# Helm 버전 확인 (>= 3.10 권장)
helm version
```

---

## 1. 네임스페이스 생성

```bash
kubectl create namespace monitoring
```

---

## 2. Helm 저장소 추가

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 사용 가능한 버전 확인
helm search repo grafana/grafana --versions | head -10
```

---

## 3. Admin 패스워드 Secret 생성

```bash
kubectl create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=<YOUR_PASSWORD> \
  --namespace monitoring
```

---

## 4. Helm values 파일 준비

```bash
cp ../ops/config/grafana/values.yaml my-values.yaml
```

`my-values.yaml`에서 반드시 수정할 항목:

```yaml
# Admin 자격증명 (Secret 참조)
admin:
  existingSecret: grafana-admin
  userKey: admin-user
  passwordKey: admin-password

# 데이터소스 URL (실제 서비스 주소로 변경)
datasources:
  datasources.yaml:
    datasources:
      - url: http://prometheus.monitoring.svc.cluster.local:9090   # Prometheus
      - url: http://loki.monitoring.svc.cluster.local:3100          # Loki
```

---

## 5. Grafana 설치

```bash
GRAFANA_CHART_VERSION="8.10.0"

helm install grafana grafana/grafana \
  --namespace monitoring \
  --values my-values.yaml \
  --version ${GRAFANA_CHART_VERSION} \
  --wait
```

---

## 6. 설치 확인

```bash
# 파드 상태 확인
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana

# 서비스 확인
kubectl get svc -n monitoring grafana
```

정상 설치 시:
```
NAME                      READY   STATUS    RESTARTS   AGE
grafana-xxxxxxxxxx-xxxxx  1/1     Running   0          1m
```

---

## 7. Grafana 접속 (포트 포워드)

```bash
kubectl port-forward svc/grafana -n monitoring 3000:80
```

브라우저에서 `http://localhost:3000` 접속
- 초기 계정: `admin` / `<YOUR_PASSWORD>`

---

## 8. 업그레이드

```bash
# 업그레이드 전 현재 values 확인
helm get values grafana -n monitoring

helm upgrade grafana grafana/grafana \
  --namespace monitoring \
  --values my-values.yaml \
  --version 8.11.0
```

---

## 9. 롤백

```bash
helm rollback grafana 1 -n monitoring
```

---

## 10. 삭제

```bash
helm uninstall grafana -n monitoring

# PVC 삭제 (데이터베이스 포함)
kubectl delete pvc -n monitoring -l app.kubernetes.io/name=grafana
```

> **주의**: PVC를 삭제하면 Grafana에서 직접 만든 대시보드, 알림, 사용자 설정이 모두 사라집니다.
> Provisioning으로 관리하는 대시보드는 영향 없습니다.

---

## 설치 시 자주 발생하는 문제

| 증상 | 원인 | 해결 방법 |
|------|------|----------|
| 파드 `CrashLoopBackOff` | DB 초기화 실패 | PVC StorageClass 확인 (`gp2` / `gp3`) |
| `admin-password` 인식 안됨 | Secret 키 이름 불일치 | `userKey`, `passwordKey` 값 확인 |
| 데이터소스 `Bad Gateway` | 서비스 이름 오타 | `kubectl get svc -n monitoring` 으로 실제 서비스 이름 확인 |
| 대시보드 프로비저닝 안됨 | ConfigMap 마운트 실패 | `kubectl describe pod grafana-xxx -n monitoring` 이벤트 확인 |


---

## systemd 설치

Kubernetes 없이 단일 VM이나 베어메탈에서 돌릴 때는 바이너리를 내려받아 systemd로 관리합니다.

1. 바이너리 또는 패키지를 설치합니다.
2. 설정 파일을 /etc/<component>/ 아래에 둡니다.
3. `systemctl enable --now <service>`로 등록합니다.
4. `journalctl -u <service> -f`로 로그를 확인합니다.

## Docker Compose 설치

로컬 개발이나 빠른 검증은 Docker Compose가 가장 간단합니다.

1. `compose.yaml`을 만들고 이미지, 포트, 볼륨을 정의합니다.
2. `docker compose up -d`로 올립니다.
3. `docker compose logs -f`와 `docker compose ps`로 상태를 확인합니다.
4. 개발용은 같은 설정을 유지하되, 운영용은 Helm 또는 systemd를 사용합니다.
