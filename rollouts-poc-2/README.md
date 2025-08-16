# Argo Rollouts + Istio on Minikube (macOS) – Robust PoC

## Quickstart
```bash
make all
# in a second terminal:
make tunnel
# back to first:
make url
make open
make rollouts-status
make rollouts-dashboard
make load
# trigger canary:
helm upgrade rollouts-demo helm/rollouts-demo -n demo --set image.tag=green
```
Fallback without tunnel:
```bash
make istio-nodeport && make open-nodeport
```
Diagnostics:
```bash
make diag
```
