# Grafana 업그레이드 가이드

Grafana 업그레이드는 Helm chart, Grafana 애플리케이션 버전, plugin, datasource/provisioning 설정에 영향을 줍니다. 운영 환경에서는 DB/PVC 백업과 plugin 호환성 확인이 먼저 필요합니다.

## 1. 사전 점검

```bash
export NAMESPACE="monitoring"
export RELEASE="grafana"
export CHART_VERSION="8.11.0"
export VALUES_FILE="my-values.yaml"

helm status ${RELEASE} -n ${NAMESPACE}
helm history ${RELEASE} -n ${NAMESPACE}
helm get values ${RELEASE} -n ${NAMESPACE} > values-before-upgrade.yaml
kubectl get pods,svc,pvc -n ${NAMESPACE} -l app.kubernetes.io/name=grafana
```

대시보드, datasource, alerting을 외부 DB나 provisioning으로 관리하지 않는 경우 PVC snapshot 또는 Grafana backup을 준비합니다.

## 2. Helm 업그레이드

```bash
helm repo update grafana
helm upgrade ${RELEASE} grafana/grafana \
  --namespace ${NAMESPACE} \
  --values ${VALUES_FILE} \
  --version ${CHART_VERSION} \
  --timeout 10m \
  --wait
```

## 3. 확인

```bash
kubectl rollout status deployment/${RELEASE} -n ${NAMESPACE}
kubectl get pods -n ${NAMESPACE} -l app.kubernetes.io/name=grafana
kubectl logs -n ${NAMESPACE} deployment/${RELEASE} --tail=100
kubectl port-forward svc/${RELEASE} -n ${NAMESPACE} 3000:80
```

로그인, datasource health check, dashboard 로딩, alert rule 평가 상태를 확인합니다.

## 4. 롤백

```bash
helm history ${RELEASE} -n ${NAMESPACE}
helm rollback ${RELEASE} <REVISION> -n ${NAMESPACE} --wait
```

Grafana DB migration이 수행된 뒤에는 애플리케이션 rollback만으로 DB schema가 되돌아가지 않을 수 있습니다. major 버전 변경은 백업 복구 절차를 준비한 뒤 진행합니다.

## 5. systemd / Docker Compose

systemd 설치는 패키지 또는 tarball을 교체한 뒤 `grafana-server`를 재시작합니다. Docker Compose 설치는 image tag를 변경하고 `docker compose pull && docker compose up -d`를 실행합니다. 두 방식 모두 `/var/lib/grafana`, `grafana.ini`, provisioning 디렉터리를 먼저 백업합니다.

