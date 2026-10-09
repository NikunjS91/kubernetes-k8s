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
| Use case | Local development | Production / real projects |

With EKS, you only manage the **worker nodes** — AWS handles the entire control plane.

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
2. Attach permissions: `AdministratorAccess` (or specific EKS permissions)
3. Click the user → Security credentials → Create access key
4. Copy the Access Key ID and Secret

```bash
# Test that the CLI is working
aws s3 ls    # Lists your S3 buckets — if it works, credentials are valid
```

### Step 3 — Install eksctl

`eksctl` is the official CLI tool for creating and managing EKS clusters.

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

> This creates only the control plane. No worker nodes yet — apps can't run until Step 6.

### Step 5 — Associate IAM OIDC Provider

This connects the cluster to AWS IAM so pods can assume IAM roles (needed for services like S3, RDS access from pods).

```bash
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster tws-cluster \
  --approve
```

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

> `eksctl` uses **CloudFormation** under the hood — you'll see a stack in CloudFormation console while it runs. Takes ~15-20 minutes.

```bash
# Verify nodes are ready
kubectl get nodes
```

---

## Part 2 — Monitoring & Observability

### The Three Pillars of Observability

| Pillar | Question it answers | Tools |
|--------|-------------------|-------|
| **Metrics** | What is happening? (CPU, RAM, network) | Prometheus, Node Exporter |
| **Logs** | Why is it happening? (errors, stack traces) | Loki, Promtail |
| **Traces** | How did we get here? (request path through services) | Jaeger, OpenTelemetry |

### How Metrics Flow in a K8s Cluster

```
Nodes ──► Node Exporter (port 9100) ──┐
                                       ├──► Prometheus ──► Grafana dashboards
K8s Components ──► kube-state-metrics ─┘       │
(etcd, scheduler, API server)              PromQL queries
```

**Node Exporter** — runs on every node, exposes hardware metrics (CPU, memory, disk, network) on `:9100/metrics`

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
```

#### Step 4 — Access Prometheus

```bash
kubectl port-forward svc/prometheus-stack-kube-prom-prometheus \
  9090:9090 -n monitoring --address=0.0.0.0
```

Visit `http://localhost:9090` → Go to **Status → Targets** to see all scrape jobs (Node Exporter, kube-state-metrics, API server, etc.).

> **Pre-configured scraping:** Because we used the Helm chart, all scrape configs are already set up. You don't need to manually write `prometheus.yml`.

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
2. Enter dashboard ID `15760` (Kubernetes cluster overview) or `1860` (Node Exporter Full)
3. Select **Prometheus** as the data source
4. Click **Import**

---

## Useful Commands

```bash
# EKS cluster management
eksctl get cluster
eksctl get nodegroup --cluster=tws-cluster
eksctl delete cluster --name=tws-cluster    # WARNING: destroys everything

# Monitoring
kubectl get pods -n monitoring
kubectl get svc -n monitoring

# Helm
helm list -n monitoring
helm upgrade prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
helm uninstall prometheus-stack -n monitoring
```
