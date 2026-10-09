# Chat Application — Kubernetes Project

A 3-tier chat application deployed on Kubernetes using Minikube (local).

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  Ingress (chat-ns.com)           │
└──────────────┬──────────────────────────────────┘
               │
     ┌─────────▼──────────┐
     │   Frontend: React  │  Port 80
     └─────────┬──────────┘
               │ /api/*
     ┌─────────▼──────────┐
     │  Backend: Node.js  │  Port 5001
     └─────────┬──────────┘
               │
     ┌─────────▼──────────┐
     │    DB: MongoDB      │  Port 27017  (PV + PVC)
     └────────────────────┘
```

| Tier | Technology | Port |
|------|-----------|------|
| Frontend | ReactJS (served via Nginx) | 80 |
| Backend | Node.js | 5001 |
| Database | MongoDB | 27017 |

All resources live in the `chat-application` namespace.

---

## Prerequisites

### Start Minikube

```bash
# If already installed
minikube start --driver=docker    # Docker must be installed and running

# Fresh install (macOS ARM)
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-darwin-arm64
sudo install minikube-darwin-arm64 /usr/local/bin/minikube
minikube start --driver=docker
```

---

## Step 1 — Build & Push Docker Images

Each microservice needs its own Docker image. A `Dockerfile` must exist in each service directory.

```bash
# Log in to Docker Hub
docker login
# Enter your username and the access token generated from Docker Hub settings

# Build and push backend
cd backend/
docker build -t <dockerhub-username>/chat-app-backend:latest .
docker push <dockerhub-username>/chat-app-backend:latest

# Build and push frontend
cd ../frontend/
docker build -t <dockerhub-username>/chat-app-frontend:latest .
docker push <dockerhub-username>/chat-app-frontend:latest

# Pull the official MongoDB image (no build needed)
docker pull mongo

# Verify images exist locally
docker images
```

> To re-tag an existing image: `docker image tag <old-tag> <new-tag>` then `docker image push <new-tag>`

---

## Step 2 — Create the Namespace

```yaml
# namespace.yml
apiVersion: v1
kind: Namespace
metadata:
  name: chat-application
```

```bash
kubectl apply -f namespace.yml
kubectl get ns    # Verify chat-application appears
```

---

## Step 3 — Create the Secret (Base64-encoded passwords)

Encode your MongoDB password before putting it in the secret:

```bash
echo -n "your-password-here" | base64
# Copy the output and paste it into secret.yml
```

```yaml
# secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
  namespace: chat-application
type: Opaque
data:
  mongo-root-password: <base64-encoded-password>
```

```bash
kubectl apply -f secret.yml
```

> **Gotcha:** Forgetting `-n` in `echo -n "..."` encodes a trailing newline — the decoded value will be wrong and MongoDB auth will fail silently.

---

## Step 4 — Deploy MongoDB (with Persistent Storage)

Apply in this order — PV must exist before PVC, PVC before Deployment:

```bash
# 1. Persistent Volume
kubectl apply -f mongodb-pv.yml
kubectl get pv -n chat-application

# 2. Persistent Volume Claim
kubectl apply -f mongodb-pvc.yml
kubectl get pvc -n chat-application     # Status should be "Bound"

# 3. MongoDB Deployment
kubectl apply -f mongodb-deployment.yml
kubectl get pods -n chat-application    # Wait for Running

# 4. MongoDB Service
kubectl apply -f mongodb-service.yml
```

**Key point in `mongodb-deployment.yml`:** The `MONGO_INITDB_ROOT_PASSWORD` env var must reference the secret:

```yaml
env:
  - name: MONGO_INITDB_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mongodb-secret
        key: mongo-root-password
  - name: MONGO_INITDB_ROOT_USERNAME
    value: admin
```

---

## Step 5 — Deploy Backend

```bash
kubectl apply -f backend-deployment.yml
kubectl apply -f backend-service.yml
```

The backend deployment must include the MongoDB connection URI as an env var:

```yaml
env:
  - name: MONGO_URI
    value: "mongodb://admin:$(MONGO_PASSWORD)@mongodb-service:27017/chatdb"
```

> **Gotcha:** The frontend uses Nginx as a reverse proxy with path `/api` pointing to the backend service. If the backend service isn't deployed before the frontend, the frontend pod fails to start because Nginx can't resolve the upstream.

---

## Step 6 — Deploy Frontend

```bash
kubectl apply -f frontend-deployment.yml
kubectl apply -f frontend-service.yml
```

**Frontend is on port 80, Backend is on port 5001.**

---

## Step 7 — Configure Ingress

Enable the ingress addon in Minikube first (this installs the nginx-ingress controller):

```bash
minikube addons enable ingress

# Verify the controller is running
kubectl get pods -n ingress-nginx
```

```bash
kubectl apply -f ingress.yml
kubectl get ing -n chat-application
```

Add an entry to your `/etc/hosts` to resolve the domain locally:

```bash
# Get the Minikube IP
minikube ip

# Add to /etc/hosts
echo "<minikube-ip>  chat-ns.com" | sudo tee -a /etc/hosts
```

Now visit `http://chat-ns.com` in your browser.

---

## Troubleshooting

### Check running port conflicts

```bash
lsof -i:80    # See which process is using port 80
```

### Verify all resources are running

```bash
kubectl get all -n chat-application
```

### Check pod logs

```bash
kubectl logs <pod-name> -n chat-application
kubectl logs <pod-name> -n chat-application --previous   # Logs from a crashed container
```

### Describe a failing pod

```bash
kubectl describe pod <pod-name> -n chat-application
# Look at the Events section at the bottom — most errors are explained there
```

### Common errors encountered

| Error | Cause | Fix |
|-------|-------|-----|
| Frontend CrashLoopBackOff | Backend service not deployed yet | Apply backend-service.yml first |
| MongoDB pod pending | PVC not bound to PV | Check namespace matches in pv/pvc YAMLs |
| Auth failed on MongoDB | Secret password not base64 encoded | Re-encode with `echo -n "password" \| base64` |
| Ingress not routing | Ingress addon not enabled | `minikube addons enable ingress` |

---

## Deployment Order Summary

```bash
kubectl apply -f namespace.yml
kubectl apply -f secret.yml
kubectl apply -f mongodb-pv.yml
kubectl apply -f mongodb-pvc.yml
kubectl apply -f mongodb-deployment.yml
kubectl apply -f mongodb-service.yml
kubectl apply -f backend-deployment.yml
kubectl apply -f backend-service.yml
kubectl apply -f frontend-deployment.yml
kubectl apply -f frontend-service.yml
kubectl apply -f ingress.yml
```

Or apply all at once (order not guaranteed):

```bash
kubectl apply -f k8s/
```
