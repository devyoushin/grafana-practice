# Grafana 설치 (Helm)

EKS 환경에서 `grafana/grafana` Helm Chart로 Grafana를 설치한다.

## 사전 준비

```bash
kubectl get nodes
helm version
kubectl create namespace monitoring
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

## Admin Secret

```bash
kubectl create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=<YOUR_PASSWORD> \
  --namespace monitoring
```

## values 준비

```bash
cp ../../ops/config/grafana/values.yaml my-values.yaml
```

## 설치

```bash
helm install grafana grafana/grafana \
  --namespace monitoring \
  --values my-values.yaml \
  --version 8.10.0 \
  --wait
```

## 확인

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana
kubectl get svc -n monitoring grafana
kubectl port-forward svc/grafana -n monitoring 3000:80
```

## 업그레이드 / 롤백 / 삭제

```bash
helm upgrade grafana grafana/grafana \
  --namespace monitoring \
  --values my-values.yaml \
  --version 8.11.0
helm rollback grafana 1 -n monitoring
helm uninstall grafana -n monitoring
```

```bash
kubectl delete pvc -n monitoring -l app.kubernetes.io/name=grafana
```

> PVC를 삭제하면 직접 만든 대시보드, 알림, 사용자 설정이 사라진다.

