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
               │ /api/*  (Nginx reverse proxy)
     ┌─────────▼──────────┐
     │  Backend: Node.js  │  Port 5001
     └─────────┬──────────┘
               │ mongodb://mongodb-service:27017
     ┌─────────▼──────────┐
     │    DB: MongoDB      │  Port 27017  (PV + PVC)
     └────────────────────┘
```

| Tier | Technology | Port | K8s Resource |
|------|-----------|------|-------------|
| Frontend | ReactJS served by Nginx | 80 | Deployment + Service + Ingress |
| Backend | Node.js | 5001 | Deployment + Service |
| Database | MongoDB | 27017 | Deployment + Service + PV + PVC + Secret |

All resources live in the `chat-application` namespace.

**Why namespaces?** Namespacing isolates all chat app resources — if you delete the namespace, everything goes with it cleanly. It also prevents name collisions with other apps on the same cluster.

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

# Verify cluster is running
kubectl get nodes    # Should show minikube node in Ready state
```

---

## Step 1 — Build & Push Docker Images

Each microservice needs its own Docker image. A `Dockerfile` must exist in each service directory.

**Why Docker Hub?** Minikube pulls images from a registry. If you just build locally, Minikube's internal Docker daemon can't see your local images. Push to Docker Hub so Minikube can pull them.

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

**Example Node.js Dockerfile (`backend/Dockerfile`):**

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production      # Install only production deps
COPY . .
EXPOSE 5001
CMD ["node", "server.js"]
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

# Example:
echo -n "ChatApp@2024" | base64
# Output: Q2hhdEFwcEAyMDI0
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
  mongo-root-username: YWRtaW4=          # "admin" base64 encoded
  mongo-root-password: Q2hhdEFwcEAyMDI0  # your password base64 encoded
```

```bash
kubectl apply -f secret.yml
```

> **Gotcha:** Forgetting `-n` in `echo -n "..."` encodes a trailing newline — the decoded value will be wrong and MongoDB auth will fail silently.

---

## Step 4 — Deploy MongoDB (with Persistent Storage)

**Why PV + PVC?** MongoDB data must survive pod restarts. Without persistent storage, every time MongoDB's pod is restarted (crash, update, node eviction), all your chat messages are gone.

### Persistent Volume (`mongodb-pv.yml`)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce               # Only one pod can mount this at a time
  persistentVolumeReclaimPolicy: Retain   # Keep data even after PVC is deleted
  hostPath:
    path: /data/mongodb           # Stores data on the Minikube node's filesystem
```

### Persistent Volume Claim (`mongodb-pvc.yml`)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
  namespace: chat-application
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi               # Must match or be less than the PV capacity
```

Apply in this order — PV must exist before PVC, PVC before Deployment:

```bash
# 1. Persistent Volume
kubectl apply -f mongodb-pv.yml
kubectl get pv                            # Status should be "Available"

# 2. Persistent Volume Claim
kubectl apply -f mongodb-pvc.yml
kubectl get pvc -n chat-application       # Status should be "Bound"

# 3. MongoDB Deployment
kubectl apply -f mongodb-deployment.yml
kubectl get pods -n chat-application      # Wait for Running

# 4. MongoDB Service
kubectl apply -f mongodb-service.yml
```

**Key point in `mongodb-deployment.yml`:** The credentials must come from the Secret:

```yaml
# mongodb-deployment.yml (relevant env section)
env:
  - name: MONGO_INITDB_ROOT_USERNAME
    valueFrom:
      secretKeyRef:
        name: mongodb-secret
        key: mongo-root-username
  - name: MONGO_INITDB_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mongodb-secret
        key: mongo-root-password
```

---

## Step 5 — Deploy Backend

**Why deploy backend before frontend?** The frontend's Nginx config has an `upstream` block pointing to the backend service by DNS name. If the backend Service doesn't exist when the frontend pod starts, Nginx fails to resolve the upstream and the frontend pod crashes.

```bash
kubectl apply -f backend-deployment.yml
kubectl apply -f backend-service.yml
```

The backend deployment must include the MongoDB connection URI:

```yaml
# backend-deployment.yml (relevant env section)
env:
  - name: MONGO_URI
    value: "mongodb://$(MONGO_USER):$(MONGO_PASSWORD)@mongodb-service:27017/chatdb"
  - name: MONGO_USER
    valueFrom:
      secretKeyRef:
        name: mongodb-secret
        key: mongo-root-username
  - name: MONGO_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mongodb-secret
        key: mongo-root-password
  - name: PORT
    value: "5001"
```

> **How service DNS works inside the cluster:** `mongodb-service` resolves to the MongoDB ClusterIP automatically. Kubernetes runs an internal DNS server (CoreDNS) that maps `<service-name>.<namespace>.svc.cluster.local` — within the same namespace you can just use the service name.

---

## Step 6 — Deploy Frontend

```bash
kubectl apply -f frontend-deployment.yml
kubectl apply -f frontend-service.yml
```

**Frontend is on port 80, Backend is on port 5001.**

The frontend's Nginx config proxies `/api` calls to the backend:

```nginx
# nginx.conf (inside the frontend container)
server {
    listen 80;
    location / {
        root /usr/share/nginx/html;
        try_files $uri /index.html;   # React SPA routing
    }
    location /api {
        proxy_pass http://backend-service:5001;   # Forward API calls to backend
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## Step 7 — Configure Ingress

**Why Ingress?** Without Ingress, you'd need to port-forward to access the app. Ingress gives you a proper domain name (`chat-ns.com`) and routes all traffic through a single entry point.

Enable the ingress addon in Minikube first (this installs the nginx-ingress controller):

```bash
minikube addons enable ingress

# Verify the controller is running
kubectl get pods -n ingress-nginx
```

### Ingress Manifest (`ingress.yml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: chat-ingress
  namespace: chat-application
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"   # Allow larger file uploads
spec:
  ingressClassName: nginx
  rules:
    - host: chat-ns.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yml
kubectl get ing -n chat-application
```

Add an entry to your `/etc/hosts` to resolve the domain locally:

```bash
# Get the Minikube IP
minikube ip
# Example output: 192.168.49.2

# Add to /etc/hosts
echo "192.168.49.2  chat-ns.com" | sudo tee -a /etc/hosts
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
kubectl logs -l app=backend -n chat-application --tail=50  # Logs from all backend pods
```

### Describe a failing pod

```bash
kubectl describe pod <pod-name> -n chat-application
# Look at the Events section at the bottom — most errors are explained there
# Common events to look for:
#   "Back-off pulling image" → wrong image name or not pushed to Docker Hub
#   "OOMKilled" → container needs more memory
#   "CrashLoopBackOff" → app keeps crashing, check logs
```

### Test MongoDB connection from inside the pod

```bash
kubectl exec -it <mongodb-pod> -n chat-application -- mongosh \
  -u admin -p ChatApp@2024 --authenticationDatabase admin
# If this connects, the Secret password is correct
```

### Common errors encountered

| Error | Cause | Fix |
|-------|-------|-----|
| Frontend CrashLoopBackOff | Backend service not deployed yet | Apply `backend-service.yml` first |
| MongoDB pod pending | PVC not bound to PV | Check namespace matches in pv/pvc YAMLs |
| Auth failed on MongoDB | Secret password not base64 encoded | Re-encode with `echo -n "password" \| base64` |
| Ingress not routing | Ingress addon not enabled | `minikube addons enable ingress` |
| Backend can't reach MongoDB | Wrong service name in MONGO_URI | Service must be `mongodb-service` in same namespace |
| Image pull error | Image not pushed to Docker Hub | `docker push <username>/<image>:latest` |

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

# Verify everything
kubectl get all -n chat-application
```

Or apply all at once (order not guaranteed — use if all dependencies are already met):

```bash
kubectl apply -f k8s/
```
