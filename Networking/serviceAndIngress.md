# Kubernetes Networking — Services, Ingress & StatefulSets

---

## 1. Services

Once pods are running, they need to be accessible — either internally or externally. A **Service** acts as a stable network endpoint that routes traffic to the right pods using label selectors, even as pods are created and destroyed.

### Why Services?

- Pods get new IP addresses every time they restart — Services give you a stable address.
- Services load-balance traffic across multiple pod replicas automatically.
- They decouple the consumer from knowing which pod to talk to.

### Service Types

| Type | Scope | Use Case |
|------|-------|----------|
| `ClusterIP` | Internal only | Communication between services inside the cluster |
| `NodePort` | Exposes on a node's IP + port | Simple external access for testing |
| `LoadBalancer` | External via cloud LB | Production external access on cloud providers |
| `ExternalName` | DNS alias | Pointing to an external service by name |

### Creating a Service (`service.yml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: nginx
spec:
  selector:
    app: nginx           # Must match the pod/deployment label
  ports:
    - protocol: TCP
      port: 80           # Port the service listens on
      targetPort: 80     # Port the pod container listens on
  type: ClusterIP
```

```bash
kubectl apply -f service.yml
```

> **Tip:** `selector.app: nginx` must exactly match the label defined in your Deployment's pod template (`metadata.labels.app: nginx`). A mismatch means no traffic gets routed.

### View all resources in a namespace

```bash
kubectl get all -n nginx
```

---

## 2. Accessing Services — Port Forwarding

`ClusterIP` services are only reachable inside the cluster. To access them locally or from a remote machine (e.g., an EC2 instance), use `kubectl port-forward`.

```bash
# Basic port forward — local machine only
kubectl port-forward service/nginx-service -n nginx 8080:80

# Expose to all network interfaces (required for EC2 / remote access)
kubectl port-forward service/nginx-service -n nginx 80:80 --address=0.0.0.0

# On EC2, port 80 requires root — use sudo -E to preserve env variables
sudo -E kubectl port-forward service/nginx-service -n nginx 80:80 --address=0.0.0.0
```

> **Note:** If port 80 is already in use, try port 8080 or another free port: `8080:80`.

After running this on EC2, open **port 80** (or whichever port you chose) in the EC2 Security Group **Inbound rules**, then visit `http://<your-ec2-ip>` in a browser.

---

## 3. Ingress

### The Problem Port-Forwarding Solves (And Doesn't)

Port forwarding works for a single service. But what if you have **multiple apps** — a frontend, a backend, an admin panel — all running in the same cluster? You'd need to manage multiple ports.

**Ingress** solves this by acting as a smart reverse proxy: one entry point, route traffic to different services based on the URL path or hostname.

```
http://12.161.181.168/nginx  →  nginx-service
http://12.161.181.168/app    →  myapp-service
```

### Prerequisites

- Both services must live in the **same namespace**.
- An **Ingress Controller** must be installed (it does the actual routing work).

### Install nginx-ingress Controller (for Kind/local clusters)

```bash
# Install the ingress-nginx controller (check GitHub for the latest version URL)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Verify it's running
kubectl get pods -n ingress-nginx
```

> The controller creates its own namespace (`ingress-nginx`) and sets up a LoadBalancer/NodePort service automatically.

### Creating an Ingress (`ingress.yml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /    # Strips the path prefix before forwarding
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /nginx
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
          - path: /app
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yml

# Check ingress status
kubectl get ing -n nginx
```

### Expose the Ingress Controller

```bash
sudo -E kubectl port-forward service/ingress-nginx-controller -n ingress-nginx 8080:80 --address=0.0.0.0
```

Then open port `8080` in the EC2 Security Group and visit `http://<ec2-ip>:8080/nginx` or `http://<ec2-ip>:8080/app`.

### Annotations Explained

Without the `rewrite-target` annotation, your app receives `/nginx` as the path — most apps aren't expecting that prefix. The annotation strips it so the app only sees `/`.

| Annotation | Purpose |
|------------|---------|
| `nginx.ingress.kubernetes.io/rewrite-target: /` | Rewrites the path to `/` before sending to the backend |
| `nginx.ingress.kubernetes.io/ssl-redirect: "false"` | Disables automatic HTTP → HTTPS redirect |

---

## 4. StatefulSets

### Deployments vs StatefulSets

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod identity | Random names (`app-abc123`) | Stable, ordered names (`mysql-0`, `mysql-1`) |
| Storage | Shared or ephemeral | Each pod gets its own persistent volume |
| Scaling | Parallel | Ordered (one at a time) |
| Use case | Stateless apps (Flask, Django) | Stateful apps (MySQL, MongoDB, Kafka) |

### Why StatefulSets for Databases?

If a pod in a Deployment is deleted, the replacement gets a new name and potentially different storage. For databases, this breaks data consistency. StatefulSets guarantee:
- Same pod name after restart (`mysql-0` is always `mysql-0`)
- Same PersistentVolumeClaim is reattached
- Ordered startup/shutdown (safe for leader/follower replication setups)

### StatefulSet with MySQL (`statefulset.yml`)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: mysql
spec:
  serviceName: mysql-headless    # Must match the headless Service name
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:                      # NOT "templates" — common mistake!
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
            - name: MYSQL_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: mysql-config
                  key: database-name
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

### Headless Service for StatefulSet (`service.yml`)

A StatefulSet needs a **headless service** (no ClusterIP) so that each pod gets its own DNS entry (`mysql-0.mysql-headless`).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  namespace: mysql
spec:
  clusterIP: None          # This makes it headless
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```

```bash
kubectl apply -f service.yml
kubectl apply -f statefulset.yml

# Watch pods start up (refreshes every 2 seconds)
watch kubectl get pods -n mysql
```

### Verify MySQL is Working

```bash
# Shell into the pod
kubectl exec -it mysql-0 -n mysql -- bash

# Inside the pod
mysql -u root -p
# Enter your password, then:
show databases;
```

> **Behaviour:** Delete `mysql-0` and Kubernetes immediately recreates it with the same name, same PVC, and same data intact.

---

## 5. ConfigMaps

### The Problem

Hardcoding configuration values (database names, URLs, feature flags) inside your YAML manifests makes them hard to change and reuse across environments.

**ConfigMaps** let you store config data separately and inject it into pods as environment variables or mounted files.

### Creating a ConfigMap (`configmap.yml`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: mysql
data:
  database-name: myappdb
  max-connections: "100"
```

```bash
kubectl apply -f configmap.yml
```

### Using a ConfigMap in a Pod

```yaml
env:
  - name: MYSQL_DATABASE
    valueFrom:
      configMapKeyRef:
        name: mysql-config
        key: database-name
```

> **Tip:** To update a config value, edit the ConfigMap and re-apply it. Pods using `envFrom` need to be restarted to pick up the change; pods using volume mounts see updates automatically (after a short delay).

---

## 6. Secrets

### Why Not ConfigMaps for Passwords?

ConfigMap values are stored in plain text. For sensitive data — passwords, API keys, tokens — use **Secrets**, which are base64-encoded and can be further protected with RBAC or external secret managers.

### Encoding a Password

```bash
echo -n "Nikunj112233" | base64
# Output: TmlrdW5qMTEyMjMz
```

> Use `-n` to avoid encoding a trailing newline, which would make the decoded value wrong.

### Creating a Secret (`secret.yml`)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: mysql
type: Opaque
data:
  root-password: TmlrdW5qMTEyMjMz    # base64 encoded value
```

```bash
kubectl apply -f secret.yml
```

### Using a Secret in a Pod

```yaml
env:
  - name: MYSQL_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: root-password
```

```bash
# Re-apply statefulset to pick up the secret
kubectl apply -f statefulset.yml
```

> **Important:** base64 is encoding, not encryption. Anyone with access to the cluster can decode Secrets. For production, use tools like **HashiCorp Vault**, **AWS Secrets Manager**, or Kubernetes **Sealed Secrets** to truly encrypt sensitive values.

---

## Quick Reference

```bash
# Apply all files in a directory
kubectl apply -f .

# Get all resources in a namespace
kubectl get all -n <namespace>

# Describe a resource (useful for debugging)
kubectl describe svc nginx-service -n nginx
kubectl describe ingress nginx-ingress -n nginx

# Get ingress details
kubectl get ing -n nginx

# Watch pod status (refreshes every 2s)
watch kubectl get pods -n mysql

# Exec into a pod
kubectl exec -it <pod-name> -n <namespace> -- bash

# Check logs
kubectl logs <pod-name> -n <namespace>
```
