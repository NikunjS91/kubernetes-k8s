# Kubernetes — Scaling & Scheduling

---

## 1. Resource Quotas

Without resource limits, a single pod can consume all memory on a node, starving other pods. Setting **requests** and **limits** ensures fair resource allocation.

| Field | Meaning |
|---|---|
| `requests` | Minimum guaranteed resources the pod needs to be scheduled |
| `limits` | Maximum resources the pod is allowed to consume |

Add to your deployment YAML under `spec.template.spec.containers`:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

**Full deployment example with resource limits (`deployment.yml`):**

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

```bash
kubectl apply -f deployment.yml
kubectl get pods -n nginx
kubectl describe pod/<pod-name> -n nginx   # verify resource fields
```

---

## 2. Probes

Probes let Kubernetes check whether a pod is healthy at different stages of its lifecycle.

| Probe | When it runs | Purpose |
|---|---|---|
| **Startup Probe** | During pod startup | Verifies the app has started successfully |
| **Readiness Probe** | From startup → ready | Checks if the pod is ready to receive traffic |
| **Liveness Probe** | Continuously after ready | Checks if the pod is still alive; restarts it if not |

Add to your deployment YAML under `spec.template.spec.containers`:

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 8000
  initialDelaySeconds: 10   # wait 10s before first check (let app start)
  periodSeconds: 5          # check every 5s
  failureThreshold: 3       # restart pod after 3 consecutive failures

readinessProbe:
  httpGet:
    path: /
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 5

startupProbe:
  httpGet:
    path: /healthz
    port: 8000
  failureThreshold: 30      # allow up to 30 * periodSeconds for slow starts
  periodSeconds: 10
```

**Alternative probe types:**

```yaml
# TCP socket probe — useful for non-HTTP services (e.g. databases)
livenessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 15
  periodSeconds: 10

# Exec probe — runs a command inside the container
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

```bash
kubectl apply -f deployment.yml
kubectl get pods -n nginx
kubectl describe pod/<pod-name> -n nginx   # check probe status under "Conditions"
```

---

## 3. Taints & Tolerations

### Taints

A **taint** tells the scheduler **not** to place pods on a specific node — unless the pod explicitly tolerates it.

```bash
# List all nodes
kubectl get nodes

# Taint a node (NoSchedule = don't schedule new pods here)
kubectl taint node <node-name> prod=true:NoSchedule

# Apply a deployment — pods will stay Pending if all nodes are tainted
kubectl describe pod/<pod-name> -n nginx   # shows scheduling failure reason

# Remove the taint
kubectl taint node <node-name> prod=true:NoSchedule-
```

---

### Tolerations

A **toleration** on a pod allows it to be scheduled onto a tainted node.

Add to your pod/deployment YAML under `spec.template.spec`:

```yaml
tolerations:
  - key: "prod"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

With this toleration, the pod can be scheduled on nodes tainted with `prod=true:NoSchedule`.

---

## 4. HPA & VPA (Autoscaling)

| Type | Full Name | What it does | Use case |
|---|---|---|---|
| **HPA** | Horizontal Pod Autoscaler | Increases/decreases **number of pods** | Stateless apps (nginx, apache) |
| **VPA** | Vertical Pod Autoscaler | Increases/decreases **CPU/memory** per pod | Stateful apps (MySQL) |

**KEDA** (Kubernetes Event Driven Autoscaling) — selects HPA or VPA based on metrics or external events (queue depth, CPU, etc.).

---

### Metrics Server

Required for HPA/VPA to read CPU and memory usage:

```bash
kubectl top node              # node-level metrics
kubectl top pod -n <namespace>  # pod-level metrics
```

If `metrics API not available`, the metrics-server is not installed — check `kube-system` and install it (refer to docs for EC2-specific flags).

---

### HPA Example (Apache)

**Setup namespace, deployment, and service:**
```bash
kubectl get all -n apache
```

**Port forward to test locally:**
```bash
sudo -E kubectl port-forward service/apache-service -n apache 82:80 --address=0.0.0.0
```

Add EC2 inbound rule for the port, then access via `<ec2-ip>:82`.

**DNS access within cluster:**
```
http://apache-service.apache.svc.cluster.local
```

---

**`hpa.yml`**
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
          averageUtilization: 5
```

```bash
kubectl apply -f hpa.yml
kubectl get hpa -n apache
```

**Generate load to trigger scaling:**
```bash
kubectl run -i --tty load-generator --image=busybox -n apache -- /bin/sh
# Inside the pod:
while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done
```

---

### VPA Example (Apache)

**`vpa.yml`**
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
    updateMode: "Auto"   # Auto = VPA can evict and recreate pods with new resources
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

> `updateMode` options: `Off` (recommendations only), `Initial` (apply on pod creation), `Auto` (evict and recreate).

```bash
# Clone the autoscaler repo and run the install commands from docs
# Then create vpa.yml, apply namespace/deployment/service
kubectl apply -f vpa.yml
kubectl get vpa -n apache
kubectl describe vpa apache-vpa -n apache   # shows recommended CPU/memory values
kubectl top pod -n apache
```

---

## 5. Node Affinity

Node affinity lets you constrain which nodes a pod can be scheduled on — for example, scheduling only on nodes in a specific region or datacenter.

Used when you need **fine-grained control** over pod placement beyond simple taints/tolerations.

| Rule Type | Behavior |
|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | Hard rule — pod will not schedule if no node matches |
| `preferredDuringSchedulingIgnoredDuringExecution` | Soft rule — scheduler prefers matching nodes but won't block scheduling |

**Step 1 — Label the node:**
```bash
kubectl label node <node-name> region=us-east
kubectl get nodes --show-labels   # verify the label
```

**Step 2 — Add node affinity to your deployment (`deployment.yml`):**

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

**Soft preference example (prefers `us-east`, falls back to any node):**

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80          # higher weight = stronger preference (1–100)
        preference:
          matchExpressions:
            - key: region
              operator: In
              values:
                - us-east
```

```bash
kubectl apply -f deployment.yml
kubectl get pods -n nginx -o wide   # NODE column shows which node each pod landed on
kubectl describe pod/<pod-name> -n nginx   # check "Node-Selectors" and "Tolerations"
```

**Operator options for `matchExpressions`:**

| Operator | Meaning |
|---|---|
| `In` | Label value is in the list |
| `NotIn` | Label value is NOT in the list |
| `Exists` | Label key exists (any value) |
| `DoesNotExist` | Label key does not exist |
| `Gt` / `Lt` | Label value is greater/less than (numeric strings) |
