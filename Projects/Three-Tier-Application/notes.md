# Three-Tier Application — AWS EKS Deployment

A full-stack 3-tier application deployed on **Amazon EKS**, with Docker images stored in **ECR** (Elastic Container Registry).

---

## Architecture

```
Browser
   │
   ▼
Ingress (nginx-ingress-controller)
   ├── /frontend  ──►  React App (Frontend)         ← Application Layer
   ├── /backend   ──►  Node.js API (Backend)         ← Logic Layer
   │                        │
   │                        ▼
   └──────────────   MongoDB (Database)              ← Data Layer
```

| Tier | Technology | Layer | Container |
|------|-----------|-------|-----------|
| Frontend | React | Application Layer | Custom Docker image → ECR |
| Backend | Node.js | Logic Layer | Custom Docker image → ECR |
| Database | MongoDB | Data Layer | Official `mongo` image |

**Why 3 tiers?** Each layer has a single responsibility and can be scaled, updated, or replaced independently. The frontend never talks directly to the database — it always goes through the backend API.

---

## Part 1 — Setup EC2 & Build Docker Images

### Step 1 — Launch EC2 and Pull Source Code

```bash
# SSH into your EC2 instance
ssh -i <your-key.pem> ubuntu@<ec2-public-ip>

# Pull the application source code
git clone <your-repo-url>
cd <project-folder>
```

### Step 2 — Install Docker on EC2

```bash
sudo apt-get update
sudo apt-get install -y docker.io

# Add your user to the docker group so you don't need sudo for every docker command
sudo usermod -aG docker $USER

# Apply group change without logging out
newgrp docker

# Verify Docker works
docker ps
```

### Step 3 — Write Dockerfiles

Each tier needs its own `Dockerfile`. A typical Node.js backend Dockerfile:

```dockerfile
# backend/Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --only=production

COPY . .

EXPOSE 5000
CMD ["node", "server.js"]
```

**Dockerfile steps explained:**

| Instruction | Example | Purpose |
|-------------|---------|---------|
| `FROM` | `node:18-alpine` | Base image — alpine keeps image size small |
| `WORKDIR` | `/app` | All subsequent commands run from this directory |
| `COPY package*.json ./` | — | Copy dependency manifest first (layer cache optimisation) |
| `RUN npm install` | — | Install dependencies as a separate layer — only re-runs when package.json changes |
| `COPY . .` | — | Copy actual source code last |
| `EXPOSE` | `5000` | Documents which port the container listens on |
| `CMD` | `["node", "server.js"]` | Default command when container starts |

### Step 4 — Build & Test Docker Images Locally

```bash
# Build backend image
docker build -t three-tier-backend:latest ./backend

# Build frontend image
docker build -t three-tier-frontend:latest ./frontend

# Verify images were created
docker images

# Quick test — run locally to verify the image works
docker run -p 5000:5000 three-tier-backend:latest

# Open EC2 Security Group → Inbound Rules → add port 5000
# Then visit: http://<ec2-ip>:5000

# Kill the test container when done
docker kill <container-id>
```

---

## Part 2 — Push Images to AWS ECR

**Why ECR over Docker Hub for AWS projects?** ECR is private by default, lives in your AWS account, integrates with IAM for access control, and EKS pulls from ECR without needing to create image pull secrets.

### Step 1 — Install AWS CLI

```bash
sudo apt-get install -y unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version
```

### Step 2 — Create IAM User & Configure CLI

```bash
aws configure
# AWS Access Key ID:     <from IAM user>
# AWS Secret Access Key: <from IAM user>
# Default region:        us-east-1
# Output format:         json
```

**How to get credentials:**
1. AWS Console → IAM → Users → Create user (e.g. `ecr-user`)
2. Attach policy: `AdministratorAccess` (or `AmazonEC2ContainerRegistryFullAccess` for least privilege)
3. Security credentials → Create access key → CLI

### Step 3 — Create ECR Repositories

One repository per service — each holds all versions (tags) of that image.

```bash
# Create a repo for backend
aws ecr create-repository --repository-name three-tier-backend --region us-east-1

# Create a repo for frontend
aws ecr create-repository --repository-name three-tier-frontend --region us-east-1
```

**ECR repository concepts:**

| Concept | Description |
|---------|-------------|
| **Repository** | Stores all versions of one Docker image |
| **Image tag** | Version label on an image (e.g. `latest`, `v1.2.0`) |
| **Registry URI** | `<account-id>.dkr.ecr.<region>.amazonaws.com` |
| **Repository URI** | `<registry-uri>/<repo-name>` |

### Step 4 — Authenticate Docker with ECR

```bash
# Get the login token and pipe it to docker login
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  <account-id>.dkr.ecr.us-east-1.amazonaws.com
# Output: Login Succeeded
```

### Step 5 — Tag and Push Images

```bash
# Set your ECR base URL (replace with your actual account ID)
ECR_URI=<account-id>.dkr.ecr.us-east-1.amazonaws.com

# Backend — tag and push
docker tag three-tier-backend:latest $ECR_URI/three-tier-backend:latest
docker push $ECR_URI/three-tier-backend:latest

# Frontend — tag and push
docker tag three-tier-frontend:latest $ECR_URI/three-tier-frontend:latest
docker push $ECR_URI/three-tier-frontend:latest
```

**Docker image lifecycle:**

| Step | Command | What happens |
|------|---------|-------------|
| Build | `docker build -t <name> .` | Creates image from Dockerfile |
| Tag | `docker tag <source> <target>` | Adds a new name/tag to existing image |
| Push | `docker push <target>` | Uploads image to registry |
| Pull | `docker pull <target>` | Downloads image from registry to local |

---

## Part 3 — Setup EKS Cluster

### Step 1 — Install kubectl and eksctl

```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/

# Verify both
kubectl version --client
eksctl version
```

### Step 2 — Create EKS Cluster

```bash
eksctl create cluster \
  --name three-tier-cluster \
  --region us-east-1 \
  --version latest \
  --nodegroup-name three-tier-ng \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 4 \
  --without-nodegroup

# Takes ~15-20 minutes. eksctl updates ~/.kube/config automatically.
```

### Step 3 — Bind kubectl to the EKS Cluster

```bash
# Update kubeconfig to point kubectl at the new cluster
aws eks update-kubeconfig --region us-east-1 --name three-tier-cluster

# Verify connection
kubectl get nodes    # Should show EC2 worker nodes in Ready state
kubectl cluster-info
```

**Why this step?** `kubectl` uses `~/.kube/config` to know which cluster to talk to. `update-kubeconfig` adds the EKS cluster's credentials and sets it as the active context. Without this, `kubectl` commands would still point at your local Kind/Minikube cluster.

---

## Part 4 — Deploy to Kubernetes

### Deployment order matters:

```
1. Secret (MongoDB credentials)
2. MongoDB Deployment + Service
3. Backend Deployment + Service
4. Frontend Deployment + Service
5. Ingress
```

### Step 1 — Create MongoDB Secret

```bash
# Encode credentials
echo -n "admin" | base64        # YWRtaW4=
echo -n "password123" | base64  # cGFzc3dvcmQxMjM=
```

```yaml
# secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
  namespace: three-tier
type: Opaque
data:
  mongo-user: YWRtaW4=
  mongo-password: cGFzc3dvcmQxMjM=
```

### Step 2 — MongoDB Deployment & Service

```yaml
# mongodb-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
  namespace: three-tier
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:6.0
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: mongo-user
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: mongo-password
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  namespace: three-tier
spec:
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
  type: ClusterIP    # Internal only — only the backend should talk to MongoDB
```

### Step 3 — Backend Deployment

```yaml
# backend-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: three-tier
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: <account-id>.dkr.ecr.us-east-1.amazonaws.com/three-tier-backend:latest
          ports:
            - containerPort: 5000
          env:
            - name: MONGO_URI
              value: "mongodb://$(MONGO_USER):$(MONGO_PASS)@mongodb-service:27017/appdb"
            - name: MONGO_USER
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: mongo-user
            - name: MONGO_PASS
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: mongo-password
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: three-tier
spec:
  selector:
    app: backend
  ports:
    - port: 5000
      targetPort: 5000
  type: ClusterIP
```

### Step 4 — Frontend Deployment

```yaml
# frontend-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: three-tier
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: <account-id>.dkr.ecr.us-east-1.amazonaws.com/three-tier-frontend:latest
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: three-tier
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
```

### Step 5 — Ingress

```yaml
# ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: three-tier-ingress
  namespace: three-tier
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /frontend
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
          - path: /backend
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 5000
```

### Apply Everything

```bash
kubectl create namespace three-tier

kubectl apply -f secret.yml
kubectl apply -f mongodb-deployment.yml
kubectl apply -f backend-deployment.yml
kubectl apply -f frontend-deployment.yml
kubectl apply -f ingress.yml

# Verify all pods are Running
kubectl get all -n three-tier
```

---

## Part 5 — ALB Ingress Controller (AWS Load Balancer)

For EKS, use the **AWS Load Balancer Controller** instead of nginx — it provisions an actual AWS Application Load Balancer (ALB) per Ingress resource.

### Step 1 — Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### Step 2 — Download & Create IAM Policy

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

### Step 3 — Associate OIDC Provider with EKS

```bash
# Required so EKS pods can assume IAM roles
eksctl utils associate-iam-oidc-provider \
  --cluster three-tier-cluster \
  --approve
```

### Step 4 — Create IAM Service Account

```bash
eksctl create iamserviceaccount \
  --cluster three-tier-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

| Concept | Purpose |
|---------|---------|
| OIDC Provider | Allows EKS pods to authenticate with AWS IAM |
| IAM Service Account | Links a K8s ServiceAccount to an IAM Role |
| IAM Policy | Grants the controller permission to manage ALBs, target groups, etc. |

### Step 5 — Deploy the Controller via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=three-tier-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Verify the controller is running
kubectl get deployment -n kube-system aws-load-balancer-controller
```

### Step 6 — Update Ingress to Use ALB

```yaml
# ingress.yml (ALB version)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: three-tier-ingress
  namespace: three-tier
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 5000
```

```bash
kubectl apply -f ingress.yml

# ALB takes ~2 min to provision — watch for the ADDRESS to appear
kubectl get ingress -n three-tier --watch
```

---

## Troubleshooting

| Problem | Command | What to look for |
|---------|---------|-----------------|
| Pod stuck Pending | `kubectl describe pod <name> -n three-tier` | Image pull error, resource limits, no nodes |
| Image pull error | `kubectl logs <pod> -n three-tier` | ECR login expired or wrong image URI |
| Backend can't reach MongoDB | `kubectl exec -it <backend-pod> -- curl mongodb-service:27017` | Service name mismatch |
| Ingress not working | `kubectl get ing -n three-tier` | ADDRESS empty = controller not installed |

```bash
# Useful debug commands
kubectl get all -n three-tier
kubectl describe pod <pod-name> -n three-tier
kubectl logs <pod-name> -n three-tier
kubectl exec -it <pod-name> -n three-tier -- sh
```
