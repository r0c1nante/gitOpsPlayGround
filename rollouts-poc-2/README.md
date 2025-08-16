# Argo Rollouts + Istio on Minikube (macOS) – PoC

## Quickstart
```bash
make all                 # start cluster, install Istio/Prometheus/Rollouts, deploy app
make url                 # prints http://<istio_gw_ip>/
make open                # curls the app via Istio gateway
make rollouts-status     # observe rollout
make rollouts-dashboard  # UI
```

### Trigger canary
```bash
helm upgrade rollouts-demo helm/rollouts-demo -n demo --set image.tag=green
```

### Optional: Grafana
```bash
helm upgrade --install promstack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace \
  --set grafana.enabled=true
kubectl -n monitoring port-forward svc/promstack-grafana 3000:80
open http://localhost:3000  # admin / prom-operator
```

### Fallback without tunnel
```bash
make istio-nodeport && make open-nodeport
```

See `helm/rollouts-demo/templates/analysistemplate.yaml` for the PromQL success rate check.
