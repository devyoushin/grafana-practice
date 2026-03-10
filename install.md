### 1. 리포지토리 추가 및 업데이트
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### 2. Grafana 설치 (Namespace: monitoring)
```bash
# 'grafana'라는 이름으로 설치
helm install my-grafana grafana/grafana --create-namespace --namespace monitoring```
```

### 3. 초기 비밀번호 확인
```bash
kubectl get secret --namespace monitoring my-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
