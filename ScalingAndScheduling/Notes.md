# Kubernetes — Scaling & Scheduling

---

## 1. Resource Quotas

Without resource limits, a single pod can consume all memory on a node, starving other pods.
Setting **requests** and **limits** ensures fair resource allocation across pods.

| Field | Meaning |
|---|---|
| `requests` | Minimum guaranteed resources the pod needs to be scheduled |
| `limits` | Maximum resources the pod is allowed to consume |

---

### Configuration

Add under `spec.template.spec.containers` in your deployment:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

---

### Full Example — `deployment.yml`

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
          image: nginx:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
```

---

### Commands

```bash
# Apply the deployment
kubectl apply -f deployment.yml

# List pods
kubectl get pods -n nginx

# Verify resource fields are set correctly
kubectl describe pod/<pod-name> -n nginx
```

---

## 2. Probes

Probes let Kubernetes check whether a pod is healthy at different stages of its lifecycle.

| Probe | When it runs | Purpose |
|---|---|---|
| **Startup Probe** | During pod startup | Verifies the app has started successfully |
| **Readiness Probe** | From startup → ready | Checks if the pod is ready to receive traffic |
| **Liveness Probe** | Continuously after ready | Checks if the pod is still alive; restarts it if not |

Probe timing fields:

| Field | Meaning |
|---|---|
| `initialDelaySeconds` | How long to wait before running the first check |
| `periodSeconds` | How often to run the check |
| `failureThreshold` | How many consecutive failures before taking action |

---

### Configuration — HTTP Probes

Add under `spec.template.spec.containers` in your deployment:

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8000
  failureThreshold: 30    # allow up to 30 × periodSeconds for slow starts
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 5

livenessProbe:
  httpGet:
    path: /
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3     # restart pod after 3 consecutive failures
```

---

### Configuration — TCP & Exec Probes

Use **TCP** for non-HTTP services (e.g. databases):

```yaml
livenessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 15
  periodSeconds: 10
```

Use **Exec** to run a command inside the container:

```yaml
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

### Commands

```bash
# Apply the deployment
kubectl apply -f deployment.yml

# List pods
kubectl get pods -n nginx

# Check probe status under the "Conditions" section
kubectl describe pod/<pod-name> -n nginx
```

---

## 3. Taints & Tolerations

### Concept

A **taint** marks a node so the scheduler avoids placing pods on it.
A **toleration** on a pod explicitly permits it to run on a tainted node.

Together they let you reserve nodes for specific workloads (e.g. GPU nodes, prod-only nodes).

| Taint Effect | Behaviour |
|---|---|
| `NoSchedule` | New pods are not scheduled unless they tolerate the taint |
| `PreferNoSchedule` | Scheduler avoids the node but may still place pods there |
| `NoExecute` | Evicts existing pods that don't tolerate the taint |

---

### Taint Commands

```bash
# List all nodes
kubectl get nodes

# Add a taint — pods without a matching toleration will stay Pending
kubectl taint node <node-name> prod=true:NoSchedule

# Verify — "0/1 nodes available" means all nodes are tainted
kubectl describe pod/<pod-name> -n nginx

# Remove the taint (trailing dash removes it)
kubectl taint node <node-name> prod=true:NoSchedule-
```

---

### Configuration — Tolerations

Add under `spec.template.spec` in your deployment:

```yaml
tolerations:
  - key: "prod"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

With this toleration the pod can be scheduled on any node tainted with `prod=true:NoSchedule`.

---

### Commands

```bash
# Apply the deployment with tolerations
kubectl apply -f deployment.yml

# Confirm the pod is now scheduled (no longer Pending)
kubectl get pods -n nginx -o wide
```

---

## 4. HPA & VPA (Autoscaling)

| Type | Full Name | Scales | Best for |
|---|---|---|---|
| **HPA** | Horizontal Pod Autoscaler | Number of pod replicas | Stateless apps (nginx, apache) |
| **VPA** | Vertical Pod Autoscaler | CPU / memory per pod | Stateful apps (MySQL) |

**KEDA** (Kubernetes Event-Driven Autoscaling) extends HPA to support external event sources like queue depth, Kafka topics, or custom metrics.

---

### Metrics Server

HPA and VPA both require metrics-server to be running in the cluster.

```bash
# Check if metrics are available
kubectl top node
kubectl top pod -n <namespace>
```

If you see `metrics API not available`, metrics-server is not installed.
Install it (for EC2 add `--kubelet-insecure-tls` flag) and wait ~60s for data to populate.

---

### HPA Example — Apache

#### Explanation

HPA watches CPU utilization and scales the number of pod replicas between `minReplicas` and `maxReplicas`.
When average CPU across all pods exceeds `averageUtilization`, new replicas are added.

---

#### Setup

```bash
# Verify namespace, deployment, and service are running
kubectl get all -n apache

# Port-forward the service to test from your machine
sudo -E kubectl port-forward service/apache-service -n apache 82:80 --address=0.0.0.0
# Access via: <ec2-ip>:82
# Cluster-internal DNS: http://apache-service.apache.svc.cluster.local
```

---

#### Configuration — `hpa.yml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: apache-hpa
  namespace: apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apache-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50    # scale up when average CPU > 50%
```

---

#### Commands

```bash
# Apply the HPA
kubectl apply -f hpa.yml

# Watch the HPA status (TARGETS shows current vs desired utilization)
kubectl get hpa -n apache
```

---

#### Generate Load to Trigger Scaling

```bash
# Spin up a busybox pod and run a load loop
kubectl run -i --tty load-generator --image=busybox -n apache -- /bin/sh

# Inside the pod:
while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done
```

Watch replicas increase with `kubectl get hpa -n apache` in another terminal.

---

### VPA Example — Apache

#### Explanation

VPA monitors actual CPU/memory usage and recommends (or automatically applies) updated resource requests/limits per pod.
Unlike HPA it does not add more pods — it resizes existing ones.

---

#### Configuration — `vpa.yml`

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: apache-vpa
  namespace: apache
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apache-deployment
  updatePolicy:
    updateMode: "Auto"      # evict and recreate pods with updated resource values
  resourcePolicy:
    containerPolicies:
      - containerName: apache
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: 500m
          memory: 512Mi
```

`updateMode` options:

| Mode | Behaviour |
|---|---|
| `Off` | Only generate recommendations, apply nothing |
| `Initial` | Apply recommendations only at pod creation |
| `Auto` | Evict and recreate pods with new resource values |

---

#### Commands

```bash
# Install VPA (clone autoscaler repo and run install script first)
kubectl apply -f vpa.yml

# Check VPA object and its current recommendations
kubectl get vpa -n apache
kubectl describe vpa apache-vpa -n apache

# Monitor actual pod resource usage
kubectl top pod -n apache
```

---

## 5. Node Affinity

Node affinity constrains which nodes a pod can be scheduled on using node labels.
Use it when you need placement control beyond taints/tolerations — e.g. schedule only on nodes in `us-east`, or prefer GPU nodes.

| Rule Type | Behaviour |
|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | Hard rule — pod stays Pending if no node matches |
| `preferredDuringSchedulingIgnoredDuringExecution` | Soft rule — scheduler prefers matching nodes but won't block |

---

### Step 1 — Label the Node

```bash
# Add a label to the target node
kubectl label node <node-name> region=us-east

# Verify the label was applied
kubectl get nodes --show-labels
```

---

### Step 2 — Configuration — Hard Rule (`required`)

Add under `spec.template.spec` in your deployment:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: region
              operator: In
              values:
                - us-east
```

Pod will **not** schedule unless a node has `region=us-east`.

---

### Step 3 — Configuration — Soft Rule (`preferred`)

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80          # 1–100; higher = stronger preference
        preference:
          matchExpressions:
            - key: region
              operator: In
              values:
                - us-east
```

Pod **prefers** `us-east` nodes but falls back to any available node.

---

### Full Example — `deployment.yml`

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
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: region
                    operator: In
                    values:
                      - us-east
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

---

### Commands

```bash
# Apply the deployment
kubectl apply -f deployment.yml

# NODE column confirms which node each pod was placed on
kubectl get pods -n nginx -o wide

# Check "Node-Selectors" and affinity details
kubectl describe pod/<pod-name> -n nginx
```

---

### Operator Reference

| Operator | Meaning |
|---|---|
| `In` | Label value is in the provided list |
| `NotIn` | Label value is NOT in the provided list |
| `Exists` | Label key exists (any value) |
| `DoesNotExist` | Label key does not exist |
| `Gt` / `Lt` | Label value is greater / less than (numeric strings) |
