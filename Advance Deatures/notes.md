# Kubernetes Advanced Features — Helm, Init Containers, Sidecars & Service Mesh

---

## 1. Helm — Kubernetes Package Manager

### The Problem

Every Kubernetes app requires multiple manifests: Deployment, Service, ConfigMap, Secret, Ingress, PVC... Writing and managing these individually across dev, staging, and prod environments is error-prone and repetitive.

**Helm** packages all these manifests into a single unit called a **Chart**, and lets you install, upgrade, and roll back the whole thing with one command — just like `apt-get` or `brew` for apps.

### Install Helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Create a Chart

```bash
helm create apache-helm
cd apache-helm
tree    # tree is a separate utility — install with: brew install tree
```

A new chart has this structure:

```
apache-helm/
├── Chart.yaml          # Chart metadata (name, version, description)
├── values.yaml         # Default config values — this is what you edit
├── charts/             # Sub-chart dependencies
└── templates/          # Kubernetes manifest templates (Deployment, Service, etc.)
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

> **How it works:** Templates use Go templating syntax (`{{ .Values.replicaCount }}`). Helm substitutes values from `values.yaml` at install time. You only edit `values.yaml` — not the templates.

### Package a Chart

```bash
# Package the chart into a .tgz archive (portable, importable/exportable)
helm package apache-helm
# Output: apache-helm-0.1.0.tgz
```

### Install a Chart

```bash
# Install with a release name into the default namespace
helm install dev-apache apache-helm

# Install into a specific namespace (creates it if it doesn't exist)
helm install dev-apache apache-helm -n dev-apache --create-namespace

# Install prod environment from the same chart
helm install prd-apache apache-helm -n prd-apache --create-namespace
```

> **Key concept:** One chart, multiple named releases. `dev-apache` and `prd-apache` are completely independent deployments from the same chart. Change `values.yaml` to point each at different image tags, replica counts, resource limits, etc.

### Upgrade and Rollback

```bash
# After editing values.yaml, upgrade the running release
helm upgrade prd-apache ./apache-helm -n prd-apache

# Roll back to revision 1 if something goes wrong
helm rollback prd-apache 1 -n prd-apache

# See revision history
helm history prd-apache -n prd-apache
```

### Useful Helm Commands

```bash
helm list -A                    # List all releases across all namespaces
helm repo list                  # List configured chart repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update                # Refresh repo index
helm uninstall dev-apache -n dev-apache   # Remove a release completely
helm get values prd-apache -n prd-apache  # Show values for a running release
```

---

## 2. Init Containers

### What Are They?

An **Init Container** runs to completion *before* the main application container starts. It's used to do prerequisite work: create directories, wait for a service to be ready, fetch secrets, seed config files, etc.

If the init container fails, Kubernetes retries it until it succeeds (or the pod fails). The main container never starts until all init containers have exited cleanly.

### Init vs Regular Containers

| Feature | Init Container | Main Container |
|---------|---------------|----------------|
| Run order | Before main | After all inits |
| Runs until | Completes and exits | Runs continuously |
| Purpose | Setup / preconditions | Application logic |
| Failure behavior | Pod retried | Pod marked failed |

### Example (`initcontainer.yaml`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
    - name: init-setup
      image: busybox
      command: ["sh", "-c", "mkdir -p /data/app && echo 'folder ready' > /data/app/status.txt"]
      volumeMounts:
        - name: shared-data
          mountPath: /data

  containers:
    - name: main-app
      image: nginx
      volumeMounts:
        - name: shared-data
          mountPath: /data   # Sees the folder created by init container
  
  volumes:
    - name: shared-data
      emptyDir: {}
```

```bash
kubectl apply -f initcontainer.yaml

# Watch pod lifecycle — you'll see Init:0/1 → Running
kubectl get pods -w

# Check init container logs
kubectl logs init-demo -c init-setup

# Check main container logs
kubectl logs init-demo -c main-app
```

> **Common use case:** Wait for a database to be ready before starting the app server. Use `init-container` to poll the DB connection and only exit 0 when it's available.

---

## 3. Sidecar Containers

### What Are They?

A **Sidecar Container** runs *alongside* the main container in the same pod, sharing the same network namespace and optionally the same volumes. It's a helper that augments the main app without modifying it.

Think of it as plugging in a log collector, a metrics exporter, or a proxy, without changing a single line of your application code.

### Common Sidecar Patterns

| Pattern | Example |
|---------|---------|
| Log collection | Fluentd shipping logs to Elasticsearch |
| Metrics export | Prometheus exporter alongside your app |
| Proxy / mTLS | Envoy proxy (used by Istio service mesh) |
| Config reload | Watcher that hot-reloads config on change |

### Example (`sidecar-container.yml`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-test
spec:
  containers:
    - name: main-container
      image: busybox
      command: ["sh", "-c", "while true; do echo \"$(date): main app log\" >> /var/log/app.log; sleep 5; done"]
      volumeMounts:
        - name: log-volume
          mountPath: /var/log

    - name: sidecar-container   # Runs in parallel with main-container
      image: busybox
      command: ["sh", "-c", "tail -f /var/log/app.log"]
      volumeMounts:
        - name: log-volume
          mountPath: /var/log   # Shares the same volume — reads what main writes

  volumes:
    - name: log-volume
      emptyDir: {}
```

```bash
kubectl apply -f sidecar-container.yml

# Check both containers are running (READY column shows 2/2)
kubectl get pods

# View logs from each container
kubectl logs sidecar-test -c main-container
kubectl logs sidecar-test -c sidecar-container
```

---

## 4. Service Mesh with Istio

### The Problem at Scale

With hundreds of microservices, you need:
- **Traffic management** — route 10% of traffic to v2, 90% to v1
- **Observability** — which service called which, with what latency
- **Security** — mutual TLS between every service automatically
- **Resilience** — retries, timeouts, circuit breakers

Doing this in application code for every service is unsustainable. **Istio** handles all of it at the infrastructure layer using sidecar proxies (Envoy) injected into every pod.

### Istio Architecture

| Component | Role |
|-----------|------|
| **Envoy** | Sidecar proxy injected into each pod. All traffic flows through it |
| **Istiod** | Control plane — configures all Envoy proxies |
| **Pilot** | Distributes routing rules to Envoy proxies |
| **Citadel** | Issues mTLS certificates between services |
| **Galley** | Validates and ingests Istio configuration |

### Setup

```bash
# Create a dedicated cluster for Istio testing
kind create cluster --name istio-testing

# Download Istio (check istio.io for latest version)
curl -L https://istio.io/downloadIstio | sh -
cd istio-*/bin

# Move the CLI binary to your PATH so it works anywhere
mv istioctl /usr/local/bin

# Install Istio on the cluster
istioctl install -f samples/bookinfo/demo-profile-no-gateways.yaml -y

# Enable automatic sidecar injection for the default namespace
kubectl label namespace default istio-injection=enabled

# Verify the label was applied
kubectl get namespace -L istio-injection

# If it already had a conflicting label:
kubectl label namespace default istio-injection=enabled --overwrite
```

### Install Gateway API CRDs (required for Istio gateways)

```bash
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.6.0" | kubectl apply -f -; }
```

> These CRDs are not built into Kubernetes — they extend it to support Istio's gateway model.

### Deploy the Sample Bookinfo App

```bash
# Deploy — Istio auto-injects Envoy sidecar into each pod
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# Verify the app works inside the cluster
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" \
  -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"

# Expose via gateway
kubectl apply -f samples/bookinfo/gateway-api/bookinfo-gateway.yaml

# Use ClusterIP (no cloud LB needed for local)
kubectl annotate gateway bookinfo-gateway networking.istio.io/service-type=ClusterIP --namespace=default

# Check gateway was created
kubectl get gateway

# Port forward to access in browser
kubectl port-forward svc/bookinfo-gateway-istio 8080:80
```

### Kiali Dashboard — Visualise the Service Mesh

Kiali is Istio's observability UI — it shows a live graph of services, traffic flow, error rates, and latencies.

```bash
# Install Kiali and addons (Prometheus, Grafana, Jaeger)
kubectl apply -f samples/addons/kiali.yaml

# Wait for Kiali to be ready
kubectl rollout status deployment/kiali -n istio-system

# Open the Kiali dashboard
istioctl dashboard kiali
```

---

## Quick Reference

```bash
# Helm
helm install <release> <chart> -n <ns> --create-namespace
helm upgrade <release> ./<chart> -n <ns>
helm rollback <release> <revision> -n <ns>
helm uninstall <release> -n <ns>

# Init / Sidecar containers
kubectl get pods -w                           # Watch pod lifecycle
kubectl logs <pod> -c <container-name>        # Logs from a specific container

# Istio
istioctl install -f <profile.yaml> -y
kubectl label namespace <ns> istio-injection=enabled
istioctl dashboard kiali
kubectl get gateway
```
