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

start: ## Start Minikube (no nginx addon)
	minikube start --driver=$(MINIKUBE_DRIVER) --cpus=$(MINIKUBE_CPUS) --memory=$(MINIKUBE_MEM)

tunnel: ## Run minikube tunnel (needs sudo). Keep this terminal open.
	@echo "Starting 'minikube tunnel' (needs sudo). Leave this window open; Ctrl+C to stop."
	@sudo -E minikube tunnel

stop: ## Stop Minikube
	minikube stop || true

clean: ## Delete Minikube
	minikube delete || true

istio-install: ## Install Istio demo profile & enable sidecar injection for $(NS)
	istioctl install -y --set profile=demo
	kubectl create namespace $(NS) --dry-run=client -o yaml | kubectl apply -f -
	kubectl label namespace $(NS) istio-injection=enabled --overwrite

prom-stack: ## Install kube-prometheus-stack with selectors disabled (scrape all monitors)
	helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
	helm repo update
	helm upgrade --install promstack prometheus-community/kube-prometheus-stack \
	  -n monitoring --create-namespace -f monitoring/prom-values.yaml

monitoring-apply: ## Apply PodMonitors + RBAC so Prom can scrape Envoy metrics
#	kubectl apply -f monitoring/prom-rbac.yaml
	kubectl apply -f monitoring/podmonitors.yaml

rollouts-install: ## Install Argo Rollouts controller (with dashboard)
	helm repo add argo https://argoproj.github.io/argo-helm
	helm repo update
	helm upgrade --install argo-rollouts argo/argo-rollouts -n argo-rollouts --create-namespace \
	  --set dashboard.enabled=true

app-install: ## Install sample app via Helm
	helm upgrade --install $(RELEASE) $(CHART) -n $(NS)

app-upgrade: ## Re-apply Helm release (e.g., after values change)
	helm upgrade $(RELEASE) $(CHART) -n $(NS)

get-gateway-ip: ## Print istio-ingressgateway EXTERNAL-IP (requires 'make tunnel')
	@IP=$$(kubectl -n istio-system get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null); \
	if [ -z "$$IP" ]; then echo "❌ No EXTERNAL-IP. Run 'make tunnel' in another terminal."; exit 2; fi; echo $$IP

url: ## Print URL through Istio gateway (requires tunnel)
	@IP=$$(make -s get-gateway-ip); echo "http://$$IP/"

open: ## Curl URL (requires tunnel)
	@IP=$$(make -s get-gateway-ip); echo "→ Curling http://$$IP/"; curl -sS http://$$IP/

istio-nodeport: ## Fallback: switch ingressgateway to NodePort (no tunnel needed)
	kubectl -n istio-system patch svc istio-ingressgateway -p '{"spec":{"type":"NodePort"}}' || true

open-nodeport: ## Open via NodePort (fallback)
	@URL=$$(minikube -n istio-system service istio-ingressgateway --url | head -n1); \
	echo "→ Curling $$URL"; curl -sS "$$URL/"

load: ## Generate load against gateway (requires tunnel or NodePort)
	@echo "Ctrl+C to stop"; \
	if IP=$$(make -s get-gateway-ip 2>/dev/null); then \
	  while true; do curl -s "http://$$IP/" >/dev/null; sleep 0.2; done; \
	else \
	  URL=$$(minikube -n istio-system service istio-ingressgateway --url | head -n1); \
	  while true; do curl -s "$$URL/" >/dev/null; sleep 0.2; done; \
	fi

rollouts-status: ## Show rollout status
	kubectl argo rollouts get rollout $(RELEASE) -n $(NS)

rollouts-dashboard: ## Open Argo Rollouts dashboard
	kubectl argo rollouts dashboard -n $(NS)

diag: ## Quick diagnostics
	@echo "=== minikube status ==="; minikube status || true
	@echo "=== istio-system svc/pods ==="; kubectl -n istio-system get svc istio-ingressgateway || true; kubectl -n istio-system get pods || true
	@echo "=== monitoring PodMonitors ==="; kubectl -n monitoring get podmonitors || true
	@echo "=== demo rollout/svc/pods ==="; kubectl -n $(NS) get rollout,svc,po -o wide || true

all: start istio-install prom-stack monitoring-apply rollouts-install app-install ## Full setup
