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

### What `values.yaml` Looks Like

This is the only file you typically change. Everything else is wired to these values via `{{ .Values.* }}` in the templates.

```yaml
# values.yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

ingress:
  enabled: true
  host: myapp.example.com
```

For a **dev** environment you might override with fewer replicas and a debug image tag, for **prod** you use more replicas and a pinned release tag — all from the same chart.

### Preview Rendered Manifests (Without Installing)

Before applying, use `helm template` to see exactly what YAML Helm will generate. This is great for debugging or CI validation.

```bash
helm template dev-apache ./apache-helm -n dev-apache
# Prints all rendered YAML to stdout — no cluster interaction needed
```

### Lint a Chart

```bash
helm lint ./apache-helm
# Checks for syntax errors and common mistakes before you install
```

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

# Override specific values at install time (without editing values.yaml)
helm install dev-apache apache-helm -n dev-apache --set replicaCount=1 --set image.tag=latest

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

> **Why rollback matters:** If you upgrade and your app starts crashing, `helm rollback` restores the previous exact state — Deployment, ConfigMap, Service — in one command. Much safer than manually reverting YAML files.

### Useful Helm Commands

```bash
helm list -A                    # List all releases across all namespaces
helm repo list                  # List configured chart repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update                # Refresh repo index
helm uninstall dev-apache -n dev-apache   # Remove a release completely
helm get values prd-apache -n prd-apache  # Show values for a running release
helm status prd-apache -n prd-apache      # Show release status and notes
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

### Example 1 — Create a Directory Before App Starts (`initcontainer.yaml`)

This is the simplest case: the init container creates a folder that the main app needs.

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

### Example 2 — Wait for a Service to Be Ready (Real-World Pattern)

This is the most common production use case: your app needs the database to be up before it starts. Without this, the app crashes on startup and Kubernetes spends time restarting it.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-with-db-wait
spec:
  initContainers:
    - name: wait-for-mysql
      image: busybox
      # Keep trying to connect until MySQL responds — only then exit 0
      command:
        - sh
        - -c
        - |
          until nslookup mysql-service; do
            echo "Waiting for MySQL to be available..."
            sleep 2
          done
          echo "MySQL is ready!"

  containers:
    - name: myapp
      image: myapp:latest
      env:
        - name: DB_HOST
          value: mysql-service
```

> **Why this works:** `nslookup mysql-service` succeeds only when the MySQL Service exists and has a DNS entry. The init container loops until that happens, so `myapp` never starts until MySQL is reachable.

```bash
kubectl apply -f initcontainer.yaml

# Watch pod lifecycle — you'll see Init:0/1 → Init:1/1 → Running
kubectl get pods -w

# Check init container logs
kubectl logs init-demo -c init-setup

# Check main container logs
kubectl logs init-demo -c main-app
```

---

## 3. Sidecar Containers

### What Are They?

A **Sidecar Container** runs *alongside* the main container in the same pod, sharing the same network namespace and optionally the same volumes. It's a helper that augments the main app without modifying it.

Think of it as plugging in a log collector, a metrics exporter, or a proxy, without changing a single line of your application code.

### Common Sidecar Patterns

| Pattern | Example | Why Not in the Main App? |
|---------|---------|--------------------------|
| Log collection | Fluentd shipping logs to Elasticsearch | Log shipping logic clutters app code |
| Metrics export | Prometheus exporter alongside your app | Metrics format shouldn't be your app's concern |
| Proxy / mTLS | Envoy proxy (used by Istio service mesh) | Security layer belongs at infra level |
| Config reload | Watcher that hot-reloads config on change | Separation of app logic from config management |

### Example 1 — Log Shipping Sidecar (`sidecar-container.yml`)

The main app writes logs to a shared volume. The sidecar reads and ships them — the app never needs to know where logs go.

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

### Example 2 — Nginx with a Metrics Exporter Sidecar

A real-world pattern: Nginx doesn't natively expose Prometheus metrics, so a sidecar exporter scrapes Nginx's status page and converts it to Prometheus format.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-metrics
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
      # Nginx status page must be enabled for the exporter to scrape
      volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d

    - name: nginx-exporter          # Sidecar: converts nginx stats → Prometheus metrics
      image: nginx/nginx-prometheus-exporter:latest
      args:
        - -nginx.scrape-uri=http://localhost/nginx_status
      ports:
        - containerPort: 9113       # Prometheus scrapes this port

  volumes:
    - name: nginx-config
      configMap:
        name: nginx-status-config
```

> **Why localhost works:** Both containers share the same network namespace inside the pod. The sidecar can reach Nginx on `localhost:80` without any service or IP address.

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
- **Traffic management** — route 10% of traffic to v2, 90% to v1 (canary deployments)
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
# This label tells Istio to inject Envoy into every new pod in this namespace
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

# Verify — each pod should show 2/2 READY (app + envoy sidecar)
kubectl get pods

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

### Traffic Splitting with VirtualService (Canary Deployment)

This is one of Istio's most powerful features. You can send a percentage of traffic to a new version without changing any application code or redeploying.

```yaml
# virtualservice.yaml — Route 90% to v1, 10% to v2
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 90       # 90% of traffic goes to v1 (stable)
        - destination:
            host: reviews
            subset: v2
          weight: 10       # 10% goes to v2 (canary — testing new version)
---
# destinationrule.yaml — Define what v1 and v2 mean
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels:
        version: v1       # Pods with label version=v1
    - name: v2
      labels:
        version: v2       # Pods with label version=v2
```

```bash
kubectl apply -f virtualservice.yaml
kubectl apply -f destinationrule.yaml

# To gradually shift more traffic to v2 (promote the canary):
# Edit weight: v1=50, v2=50 → then v1=0, v2=100
kubectl apply -f virtualservice.yaml
```

> **Why this is better than a standard K8s rollout:** With K8s rolling updates, traffic split is determined by replica count (5 old pods + 1 new = ~17% new). With Istio you get exact percentages regardless of replica count, plus you can roll back instantly by changing a number.

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

In Kiali you'll see:
- A real-time service graph showing which service calls which
- Request rates and error rates per edge
- Response time percentiles
- The VirtualService traffic weights visualised

---

## Quick Reference

```bash
# Helm
helm create <chart>                                   # Scaffold a new chart
helm template <release> ./<chart>                     # Preview rendered YAML
helm lint ./<chart>                                   # Check for errors
helm install <release> <chart> -n <ns> --create-namespace
helm upgrade <release> ./<chart> -n <ns>
helm rollback <release> <revision> -n <ns>
helm history <release> -n <ns>
helm uninstall <release> -n <ns>

# Init / Sidecar containers
kubectl get pods -w                           # Watch pod lifecycle
kubectl logs <pod> -c <container-name>        # Logs from a specific container

# Istio
istioctl install -f <profile.yaml> -y
kubectl label namespace <ns> istio-injection=enabled
kubectl get pods                              # Check 2/2 READY = sidecar injected
istioctl dashboard kiali
kubectl get virtualservice
kubectl get destinationrule
kubectl get gateway
```
