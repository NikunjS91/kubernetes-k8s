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

## 7. Kubeadm Setup (Bare Metal / EC2)

Kubeadm lets you connect two real servers (EC2 instances) in a master-worker configuration.

- Launch **2 EC2 instances** and SSH into each in separate terminals.
- One acts as the **master (control plane)**, the other as the **worker**.
- Open port **6443** in the EC2 security group (used by the API server).

> Reference: https://github.com/LondheShubham153/kubestarter/blob/main/Kubeadm_Installation_Scripts_and_Documentation/README.md

### What is containerd?
`containerd` is a container runtime used under the hood by Docker and Kubernetes to manage containers.

---

### Run on BOTH Master and Worker Nodes

**1. Disable Swap** — required for Kubernetes to function correctly.
```bash
sudo swapoff -a
```

**2. Load Kernel Modules** — required for Kubernetes networking.
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

**3. Set Sysctl Parameters**
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
lsmod | grep br_netfilter
lsmod | grep overlay
```

**4. Install containerd**
```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y containerd.io

containerd config default | sed -e 's/SystemdCgroup = false/SystemdCgroup = true/' \
  -e 's/sandbox_image = "registry.k8s.io\/pause:3.6"/sandbox_image = "registry.k8s.io\/pause:3.9"/' \
  | sudo tee /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl status containerd
```

**5. Install Kubernetes Components**
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

### Run ONLY on Master Node

**1. Initialize the cluster**
```bash
sudo kubeadm init
```

**2. Set up local kubeconfig**
```bash
mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config
```

**3. Install Calico network plugin**
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```

**4. Generate join command** (copy output for worker nodes)
```bash
kubeadm token create --print-join-command
```

---

### Run on ALL Worker Nodes

```bash
# Pre-flight check
sudo kubeadm reset pre-flight checks

# Paste the join command from master, add sudo + --v=5
sudo <paste-join-command-here> --cri-socket "unix:///run/containerd/containerd.sock" --v=5
```

---

### Verify Cluster

```bash
# On master node
kubectl get nodes
```

---

## 8. Core Kubernetes Objects

Kubernetes is known for **self-healing** and **auto-scaling**.

The typical workflow:
> Container → Pod → Deployment → Service → User

The same pattern applies to different apps (e.g., nginx, MySQL) — just swap the image.

---

### Namespaces

A **namespace** is an isolated group (like a WhatsApp group) — resources inside one namespace don't affect another.

| Namespace | Purpose |
|---|---|
| `default` | Resources with no namespace assigned |
| `kube-node-lease` | Node heartbeat/lease information |
| `kube-public` | Publicly accessible resources |
| `kube-system` | System-level services |
| `local-path-storage` | Local storage pods |

```bash
kubectl get namespace          # list all namespaces
kubectl create ns nginx        # create a namespace
kubectl get pods -n nginx      # list pods in a namespace
```

---

### Pods

The smallest deployable unit in K8s — containers run inside pods.

```bash
kubectl run nginx --image=nginx              # create pod in default namespace
kubectl run nginx --image=nginx -n nginx     # create pod in specific namespace
kubectl delete pod nginx                     # delete a pod
kubectl exec -it nginx-pod -n nginx -- bash  # enter pod shell
```

**Pod YAML (`pod.yml`)**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f pod.yml
kubectl get pods -n nginx
```

---

### Deployments

| Object | Description |
|---|---|
| **ReplicaSet** | Ensures N copies of a pod are always running |
| **StatefulSet** | Like ReplicaSet but pods are numbered/ordered (stateful apps) |
| **Deployment** | Builds on ReplicaSet + adds **rolling updates** (zero downtime upgrades) |

**Labels & Selectors** — Labels tag a pod (e.g. `app: nginx`). Selectors tell the deployment which pods to manage/replicate.

**Deployment YAML (`deployment.yml`)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deployment.yml

# Scale pods up/down
kubectl scale deployment/nginx-deployment -n nginx --replicas=5
```
