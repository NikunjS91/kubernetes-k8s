# Kubernetes (K8s) — Core Concepts & Setup

---

## 1. What is Kubernetes?

- Kubernetes (K8s) is a **container orchestration tool**.
- Maintained under **CNCF (Cloud Native Computing Foundation)**, which supports open-source cloud-native projects by providing resources.

---

## 2. Monolithic vs Microservices

| | Monolithic | Microservices |
|---|---|---|
| Failure impact | One service fails → all fail | Failure is isolated to one service |
| Cost | Higher | Lower |
| Deployment | Single deployable unit | Each service deployed independently |

- **Docker** can be used to containerize each microservice.
- **Kubernetes** manages and orchestrates those containers at scale.

---

## 3. Kubernetes Architecture

### Key Terms
- **Node** — A server (physical or virtual machine)
- **Cluster** — A group of multiple nodes working together

---

### Master Node (Control Plane)
The master node acts as the **headquarters** — it sends work to worker nodes and manages the cluster state.

| Component | Role |
|---|---|
| **API Server** | Gateway for all communication within the cluster. No direct component-to-component communication; everything goes through the API server. |
| **Scheduler** | Assigns containers/pods to worker nodes. A pod is the smallest K8s unit — containers run inside pods. |
| **etcd** | Key-value database that stores all cluster state and scheduler task data. |
| **Controller Manager** | Monitors and ensures everything is running correctly — nodes, pods, clusters, and services. |

---

### Worker Node
Worker nodes **run the actual workloads** (Docker containers inside pods).

| Component | Role |
|---|---|
| **kubelet** | Agent on each worker node. Checks if pods are running and reports status back to the API server, which updates the scheduler, etcd, and controller manager. |
| **kube-proxy (Service Proxy)** | Pods are isolated by default. The service proxy enables external access to pods by communicating with the API server → kubelet → pod. |

---

### Networking
- All nodes (master + worker) communicate over a **CNI (Container Network Interface)** network.
- Examples: **Weave Net**, **Calico**, **Flannel**

---

### kubectl
- CLI tool that talks to the **API server**.
- Used to manage and monitor the entire K8s cluster.

---

## 4. Ways to Create a Cluster

| Method | Description |
|---|---|
| **kubeadm** | Provision 2+ EC2 instances, install kubeadm, and join them into a cluster. |
| **Minikube** | Lightweight single-node cluster for local development or EC2. |
| **KIND (Kubernetes in Docker)** | Runs a full K8s cluster inside Docker containers. Great for local testing. |
| **EKS / AKS / GKE** | Managed K8s by AWS / Azure / Google Cloud — provider handles the control plane. |

---

## 5. Setup on AWS EC2

### Connect to EC2
```bash
# Connect using your .pem key
ssh -i your-key.pem ec2-user@<ec2-public-ip>
```

---

### Install Docker
```bash
sudo apt-get install docker.io

# Fix permission error for docker commands
sudo usermod -aG docker $USER && newgrp docker
```

---

### Install KIND
```bash
if ! command -v kind &>/dev/null; then
  echo "Installing Kind..."

  ARCH=$(uname -m)
  if [ "$ARCH" = "x86_64" ]; then
    curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.29.0/kind-linux-amd64
  elif [ "$ARCH" = "aarch64" ]; then
    curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.29.0/kind-linux-arm64
  else
    echo "Unsupported architecture: $ARCH"
    exit 1
  fi

  chmod +x ./kind
  sudo mv ./kind /usr/local/bin/kind
  echo "Kind installed successfully."
else
  echo "Kind is already installed."
fi
```

---

### Install kubectl
```bash
if ! command -v kubectl &>/dev/null; then
  echo "Installing kubectl..."

  ARCH=$(uname -m)
  VERSION=$(curl -Ls https://dl.k8s.io/release/stable.txt)

  if [ "$ARCH" = "x86_64" ]; then
    curl -Lo ./kubectl "https://dl.k8s.io/release/${VERSION}/bin/linux/amd64/kubectl"
  elif [ "$ARCH" = "aarch64" ] || [ "$ARCH" = "arm64" ]; then
    curl -Lo ./kubectl "https://dl.k8s.io/release/${VERSION}/bin/linux/arm64/kubectl"
  else
    echo "Unsupported architecture: $ARCH"
    exit 1
  fi

  chmod +x ./kubectl
  sudo mv ./kubectl /usr/local/bin/kubectl
  echo "kubectl installed successfully."
else
  echo "kubectl is already installed."
fi
```

---

### Verify Installations
```bash
kubectl version
docker --version
kind --version
```

---

## 6. Configuration — KIND Cluster

### YAML Basics
- YAML uses **key: value** pairs.
- Lists use `-` as a prefix and are written vertically.

### Create the Cluster Config

```bash
mkdir kind-cluster && cd kind-cluster
```

Create `config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: tws-cluster

nodes:
  - role: control-plane
    image: kindest/node:v1.35.0
    extraPortMappings:
      - containerPort: 30080
        hostPort: 8080
        protocol: TCP

  - role: worker
    image: kindest/node:v1.35.0

  - role: worker
    image: kindest/node:v1.35.0
```

### Create the Cluster
```bash
kind create cluster --name tws-cluster --config=config.yaml
```

---

## 7. Minikube (Coming Soon)
