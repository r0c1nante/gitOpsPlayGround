SHELL := /bin/bash
NS ?= demo
CHART := helm/rollouts-demo
RELEASE := rollouts-demo
MINIKUBE_DRIVER ?= docker
MINIKUBE_CPUS ?= 4
MINIKUBE_MEM ?= 8192

.PHONY: help
help: ## Show targets
	@grep -E '^[a-zA-Z_-]+:.*?##' Makefile | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-25s\033[0m %s\n", $$1, $$2}'

start: ## Start Minikube (no nginx addon, no auto-tunnel)
	minikube start --driver=$(MINIKUBE_DRIVER) --cpus=$(MINIKUBE_CPUS) --memory=$(MINIKUBE_MEM)

tunnel: ## Run minikube tunnel in the foreground (requires sudo). Keep this running.
	@echo "Starting 'minikube tunnel' (needs sudo). Leave this terminal open. Ctrl+C to stop."
	@sudo -E minikube tunnel
diag: ## Quick diagnostics
	@echo "=== minikube status ==="; minikube status || true
	@echo "=== istio-system svc/pods ==="; kubectl -n istio-system get svc istio-ingressgateway || true; kubectl -n istio-system get pods || true
	@echo "=== demo rollout/svc/pods ==="; kubectl -n $(NS) get rollout,svc,po -o wide || true

open-nodeport: ## Open via minikube service (fallback)
	@URL=$$(minikube -n istio-system service istio-ingressgateway --url | head -n1); \
	echo "→ Curling $$URL"; curl -sS $$URL/

stop: ## Stop Minikube and tunnel
	minikube stop
#	@if [ -f .tunnel.pid ]; then kill $$(cat .tunnel.pid) || true; rm .tunnel.pid; fi

clean: ## Delete Minikube cluster
	minikube delete
istio-install: ## Install Istio demo profile & label namespace for sidecar injection
	istioctl install -y --set profile=demo
	kubectl create namespace $(NS) --dry-run=client -o yaml | kubectl apply -f -
	kubectl label namespace $(NS) istio-injection=enabled --overwrite

prom-stack: ## Install Prometheus (kube-prometheus-stack); Grafana disabled by default
	helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
	helm repo update
	helm upgrade --install promstack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace \
		--set grafana.enabled=true  \
		 --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
         --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false

rollouts-install: ## Install Argo Rollouts controller (via Helm)
	helm repo add argo https://argoproj.github.io/argo-helm
	helm repo update
	helm upgrade --install argo-rollouts argo/argo-rollouts -n argo-rollouts --create-namespace \
		--set dashboard.enabled=true

app-install: ## Install sample app via Helm
	helm upgrade --install $(RELEASE) $(CHART) -n $(NS)

app-upgrade: ## Upgrade image tag to trigger canary (see values.yaml)
	helm upgrade $(RELEASE) $(CHART) -n $(NS)

get-gateway-ip: ## Print istio-ingressgateway EXTERNAL-IP or fail fast with hint
	@IP=$$(kubectl -n istio-system get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null); \
	if [ -z "$$IP" ]; then \
	  echo "❌ No EXTERNAL-IP. Run 'make tunnel' in another terminal and wait 5–10s."; \
	  exit 2; \
	fi; echo $$IP

url: ## Print URL (requires tunnel)
	@IP=$$(make -s get-gateway-ip); echo "http://$$IP/"


open: ## Curl URL (requires tunnel)
	@IP=$$(make -s get-gateway-ip); echo "→ Curling http://$$IP/"; curl -sS http://$$IP/

# --- Optional fallback if tunnel is not desired ---
# Switch Istio ingressgateway to NodePort and use minikube service to resolve URL.
# Usage: make istio-nodeport && make open-nodeport
istio-nodeport: ## Patch Istio ingressgateway to NodePort (fallback; no tunnel needed)
	kubectl -n istio-system patch svc istio-ingressgateway -p '{"spec":{"type":"NodePort"}}'

open-nodeport: ## Open URL via minikube service (fallback)
	minikube -n istio-system service istio-ingressgateway --url

rollouts-status: ## Show rollout status
	kubectl argo rollouts get rollout $(RELEASE) -n $(NS)

rollouts-dashboard: ## Open Argo Rollouts dashboard
	kubectl argo rollouts dashboard -n $(NS)

endpoints: ## Show Gateway/VS endpoints
	kubectl -n $(NS) get gateway,virtualservice,svc

all: start istio-install prom-stack rollouts-install app-install endpoints ## Full setup (no /etc/hosts, no nginx)
