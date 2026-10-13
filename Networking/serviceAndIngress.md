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

### Service YAML Fields Explained

| Field | Value / Example | Description |
|-------|----------------|-------------|
| `apiVersion` | `v1` | Services are in the core API group |
| `kind` | `Service` | Resource type |
| `metadata.name` | `nginx-service` | DNS name used by other pods to reach this service |
| `metadata.namespace` | `nginx` | Namespace the service lives in |
| `spec.selector` | `app: nginx` | Label query — routes traffic to pods with this label |
| `spec.ports[].port` | `80` | Port the **service** listens on (what callers connect to) |
| `spec.ports[].targetPort` | `80` | Port the **pod container** listens on (where traffic is forwarded) |
| `spec.ports[].nodePort` | `30080` | NodePort only — static port exposed on every node (30000–32767) |
| `spec.ports[].protocol` | `TCP` | `TCP` (default), `UDP`, or `SCTP` |
| `spec.type` | `ClusterIP` | Service type — controls how the service is exposed |

### Port Fields Side by Side

| Field | Who uses it | Example | Analogy |
|-------|------------|---------|---------|
| `port` | Other pods calling this service | `80` | The front door of the building |
| `targetPort` | The container inside the pod | `8080` | The actual office inside the building |
| `nodePort` | External users / browsers | `30080` | The building's street address |

> `port` and `targetPort` can be different. E.g. the service listens on `80` but your Node.js app listens on `3000` — set `port: 80, targetPort: 3000`.

### Example 1 — ClusterIP (Internal Service)

Use this when only other pods inside the cluster need to reach this service. The backend calling a database is a perfect example.

```yaml
# service-clusterip.yml
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

### Example 2 — NodePort (External Access for Testing)

NodePort exposes the service on a static port (30000–32767) on every node. You access it at `<any-node-ip>:<nodePort>`. Useful for local Kind/Minikube setups where there's no cloud load balancer.

```yaml
# service-nodeport.yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
  namespace: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80           # ClusterIP port (internal)
      targetPort: 80     # Pod port
      nodePort: 30080    # External port on every node (must be 30000–32767)
  type: NodePort
```

```bash
# Access via any node IP:
# http://<node-ip>:30080

# On Minikube:
minikube service nginx-nodeport -n nginx --url
# Prints the full URL to open in your browser
```

> **When to use NodePort vs LoadBalancer:** NodePort is for local dev and testing — it works without a cloud provider. LoadBalancer is for production on AWS/GCP/Azure where the cloud creates a real load balancer with a public IP.

### Service Type Decision Guide

| Scenario | Use Type | Why |
|----------|---------|-----|
| Backend pod calling a database pod | `ClusterIP` | No external access needed — internal only |
| Testing your app on Minikube/Kind | `NodePort` | No cloud LB available — access via node IP |
| Production app on AWS/GCP/Azure | `LoadBalancer` | Gets a real public IP automatically |
| Redirecting to an external URL/domain | `ExternalName` | DNS alias — no proxy, just CNAME |
| Internal microservice-to-microservice | `ClusterIP` | Keeps traffic inside the cluster |
| Database in a StatefulSet | `ClusterIP: None` (headless) | Direct pod DNS, no load balancing |

```bash
kubectl apply -f service.yml
```

> **Tip:** `selector.app: nginx` must exactly match the label defined in your Deployment's pod template (`metadata.labels.app: nginx`). A mismatch means no traffic gets routed.

### View all resources in a namespace

```bash
kubectl get all -n nginx

# Check which pods a service is routing to:
kubectl describe svc nginx-service -n nginx
# Look at "Endpoints:" — these are the pod IPs currently selected
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

> **Port forwarding is not production-grade.** It's a development/debugging tool. For real external traffic, use NodePort, LoadBalancer, or Ingress.

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

### Example 1 — Path-Based Routing (`ingress-path.yml`)

Route traffic to different services based on the URL path — all on the same IP.

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

### Example 2 — Host-Based Routing (Multiple Domains)

Route traffic to different services based on the domain name — cleaner for production where each app has its own subdomain.

```yaml
# ingress-host.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
  namespace: production
spec:
  ingressClassName: nginx
  rules:
    - host: api.myapp.com          # api.myapp.com → backend service
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 5000

    - host: myapp.com              # myapp.com → frontend service
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

> For local testing with host-based routing, add entries to `/etc/hosts`:
> ```
> <cluster-ip>  myapp.com
> <cluster-ip>  api.myapp.com
> ```

### Example 3 — TLS / HTTPS Ingress

In production you always want HTTPS. Create a TLS Secret and reference it in the Ingress.

```bash
# Generate a self-signed cert for local testing
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=myapp.com"

# Create the TLS Secret
kubectl create secret tls myapp-tls --key tls.key --cert tls.crt -n production
```

```yaml
# ingress-tls.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"   # Force HTTP → HTTPS
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.com
      secretName: myapp-tls        # The Secret created above
  rules:
    - host: myapp.com
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

# Check ingress status — ADDRESS column shows the assigned IP/hostname
kubectl get ing -n nginx
```

### Expose the Ingress Controller

```bash
sudo -E kubectl port-forward service/ingress-nginx-controller -n ingress-nginx 8080:80 --address=0.0.0.0
```

Then open port `8080` in the EC2 Security Group and visit `http://<ec2-ip>:8080/nginx` or `http://<ec2-ip>:8080/app`.

### Ingress YAML Fields Explained

| Field | Value / Example | Description |
|-------|----------------|-------------|
| `metadata.annotations` | `rewrite-target: /` | Controller-specific behaviour config (not part of K8s core spec) |
| `spec.ingressClassName` | `nginx` | Which Ingress Controller handles this resource |
| `spec.tls[].hosts` | `myapp.com` | Domains to apply TLS to |
| `spec.tls[].secretName` | `myapp-tls` | K8s Secret holding the TLS cert and key |
| `spec.rules[].host` | `api.myapp.com` | Domain to match (omit for any domain) |
| `spec.rules[].http.paths[].path` | `/api` | URL path prefix to match |
| `spec.rules[].http.paths[].pathType` | `Prefix` | How the path is matched — see table below |
| `backend.service.name` | `backend-service` | Service to forward matched traffic to |
| `backend.service.port.number` | `5000` | Service port to forward to |

### Ingress `pathType` Values

| pathType | Behaviour | Example |
|----------|-----------|---------|
| `Prefix` | Matches the path and anything under it | `/api` matches `/api`, `/api/users`, `/api/v2/orders` |
| `Exact` | Matches only the exact path string | `/api` matches only `/api` — not `/api/users` |
| `ImplementationSpecific` | Controller decides (often regex) | Used for nginx regex captures like `/api/(.*)` |

### Annotations Explained

Without the `rewrite-target` annotation, your app receives `/nginx` as the path — most apps aren't expecting that prefix. The annotation strips it so the app only sees `/`.

| Annotation | Value | Purpose |
|------------|-------|---------|
| `nginx.ingress.kubernetes.io/rewrite-target` | `/` | Rewrites the path before sending to the backend |
| `nginx.ingress.kubernetes.io/ssl-redirect` | `"false"` | Disables automatic HTTP → HTTPS redirect |
| `nginx.ingress.kubernetes.io/proxy-body-size` | `"10m"` | Increase max upload size (default 1m) |
| `nginx.ingress.kubernetes.io/rate-limit` | `"100"` | Rate limit: max requests per second per IP |
| `nginx.ingress.kubernetes.io/proxy-connect-timeout` | `"60"` | Seconds to wait when connecting to backend |
| `nginx.ingress.kubernetes.io/proxy-read-timeout` | `"60"` | Seconds to wait for backend response |
| `nginx.ingress.kubernetes.io/use-regex` | `"true"` | Enable regex in path matching |

---

## 4. StatefulSets

### Deployments vs StatefulSets

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod identity | Random names (`app-abc123`) | Stable, ordered names (`mysql-0`, `mysql-1`) |
| Storage | Shared or ephemeral | Each pod gets its own persistent volume |
| Scaling order | All pods start/stop in parallel | Ordered one at a time (`mysql-0` before `mysql-1`) |
| DNS per pod | No — only service-level DNS | Yes — `mysql-0.mysql-headless.mysql.svc.cluster.local` |
| Use case | Stateless apps (Flask, Django, Nginx) | Stateful apps (MySQL, MongoDB, Kafka, Zookeeper) |
| Service needed | Regular ClusterIP service | Headless service (`clusterIP: None`) |
| Pod restart behaviour | New pod gets a new name | Same name reused — `mysql-0` always comes back as `mysql-0` |

### StatefulSet YAML Fields Explained

| Field | Value / Example | Description |
|-------|----------------|-------------|
| `spec.serviceName` | `mysql-headless` | Name of the headless Service — must exist before StatefulSet |
| `spec.replicas` | `1` | Number of pod replicas (each gets its own PVC) |
| `spec.template` | — | Pod definition — same as Deployment (NOT `templates`) |
| `spec.volumeClaimTemplates` | — | PVC template — K8s creates one PVC per replica automatically |
| `volumeClaimTemplates[].accessModes` | `ReadWriteOnce` | Each PVC is exclusive to its pod |
| `volumeClaimTemplates[].storage` | `1Gi` | Storage per replica |

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
          # Health check — K8s won't send traffic until MySQL is ready
          readinessProbe:
            exec:
              command: ["mysqladmin", "ping", "-h", "localhost"]
            initialDelaySeconds: 10
            periodSeconds: 5
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

> **Why headless for StatefulSets?** With a regular ClusterIP service, DNS resolves to the service IP and requests are load-balanced randomly across pods. With a headless service, DNS returns the actual pod IPs — so `mysql-0.mysql-headless` always resolves to pod `mysql-0` specifically. This is important for databases where you need to target the primary vs replica.

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

# Verify persistence: delete the pod — same name recreated with same data
kubectl delete pod mysql-0 -n mysql
kubectl get pods -n mysql -w   # Watch mysql-0 come back
kubectl exec -it mysql-0 -n mysql -- mysql -u root -p -e "show databases;"
# Your databases are still there
```

> **Behaviour:** Delete `mysql-0` and Kubernetes immediately recreates it with the same name, same PVC, and same data intact.

---

## 5. ConfigMaps

### The Problem

Hardcoding configuration values (database names, URLs, feature flags) inside your YAML manifests makes them hard to change and reuse across environments.

**ConfigMaps** let you store config data separately and inject it into pods as environment variables or mounted files.

### ConfigMap Fields Explained

| Field | Description |
|-------|-------------|
| `data` | Key-value pairs for plain text config |
| `binaryData` | Key-value pairs for binary data (base64 encoded) |
| `data.<key>` | A single config value (string) |
| `data.<key>: \|` | A multiline value — entire file content as a single key |

### ConfigMap Injection Methods

| Method | YAML keyword | Best for | Reload on change? |
|--------|-------------|---------|------------------|
| Single env var | `configMapKeyRef` | Picking specific keys | No — pod restart needed |
| All keys as env vars | `configMapRef` (via `envFrom`) | Injecting all config at once | No — pod restart needed |
| Mounted as a file | `configMap` (via `volumes`) | Apps that read config files | Yes — auto-updated after ~1 min |

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
  app-config.properties: |
    db.host=mysql-headless
    db.port=3306
    log.level=INFO
```

```bash
kubectl apply -f configmap.yml
```

### Using a ConfigMap — As Environment Variables

```yaml
env:
  - name: MYSQL_DATABASE
    valueFrom:
      configMapKeyRef:
        name: mysql-config
        key: database-name

# Or inject all keys as env vars at once:
envFrom:
  - configMapRef:
      name: mysql-config
```

### Using a ConfigMap — As a Mounted File

Useful when your app reads a config file at startup (e.g., `app.properties`, `nginx.conf`).

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /etc/app          # ConfigMap keys become files here

volumes:
  - name: config-volume
    configMap:
      name: mysql-config
      items:
        - key: app-config.properties
          path: app.properties   # Mounted as /etc/app/app.properties
```

> **Tip:** To update a config value, edit the ConfigMap and re-apply it. Pods using `envFrom` need to be restarted to pick up the change; pods using volume mounts see updates automatically (after a short delay).

---

## 6. Secrets

### ConfigMap vs Secret — When to Use Which

| | ConfigMap | Secret |
|-|-----------|--------|
| **Storage** | Plain text in etcd | base64-encoded in etcd |
| **Use for** | App config, feature flags, URLs, ports | Passwords, API keys, TLS certs, tokens |
| **Readable by** | Anyone with `kubectl get cm` | Anyone with `kubectl get secret` (RBAC can restrict) |
| **Auto-base64** | No | Yes — values must be base64 encoded |
| **Volume mount** | Read-only files | Read-only files |
| **Production security** | Safe | Use Vault / Sealed Secrets for true encryption |

### Why Not ConfigMaps for Passwords?

ConfigMap values are stored in plain text. For sensitive data — passwords, API keys, tokens — use **Secrets**, which are base64-encoded and can be further protected with RBAC or external secret managers.

### Encoding a Password

```bash
echo -n "Nikunj112233" | base64
# Output: TmlrdW5qMTEyMjMz

# Decode to verify:
echo "TmlrdW5qMTEyMjMz" | base64 -d
# Output: Nikunj112233
```

> Use `-n` to avoid encoding a trailing newline, which would make the decoded value wrong.

### Secret Types

| `type` | Use for | Example |
|--------|---------|---------|
| `Opaque` | Generic key-value secrets (passwords, tokens) | DB passwords, API keys |
| `kubernetes.io/tls` | TLS certificates | Ingress HTTPS cert + key |
| `kubernetes.io/dockerconfigjson` | Private Docker registry credentials | Pulling images from private repos |
| `kubernetes.io/service-account-token` | Auto-mounted SA token | Pod → K8s API authentication |
| `kubernetes.io/basic-auth` | HTTP basic auth username + password | Legacy auth systems |
| `kubernetes.io/ssh-auth` | SSH private key | Git SSH deployments |

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
  replication-password: cmVwbGljYXRpb24xMjM=
```

```bash
kubectl apply -f secret.yml

# View secret (values are base64 — not shown in plain text)
kubectl get secret mysql-secret -n mysql -o yaml

# Decode a specific value:
kubectl get secret mysql-secret -n mysql \
  -o jsonpath="{.data.root-password}" | base64 -d
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

## 7. Kubernetes DNS — How Services Find Each Other

Every Service automatically gets a DNS name inside the cluster. Pods use this name instead of IP addresses.

### DNS Name Format

```
<service-name>.<namespace>.svc.cluster.local
```

| Short form | Full form | Works from |
|-----------|-----------|-----------|
| `mysql-service` | `mysql-service.mysql.svc.cluster.local` | Same namespace only |
| `mysql-service.mysql` | `mysql-service.mysql.svc.cluster.local` | Any namespace |
| `mysql-service.mysql.svc.cluster.local` | (same) | Any namespace |

### DNS Examples by Resource Type

| Resource | DNS Name | Example |
|----------|---------|---------|
| Service (ClusterIP) | `<name>.<ns>.svc.cluster.local` | `backend-service.production.svc.cluster.local` |
| Headless Service pod | `<pod>.<svc>.<ns>.svc.cluster.local` | `mysql-0.mysql-headless.mysql.svc.cluster.local` |
| StatefulSet pod 0 | `<name>-0.<svc>.<ns>.svc.cluster.local` | `mysql-0.mysql-headless.mysql.svc.cluster.local` |
| StatefulSet pod 1 | `<name>-1.<svc>.<ns>.svc.cluster.local` | `mysql-1.mysql-headless.mysql.svc.cluster.local` |

### Verify DNS Resolution Inside a Pod

```bash
# Run a debug pod and test DNS
kubectl run dns-test --image=busybox --rm -it -- sh

# Inside the pod:
nslookup mysql-service.mysql          # Resolves to ClusterIP
nslookup mysql-0.mysql-headless.mysql # Resolves to pod-0 IP directly (headless)
wget -qO- http://nginx-service.nginx  # Test HTTP connectivity
```

---

## Quick Reference

```bash
# Apply all files in a directory
kubectl apply -f .

# Get all resources in a namespace
kubectl get all -n <namespace>

# Check which pods a service selects (look at Endpoints:)
kubectl describe svc <service-name> -n <namespace>

# Describe a resource (useful for debugging)
kubectl describe svc nginx-service -n nginx
kubectl describe ingress nginx-ingress -n nginx

# Get ingress details (ADDRESS shows the assigned IP)
kubectl get ing -n nginx

# Minikube: open a NodePort service in browser
minikube service <service-name> -n <namespace>

# Watch pod status (refreshes every 2s)
watch kubectl get pods -n mysql

# Exec into a pod
kubectl exec -it <pod-name> -n <namespace> -- bash

# Check logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous   # Logs from previous crash
```
