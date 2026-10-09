# Voting App — Kubernetes Project with Observability

A multi-tier microservices voting application deployed on Kubernetes, extended with full monitoring using **Prometheus** and **Grafana**.

## Architecture

```
┌──────────┐     ┌──────────┐
│   Vote   │     │  Result  │   ← Frontend services (NodePort)
│ (Python) │     │ (Node.js)│
└────┬─────┘     └────┬─────┘
     │ writes          │ reads
┌────▼─────┐     ┌────▼─────┐
│  Redis   │     │ Postgres │   ← Data layer
└────┬─────┘     └──────────┘
     │
┌────▼─────┐
│  Worker  │                    ← Processes votes from Redis → Postgres
│  (.NET)  │
└──────────┘
```

| Service | Language | Role |
|---------|----------|------|
| `vote` | Python | Web UI for casting votes |
| `result` | Node.js | Web UI showing live results |
| `worker` | .NET Core | Consumes Redis queue, writes to Postgres |
| `redis` | Redis | Message queue between vote and worker |
| `db` | PostgreSQL | Persistent storage for results |

---

## Prerequisites

```bash
# Start a Kind cluster (or use an existing one)
kind create cluster --name voting-app

# Verify kubectl is pointed at it
kubectl get nodes
```

---

## Deploying the Application

Apply all manifests in order:

```bash
# Database layer first (worker depends on both)
kubectl apply -f k8s/db-deployment.yaml
kubectl apply -f k8s/db-service.yaml
kubectl apply -f k8s/redis-deployment.yaml
kubectl apply -f k8s/redis-service.yaml

# Processing layer
kubectl apply -f k8s/worker-deployment.yaml

# Frontend services
kubectl apply -f k8s/vote-deployment.yaml
kubectl apply -f k8s/vote-service.yaml
kubectl apply -f k8s/result-deployment.yaml
kubectl apply -f k8s/result-service.yaml

# Verify everything is running
kubectl get all
```

---

## Part 2 — Observability

The voting app is a good candidate for monitoring — it has multiple services, a queue, and a database. This section adds full cluster observability.

### The Three Pillars

| Pillar | Question | Tools |
|--------|---------|-------|
| **Metrics** | What is happening? | Prometheus + Node Exporter + kube-state-metrics |
| **Logs** | Why is it happening? | Loki + Promtail |
| **Traces** | How did we get here? | Jaeger + OpenTelemetry |

### How Data Flows

```
Each Node
  └── Node Exporter (:9100) ──────────────────────┐
                                                    ▼
K8s Core Components                           Prometheus
  └── kube-state-metrics ──────────────────► (time-series DB)
                                                    │
Application Pods ─────────────────────────────────►│
  └── (custom /metrics endpoint)                    │
                                                    ▼
                                              Grafana Dashboards
                                              (PromQL queries)
```

**Node Exporter** scrapes hardware-level metrics from every node (CPU, RAM, disk, network) and exposes them on port `9100`.

**kube-state-metrics** watches the Kubernetes API and exports cluster-state metrics (pod restarts, deployment health, resource requests vs. limits).

---

### Setup Monitoring Namespace

```bash
kubectl create namespace monitoring
```

### Add Prometheus Helm Repo

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm repo list    # Confirm it's there
```

### Install kube-prometheus-stack

One chart installs Prometheus, Grafana, Node Exporter, kube-state-metrics, and AlertManager:

```bash
helm install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.service.type=NodePort \
  --set prometheus.service.nodePort=30000 \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=31000
```

```bash
# Check all pods come up
kubectl --namespace monitoring get pods -l "release=prometheus-stack"
```

### Access Prometheus

```bash
kubectl port-forward svc/prometheus-stack-kube-prom-prometheus \
  9090:9090 -n monitoring --address=0.0.0.0
```

Open `http://localhost:9090` → **Status → Targets** to see what's being scraped.

Try a PromQL query to confirm data is flowing:

```promql
# CPU usage per pod
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)

# Memory usage
container_memory_working_set_bytes{namespace="default"}
```

### Access Grafana

```bash
kubectl port-forward svc/prometheus-stack-grafana \
  3000:80 -n monitoring --address=0.0.0.0
```

Open `http://localhost:3000`. Username: `admin`.

**Get the password:**

```bash
kubectl get secret prometheus-stack-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

### Import a Dashboard

1. Grafana → **Dashboards → Import**
2. Use dashboard ID **15760** (Kubernetes cluster overview) or **1860** (Node Exporter Full)
3. Select **Prometheus** as the data source → **Import**

You'll immediately see CPU, memory, pod counts, and network graphs for the entire cluster including the voting app services.

---

## Useful Commands

```bash
# Check all voting app resources
kubectl get all

# Watch pods in real time
watch kubectl get pods

# Check logs for a specific service
kubectl logs -l app=worker --tail=50
kubectl logs -l app=vote --tail=50

# Exec into a pod for debugging
kubectl exec -it <pod-name> -- sh

# Monitoring
kubectl get pods -n monitoring
kubectl get svc -n monitoring

# Helm management
helm list -n monitoring
helm upgrade prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
```
