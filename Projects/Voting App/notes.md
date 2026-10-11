# Voting App — Kubernetes Project with Observability

A multi-tier microservices voting application deployed on Kubernetes, extended with full monitoring using **Prometheus** and **Grafana**.

## Architecture

```
Browser
   │
   ├──► Vote UI (Python/Flask, port 5000)   ──► Redis (message queue)
   │                                                    │
   └──► Result UI (Node.js, port 4000)                  │
              ▲                                   Worker (.NET)
              │                                         │
         PostgreSQL  ◄────────────────────────────────┘
         (persistent storage)
```

| Service | Language | Role | Port |
|---------|----------|------|------|
| `vote` | Python (Flask) | Web UI for casting votes | 5000 |
| `result` | Node.js | Web UI showing live results | 4000 |
| `worker` | .NET Core | Reads Redis queue → writes to Postgres | internal |
| `redis` | Redis | Message queue between vote and worker | 6379 |
| `db` | PostgreSQL | Persistent storage for vote counts | 5432 |

**Why this architecture?** The vote and result services are decoupled through Redis and the worker. This means vote submission is fast (just a Redis write), and the result page updates asynchronously. In a high-traffic scenario, you can scale the vote service independently without touching the result side.

---

## Prerequisites

```bash
# Start a Kind cluster (or use an existing one)
kind create cluster --name voting-app

# Verify kubectl is pointed at it
kubectl get nodes
kubectl cluster-info
```

---

## The Kubernetes Manifests

### PostgreSQL Deployment (`db-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: db
          image: postgres:15-alpine
          env:
            - name: POSTGRES_USER
              value: "postgres"
            - name: POSTGRES_PASSWORD
              value: "postgres"
            - name: POSTGRES_DB
              value: "votes"
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: db-storage
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: db-storage
          emptyDir: {}    # For learning — use PVC for persistence in production
```

### Redis Deployment (`redis-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:alpine
          ports:
            - containerPort: 6379
```

### Vote Service (`vote-service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vote
spec:
  type: NodePort
  selector:
    app: vote
  ports:
    - port: 5000
      targetPort: 5000
      nodePort: 31000    # Access vote UI at <node-ip>:31000
```

### Result Service (`result-service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: result
spec:
  type: NodePort
  selector:
    app: result
  ports:
    - port: 4000
      targetPort: 4000
      nodePort: 31001    # Access result UI at <node-ip>:31001
```

> **Why NodePort?** Kind doesn't have a cloud load balancer. NodePort exposes services on static ports on the node, so you can open them directly in the browser. In production on EKS/GKE you'd use `type: LoadBalancer` instead.

---

## Deploying the Application

Apply in order — data layer first, then workers, then frontends:

```bash
# 1. Data layer — worker and frontends both depend on these
kubectl apply -f k8s/db-deployment.yaml
kubectl apply -f k8s/db-service.yaml
kubectl apply -f k8s/redis-deployment.yaml
kubectl apply -f k8s/redis-service.yaml

# 2. Worker — reads from Redis, writes to Postgres — depends on both above
kubectl apply -f k8s/worker-deployment.yaml

# 3. Frontend services — vote writes to Redis, result reads from Postgres
kubectl apply -f k8s/vote-deployment.yaml
kubectl apply -f k8s/vote-service.yaml
kubectl apply -f k8s/result-deployment.yaml
kubectl apply -f k8s/result-service.yaml

# 4. Verify everything is running
kubectl get all
```

**Access the UI:**

```bash
# On Kind, get the node IP
kubectl get nodes -o wide   # Note the INTERNAL-IP

# Or use port-forward
kubectl port-forward svc/vote 5000:5000
kubectl port-forward svc/result 4000:4000
```

---

## Part 2 — Observability

The voting app is an excellent monitoring candidate — multiple services, a queue, and a database all interact. You can measure queue depth, vote throughput, and worker processing lag.

### The Three Pillars

| Pillar | Question | Tools | What to measure for voting app |
|--------|---------|-------|-------------------------------|
| **Metrics** | What is happening? | Prometheus | Vote rate, Redis queue depth, worker lag |
| **Logs** | Why is it happening? | Loki + Promtail | Worker errors, failed DB writes |
| **Traces** | How did we get here? | Jaeger + OpenTelemetry | Time from vote submission to result update |

### How Data Flows

```
Each Node
  └── Node Exporter (:9100) ──────────────────────┐
                                                    ▼
K8s Core Components                           Prometheus
  └── kube-state-metrics ──────────────────► (scrapes every 15s)
                                                    │
Application Pods                                    │
  └── /metrics endpoint ──────────────────────────►│
                                                    ▼
                                              Grafana Dashboards
                                              (PromQL queries)
```

---

### Setup Monitoring

#### Create Namespace and Add Helm Repo

```bash
kubectl create namespace monitoring

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm repo list    # Confirm it's there
```

#### Install kube-prometheus-stack

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
# Check all pods come up (takes ~2 minutes)
kubectl --namespace monitoring get pods -l "release=prometheus-stack"
```

#### Access Prometheus

```bash
kubectl port-forward svc/prometheus-stack-kube-prom-prometheus \
  9090:9090 -n monitoring --address=0.0.0.0
```

Open `http://localhost:9090` → **Status → Targets** to see what's being scraped.

**PromQL queries specific to the voting app:**

```promql
# Are all voting app pods running?
kube_pod_status_phase{namespace="default", pod=~"vote.*|result.*|worker.*|redis.*|db.*"}

# Any pod restart (crash) in the last hour?
increase(kube_pod_container_status_restarts_total{namespace="default"}[1h]) > 0

# CPU usage for voting app pods
sum(rate(container_cpu_usage_seconds_total{namespace="default", pod=~"vote.*|worker.*"}[5m])) by (pod)

# Memory usage for all voting app services
container_memory_working_set_bytes{namespace="default", container!=""}

# Is Redis consuming a lot of memory? (could indicate queue backup)
container_memory_working_set_bytes{namespace="default", pod=~"redis.*"}

# Worker pod CPU — if it spikes, votes are being processed (expected under load)
rate(container_cpu_usage_seconds_total{namespace="default", pod=~"worker.*"}[1m])
```

#### Access Grafana

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

#### Import a Dashboard

1. Grafana → **Dashboards → Import**
2. Use dashboard ID **15760** (Kubernetes cluster overview) or **1860** (Node Exporter Full)
3. Select **Prometheus** as the data source → **Import**

You'll immediately see CPU, memory, pod counts, and network graphs for the entire cluster including the voting app services.

---

## Scaling Considerations

Since vote and result are stateless, you can scale them independently based on load:

```bash
# Scale vote service to 5 replicas during peak voting time
kubectl scale deployment vote --replicas=5

# Scale back down after
kubectl scale deployment vote --replicas=1

# Or use HPA (Horizontal Pod Autoscaler) to do it automatically
kubectl autoscale deployment vote --cpu-percent=60 --min=1 --max=10
```

> **Don't scale the worker too aggressively.** Multiple worker instances processing the same Redis queue can cause duplicate vote counts if not handled with proper locking. Check the worker code for idempotency before scaling it.

> **Never scale PostgreSQL with a Deployment.** Use a StatefulSet for PostgreSQL in production — multiple Deployment replicas with shared storage lead to data corruption. For this learning project, keep `replicas: 1`.

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

# Check if worker is connecting to both Redis and Postgres
kubectl logs -l app=worker | grep -E "connected|error|exception"

# Monitoring
kubectl get pods -n monitoring
kubectl get svc -n monitoring

# Helm management
helm list -n monitoring
helm upgrade prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
```
