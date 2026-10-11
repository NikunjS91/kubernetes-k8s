# Mega Project — EKS on AWS with Monitoring & Observability

A production-grade Kubernetes deployment on **Amazon EKS** (Elastic Kubernetes Service), with full observability using **Prometheus** and **Grafana**.

---

## Part 1 — EKS Cluster Setup

### Why EKS Over Kind/Minikube?

| Feature | Kind / Minikube | EKS |
|---------|----------------|-----|
| Control plane | You manage it | AWS manages it (etcd, scheduler, API server) |
| Cost | Free (local) | Paid (EC2 node instances) |
| Scalability | Limited to laptop | Full cloud autoscaling |
| High availability | Single node | Multi-AZ worker nodes |
| Use case | Local development | Production / real projects |

With EKS, you only manage the **worker nodes** — AWS handles the entire control plane. If `etcd` crashes at 3am, AWS fixes it, not you.

---

### Step 1 — Install AWS CLI

```bash
# Install unzip first (if on Ubuntu/EC2)
sudo apt-get install unzip

# Install AWS CLI v2
curl -fsSL https://awscli.amazonaws.com/v2/install.sh | bash

# Verify installation
aws --version
```

### Step 2 — Configure AWS Credentials

```bash
aws configure
# Prompts for:
#   AWS Access Key ID:     <from IAM>
#   AWS Secret Access Key: <from IAM>
#   Default region:        us-east-1
#   Output format:         json
```

**How to get credentials:**
1. Go to AWS Console → IAM → Users → Create user (`eks-user`)
2. Attach permissions: `AdministratorAccess` (or specific EKS permissions for least privilege)
3. Click the user → Security credentials → Create access key → Select "CLI"
4. Copy the Access Key ID and Secret

```bash
# Test that the CLI is working
aws s3 ls    # Lists your S3 buckets — if it works, credentials are valid
aws sts get-caller-identity   # Shows which IAM user/role is active
```

### Step 3 — Install eksctl

`eksctl` is the official CLI tool for creating and managing EKS clusters. It wraps CloudFormation so you don't have to write complex infra configs.

```bash
# macOS
brew install aws/tap/eksctl

# Verify
eksctl version
```

### Step 4 — Create the EKS Cluster (Control Plane Only)

```bash
eksctl create cluster \
  --name=tws-cluster \
  --region=us-east-1 \
  --version=latest \
  --without-nodegroup
```

> This creates only the control plane. No worker nodes yet — apps can't schedule until Step 6.

**What eksctl does behind the scenes:**
- Creates a CloudFormation stack
- Provisions the EKS control plane (managed by AWS)
- Creates VPC, subnets, and security groups
- Configures `~/.kube/config` so `kubectl` points to the new cluster

### Step 5 — Associate IAM OIDC Provider

This connects the cluster's identity system to AWS IAM, enabling **IRSA** (IAM Roles for Service Accounts). Without this, pods can't assume IAM roles to access AWS services like S3, RDS, or Secrets Manager.

```bash
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster tws-cluster \
  --approve

# Verify
aws iam list-open-id-connect-providers
```

> **Example use case:** Your Node.js app needs to read from an S3 bucket. Instead of hardcoding AWS credentials inside the pod, you create an IAM role with S3 read permissions, attach it to a ServiceAccount via IRSA, and the pod automatically gets credentials — no secrets needed.

### Step 6 — Create a Node Group (Worker Nodes)

```bash
eksctl create nodegroup \
  --cluster=tws-cluster \
  --region=us-east-1 \
  --name=tws-cluster-ng \
  --node-type=t2.medium \
  --nodes=2 \
  --nodes-min=2 \
  --nodes-max=2 \
  --node-volume-size=20 \
  --ssh-access \
  --ssh-public-key=eks-nodegroup-key
```

**Node type guide:**

| Instance | vCPU | RAM | Use for |
|----------|------|-----|---------|
| `t3.small` | 2 | 2GB | Dev/learning only |
| `t2.medium` | 2 | 4GB | Small workloads |
| `t3.large` | 2 | 8GB | Production apps |
| `m5.xlarge` | 4 | 16GB | High-traffic services |

> `eksctl` uses **CloudFormation** under the hood — you'll see a stack in CloudFormation console while it runs. Takes ~15-20 minutes.

```bash
# Verify nodes are ready
kubectl get nodes
kubectl get nodes -o wide   # Shows node IPs and instance types
```

---

## Part 2 — Monitoring & Observability

### The Three Pillars of Observability

| Pillar | Question it answers | Tools |
|--------|-------------------|-------|
| **Metrics** | What is happening? (CPU, RAM, network, error rate) | Prometheus, Node Exporter, kube-state-metrics |
| **Logs** | Why is it happening? (error messages, stack traces) | Loki, Promtail, CloudWatch Logs |
| **Traces** | How did we get here? (request path through services) | Jaeger, OpenTelemetry, AWS X-Ray |

### How Metrics Flow in a K8s Cluster

```
Each Node
  └── Node Exporter (DaemonSet, port 9100)
        Exports: CPU%, memory, disk I/O, network traffic per node

K8s API Server
  └── kube-state-metrics (Deployment)
        Exports: pod restarts, deployment replicas ready/desired,
                 resource requests vs limits, HPA status

Both ──► Prometheus ──► Grafana Dashboards
         (scrapes every 15s)   (visualises with PromQL)
```

**Node Exporter** — runs as a DaemonSet (one pod per node), exposes hardware metrics on `:9100/metrics`

**kube-state-metrics** — monitors the K8s API objects themselves (pod restarts, deployment status, resource requests vs limits)

**Prometheus** — time-series database that scrapes both exporters, stores metrics, and serves PromQL queries

**Grafana** — connects to Prometheus as a data source and renders dashboards

---

### Setup Monitoring with Helm

#### Step 1 — Create the Monitoring Namespace

```bash
kubectl create namespace monitoring
```

#### Step 2 — Add the Prometheus Community Helm Repo

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Verify the repo was added
helm repo list
```

#### Step 3 — Install the kube-prometheus-stack

This single Helm chart installs Prometheus, Grafana, Node Exporter, kube-state-metrics, and AlertManager all at once.

```bash
helm install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.service.type=NodePort \
  --set prometheus.service.nodePort=30000 \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=31000
```

```bash
# Verify everything is running
kubectl --namespace monitoring get pods -l "release=prometheus-stack"

# You should see pods for:
# - prometheus-stack-kube-prom-prometheus    (Prometheus server)
# - prometheus-stack-grafana                 (Grafana)
# - prometheus-stack-kube-state-metrics      (K8s metrics)
# - prometheus-stack-prometheus-node-exporter (one per node)
# - alertmanager                             (alerting)
```

#### Step 4 — Access Prometheus

```bash
kubectl port-forward svc/prometheus-stack-kube-prom-prometheus \
  9090:9090 -n monitoring --address=0.0.0.0
```

Visit `http://localhost:9090` → Go to **Status → Targets** to see all scrape jobs (Node Exporter, kube-state-metrics, API server, etc.).

> **Pre-configured scraping:** Because we used the Helm chart, all scrape configs are already set up. You don't need to manually write `prometheus.yml`.

**Useful PromQL Queries to Try:**

```promql
# CPU usage per node (as a percentage)
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage per pod
container_memory_working_set_bytes{namespace="default", container!=""}

# Pod restart count (detect crash loops)
kube_pod_container_status_restarts_total{namespace="default"} > 0

# Pods not in Running state (detect problems across cluster)
kube_pod_status_phase{phase!="Running", phase!="Succeeded"} == 1

# CPU request vs actual usage (detect over/under provisioned pods)
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod)
  /
sum(kube_pod_container_resource_requests{resource="cpu", namespace="default"}) by (pod)

# HTTP error rate (if your app exports request metrics)
rate(http_requests_total{status=~"5.."}[5m])
  /
rate(http_requests_total[5m])
```

#### Step 5 — Access Grafana

```bash
kubectl port-forward svc/prometheus-stack-grafana \
  3000:80 -n monitoring --address=0.0.0.0
```

Visit `http://localhost:3000`.

**Get the admin password:**

```bash
# Method 1 — from the Helm install output
kubectl --namespace monitoring get secrets prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d; echo

# Method 2
kubectl get secret --namespace monitoring \
  -l app.kubernetes.io/component=admin-secret \
  -o jsonpath="{.items[0].data.admin-password}" | base64 --decode; echo
```

Default username: `admin`

#### Step 6 — Import a Dashboard

1. In Grafana, go to **Dashboards → Import**
2. Enter a dashboard ID and click **Load**
3. Select **Prometheus** as the data source
4. Click **Import**

**Recommended Dashboard IDs:**

| ID | Name | What it shows |
|----|------|--------------|
| `15760` | Kubernetes cluster overview | CPU, memory, pod counts by namespace |
| `1860` | Node Exporter Full | Per-node hardware metrics |
| `13332` | Kubernetes Deployments | Deployment health and replica status |
| `17320` | Kubernetes Pods | Per-pod CPU, memory, restarts |

#### Step 7 — AlertManager (Get Notified When Things Break)

The kube-prometheus-stack already installs AlertManager. Add alert rules to get notified when pods crash or CPU spikes:

```yaml
# alert-rules.yml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cluster-alerts
  namespace: monitoring
  labels:
    release: prometheus-stack
spec:
  groups:
    - name: pod-alerts
      rules:
        - alert: PodCrashLooping
          expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"
            description: "Pod has restarted more than once in the last 15 minutes"

        - alert: HighCPUUsage
          expr: |
            100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
          for: 10m
          labels:
            severity: critical
          annotations:
            summary: "Node {{ $labels.instance }} CPU above 85%"
```

```bash
kubectl apply -f alert-rules.yml
# Check AlertManager: http://localhost:9090/alerts (in Prometheus UI)
```

---

## Cost Management Tips

- **Stop the cluster when not using it:** You're billed for EC2 node instances even when idle
- **`t2.medium` for learning:** Cheaper than larger instances, enough for demo workloads
- **Delete node group before the cluster:** Nodes are the expensive part

```bash
# Scale down node group to 0 to save costs (cluster stays, nodes stop)
eksctl scale nodegroup --cluster=tws-cluster --name=tws-cluster-ng --nodes=0

# Full cleanup (destroys everything)
eksctl delete nodegroup --cluster=tws-cluster --name=tws-cluster-ng
eksctl delete cluster --name=tws-cluster    # WARNING: destroys everything
```

---

## Useful Commands

```bash
# EKS cluster management
eksctl get cluster
eksctl get nodegroup --cluster=tws-cluster
kubectl get nodes -o wide                           # Node IPs and instance types

# Monitoring
kubectl get pods -n monitoring
kubectl get svc -n monitoring

# Helm
helm list -n monitoring
helm upgrade prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
helm uninstall prometheus-stack -n monitoring
```
