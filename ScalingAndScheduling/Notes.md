# Kubernetes — Scaling & Scheduling

---

## 1. Resource Quotas

### Why Resource Limits Matter

Without resource limits, a single runaway pod can consume all CPU or memory on a node, causing every other pod on that node to be killed or evicted. Requests and limits give the scheduler the information it needs to pack pods efficiently.

| Field | Meaning | Effect |
|-------|---------|--------|
| `requests` | Minimum guaranteed resources the pod needs to be **scheduled** | Scheduler uses this to decide which node has room |
| `limits` | Maximum resources the pod is **allowed** to consume | Container is throttled (CPU) or OOMKilled (memory) if exceeded |

> **CPU is compressible.** If a pod exceeds its CPU limit, it gets throttled — it runs slower but doesn't crash.  
> **Memory is not compressible.** If a pod exceeds its memory limit, it is killed immediately (`OOMKilled`).

### Pod-Level Resource Configuration

Add under `spec.template.spec.containers` in your deployment:

```yaml
resources:
  requests:
    cpu: 100m       # 100 millicores = 0.1 CPU core
    memory: 128Mi   # 128 mebibytes
  limits:
    cpu: 200m       # Pod is throttled if it tries to use more than 0.2 cores
    memory: 256Mi   # Pod is killed if it uses more than 256Mi
```

### Namespace-Level ResourceQuota

You can also cap the total resources *all* pods in a namespace can use combined. This prevents one team's namespace from consuming the whole cluster.

```yaml
# resourcequota.yml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"           # All pods in namespace can request max 4 CPUs total
    requests.memory: 4Gi        # Max 4Gi memory requested in this namespace
    limits.cpu: "8"             # Max 8 CPUs limit across all pods
    limits.memory: 8Gi
    pods: "20"                  # Max 20 pods in this namespace
    persistentvolumeclaims: "5" # Max 5 PVCs
```

```bash
kubectl apply -f resourcequota.yml
kubectl describe resourcequota dev-quota -n development
# Shows used vs hard limits for each resource
```

> **Gotcha:** Once a ResourceQuota is applied to a namespace, every pod in that namespace *must* have resource requests/limits set — otherwise the pod will be rejected. Set a `LimitRange` to provide defaults so you don't have to specify them on every deployment.

### LimitRange — Default Requests/Limits for a Namespace

```yaml
# limitrange.yml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
    - type: Container
      default:          # Applied if container doesn't specify limits
        cpu: 200m
        memory: 256Mi
      defaultRequest:   # Applied if container doesn't specify requests
        cpu: 100m
        memory: 128Mi
      max:              # No container can exceed this
        cpu: "2"
        memory: 2Gi
```

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

```bash
# Apply the deployment
kubectl apply -f deployment.yml

# Check resource usage (requires metrics-server)
kubectl top pod -n nginx

# Verify resource fields are set correctly
kubectl describe pod/<pod-name> -n nginx
# Look for: Requests / Limits sections under Containers
```

---

## 2. Probes

### Why Probes?

Without probes, Kubernetes considers a pod "ready" the moment its container process starts — even if the app hasn't finished booting. Traffic gets sent to a pod that isn't ready yet, causing errors. Probes let Kubernetes track the actual health of your app.

| Probe | When it runs | Purpose | Action on failure |
|-------|-------------|---------|------------------|
| **Startup** | Only during startup | Has the app finished initialising? | Restart the container |
| **Readiness** | Continuously after ready | Is the app ready to receive traffic? | Remove from Service endpoints (no restart) |
| **Liveness** | Continuously after ready | Is the app alive and responsive? | Restart the container |

> **Key difference between Readiness and Liveness:** Readiness failure means "don't send traffic here, but keep the pod alive." Liveness failure means "this pod is broken, kill and restart it." Use Readiness for temporary overload; use Liveness for deadlocks or unrecoverable errors.

### Probe Timing Fields

| Field | Meaning |
|-------|---------|
| `initialDelaySeconds` | How long to wait before running the first check |
| `periodSeconds` | How often to run the check |
| `failureThreshold` | How many consecutive failures before taking action |
| `successThreshold` | How many consecutive successes to consider healthy again |
| `timeoutSeconds` | How long to wait for a probe response before counting as failure |

### HTTP Probes (Most Common)

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8000
  failureThreshold: 30    # Up to 30 × 10s = 5 minutes for the app to start
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
  successThreshold: 1

livenessProbe:
  httpGet:
    path: /healthz
    port: 8000
  initialDelaySeconds: 30    # Give app time to start before liveness checks begin
  periodSeconds: 10
  failureThreshold: 3        # Restart pod after 3 consecutive failures (= 30s unresponsive)
```

### TCP Probes (For Non-HTTP Services)

Use for databases or any service that doesn't speak HTTP. Kubernetes just checks if the port accepts a connection.

```yaml
livenessProbe:
  tcpSocket:
    port: 3306    # MySQL
  initialDelaySeconds: 15
  periodSeconds: 10

readinessProbe:
  tcpSocket:
    port: 6379    # Redis
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Exec Probes (Run a Command Inside the Container)

Most flexible option — run any command. Pod is considered healthy if the command exits with code 0.

```yaml
livenessProbe:
  exec:
    command:
      - mysqladmin
      - ping
      - -h
      - localhost
  initialDelaySeconds: 30
  periodSeconds: 10
```

```yaml
readinessProbe:
  exec:
    command:
      - redis-cli
      - ping    # Returns "PONG" with exit code 0 if Redis is healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

```bash
# Check probe status under the "Conditions" section
kubectl describe pod/<pod-name> -n nginx

# Watch what happens when a liveness probe fails (pod gets restarted)
kubectl get pods -n nginx -w
# You'll see RESTARTS count increment
```

---

## 3. Taints & Tolerations

### Concept

A **taint** marks a node so the scheduler avoids placing pods on it.
A **toleration** on a pod explicitly permits it to run on a tainted node.

Together they let you **reserve nodes for specific workloads** — e.g.:
- GPU nodes: only ML training pods should schedule there (expensive, no point running nginx on them)
- Production nodes: only prod pods tolerate prod taints
- Spot/preemptible nodes: only fault-tolerant jobs tolerate `spot=true`

| Taint Effect | Behaviour |
|-------------|-----------|
| `NoSchedule` | New pods are not scheduled unless they tolerate the taint |
| `PreferNoSchedule` | Scheduler avoids the node but may still place pods there |
| `NoExecute` | Evicts **existing** pods that don't tolerate the taint + blocks new ones |

### Example: Reserve a Node for Production

```bash
# Reserve node-1 for production workloads
kubectl taint node node-1 environment=production:NoSchedule

# Verify — try deploying without a toleration, pod should stay Pending
kubectl describe pod/<pod-name>
# Events: "0/1 nodes available: 1 node(s) had untolerated taint {environment: production}"

# Remove the taint (trailing dash removes it)
kubectl taint node node-1 environment=production:NoSchedule-
```

### Toleration Configuration

Add under `spec.template.spec` in your deployment:

```yaml
tolerations:
  - key: "environment"
    operator: "Equal"
    value: "production"
    effect: "NoSchedule"
```

With this toleration the pod can be scheduled on any node tainted with `environment=production:NoSchedule`.

### Multiple Taints

A node can have multiple taints. A pod must tolerate **all** of them to be scheduled there.

```bash
# A GPU node with two taints
kubectl taint node gpu-node-1 gpu=true:NoSchedule
kubectl taint node gpu-node-1 spot=true:NoExecute

# Pod must tolerate both
tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  - key: "spot"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"
```

```bash
# Apply the deployment with tolerations
kubectl apply -f deployment.yml

# Confirm the pod is now scheduled (no longer Pending)
kubectl get pods -n nginx -o wide    # NODE column shows which node it landed on
```

---

## 4. HPA & VPA (Autoscaling)

### When to Use Which

| Type | Scales | Best for | Limitation |
|------|--------|---------|-----------|
| **HPA** | Number of pod replicas | Stateless apps (web servers, APIs) | Requires resource requests to be set; needs metrics-server |
| **VPA** | CPU / memory per pod | Stateful apps, right-sizing | Restarts pods to apply new values |
| **KEDA** | Replicas based on any event source | Queue workers, batch jobs | Needs KEDA operator installed |

### Metrics Server (Required for HPA and VPA)

HPA and VPA both require metrics-server to be running in the cluster.

```bash
# Check if metrics are available
kubectl top node
kubectl top pod -n <namespace>
```

If you see `metrics API not available`, metrics-server is not installed.

```bash
# Install metrics-server (for EC2 / kubeadm clusters, add --kubelet-insecure-tls)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# For EC2 (kubeadm), patch to skip TLS verification:
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Wait ~60s then verify
kubectl top nodes
```

---

### HPA — Horizontal Pod Autoscaler

**How it works:** HPA watches CPU (or custom metrics) and adjusts the number of running replicas to keep utilisation near the target. Scale up when busy, scale down when idle.

#### Configuration (`hpa.yml`)

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
          averageUtilization: 50    # Scale up when average CPU > 50%
```

```bash
kubectl apply -f hpa.yml

# Watch HPA status — TARGETS shows current/desired utilisation
kubectl get hpa -n apache
# NAME         REFERENCE                    TARGETS   MINPODS   MAXPODS   REPLICAS
# apache-hpa   Deployment/apache-deployment  12%/50%   1         5         1
```

#### Generate Load to Trigger Scaling

```bash
# Setup — port-forward the service
sudo -E kubectl port-forward service/apache-service -n apache 82:80 --address=0.0.0.0

# Spin up a load generator pod
kubectl run -i --tty load-generator --image=busybox -n apache -- /bin/sh

# Inside the load generator pod:
while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done
```

Watch replicas increase with `kubectl get hpa -n apache -w` in another terminal. When you stop the load, HPA scales back down (default cooldown is 5 minutes).

#### HPA with Memory Metric

```yaml
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 200Mi    # Scale up if average memory per pod > 200Mi
```

---

### VPA — Vertical Pod Autoscaler

**How it works:** VPA monitors actual resource usage and recommends (or automatically applies) updated requests/limits per pod. Unlike HPA, it doesn't add replicas — it resizes the existing ones.

**When to use VPA:** When you don't know the right CPU/memory values for a new app, run VPA in `Off` mode first to get recommendations, then tune your manifests.

#### Configuration (`vpa.yml`)

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
    updateMode: "Off"      # Start with Off to just get recommendations safely
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

| Mode | Behaviour | Use When |
|------|-----------|---------|
| `Off` | Generate recommendations only — apply nothing | Initial tuning, auditing |
| `Initial` | Apply recommendations only at pod creation | Controlled environments |
| `Auto` | Evict and recreate pods with new resource values | Confident in VPA recommendations |

```bash
# Install VPA (clone autoscaler repo and run install script first)
kubectl apply -f vpa.yml

# Check VPA recommendations
kubectl describe vpa apache-vpa -n apache
# Look for: "Recommendation:" section showing target CPU/memory

# Example recommendation output:
#   Container Recommendations:
#     Container Name: apache
#       Lower Bound:    cpu: 50m, memory: 64Mi
#       Target:         cpu: 120m, memory: 180Mi   ← Use these values in your deployment
#       Upper Bound:    cpu: 500m, memory: 512Mi
```

> **HPA and VPA conflict:** Don't use both on the same deployment targeting CPU metrics. HPA scales replicas when CPU is high; VPA reduces CPU requests, which confuses HPA. Use HPA for scaling and VPA for non-CPU metrics (memory), or just use one.

---

### KEDA — Event-Driven Autoscaling

KEDA extends HPA to scale based on **external event sources** — not just CPU. Perfect for queue workers: scale up when messages pile up, scale to zero when the queue is empty.

#### Install KEDA

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

#### Example: Scale a Worker Based on Redis Queue Depth

```yaml
# keda-scaledobject.yml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: voting-worker-scaler
  namespace: default
spec:
  scaleTargetRef:
    name: worker-deployment     # The Deployment to scale
  minReplicaCount: 0            # Scale to zero when queue is empty (saves cost)
  maxReplicaCount: 10
  triggers:
    - type: redis
      metadata:
        address: redis-service:6379
        listName: votes          # Redis list being used as a queue
        listLength: "5"          # Scale up when list has > 5 items per replica
```

```bash
kubectl apply -f keda-scaledobject.yml

# Watch KEDA scale the worker as votes pile up in Redis
kubectl get scaledobject -n default
kubectl get pods -l app=worker -w
```

> **Scale to zero:** With KEDA, `minReplicaCount: 0` lets the worker scale completely down when there's no work — something standard HPA can't do (HPA minimum is 1). This is a big cost saving for batch/queue workloads.

#### Other KEDA Triggers

```yaml
# Scale based on number of messages in an SQS queue
- type: aws-sqs-queue
  metadata:
    queueURL: https://sqs.us-east-1.amazonaws.com/123456789/my-queue
    queueLength: "10"
    awsRegion: us-east-1

# Scale based on Kafka consumer lag
- type: kafka
  metadata:
    bootstrapServers: kafka-service:9092
    consumerGroup: my-consumer-group
    topic: events
    lagThreshold: "100"    # Scale up when lag > 100 messages per replica

# Scale based on a Prometheus metric
- type: prometheus
  metadata:
    serverAddress: http://prometheus-stack:9090
    metricName: http_requests_in_flight
    query: sum(http_requests_in_flight{job="myapp"})
    threshold: "50"
```

---

## 5. Node Affinity

### Why Node Affinity?

Taints/tolerations are about repulsion — "don't put pods here." Node affinity is about attraction — "put pods here specifically."

**Use cases:**
- Pin database pods to high-memory nodes
- Place frontend pods on nodes with SSDs (lower latency)
- Keep dev and prod pods on separate node groups
- Ensure pods run only in a specific region/AZ

| Rule Type | Behaviour |
|-----------|-----------|
| `requiredDuringSchedulingIgnoredDuringExecution` | **Hard rule** — pod stays Pending if no node matches |
| `preferredDuringSchedulingIgnoredDuringExecution` | **Soft rule** — scheduler prefers matching nodes but falls back if needed |

### Step 1 — Label a Node

```bash
# Label a node for production high-memory workloads
kubectl label node <node-name> region=us-east
kubectl label node <node-name> tier=high-memory
kubectl label node <node-name> disk=ssd

# Verify labels
kubectl get nodes --show-labels
kubectl get nodes -l region=us-east    # Filter nodes by label
```

### Step 2 — Hard Affinity Rule (`required`)

Pod will **not** schedule unless a node exactly matches.

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: region
              operator: In
              values:
                - us-east       # Must be in us-east
            - key: disk
              operator: In
              values:
                - ssd           # AND must have an SSD
```

### Step 3 — Soft Affinity Rule (`preferred`)

Pod prefers matching nodes but schedules anywhere if none match.

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80          # 1–100; higher weight = stronger preference
        preference:
          matchExpressions:
            - key: region
              operator: In
              values:
                - us-east
      - weight: 20
        preference:
          matchExpressions:
            - key: disk
              operator: In
              values:
                - ssd
```

### Pod Anti-Affinity — Spread Pods Across Nodes

Anti-affinity ensures replicas don't all land on the same node. If that node goes down, you lose all replicas at once without it.

```yaml
affinity:
  podAntiAffinity:
    # Hard rule: replicas of the same app MUST be on different nodes
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: nginx         # Match pods with this label
        topologyKey: kubernetes.io/hostname   # Spread across different hostnames (nodes)

    # Soft rule: prefer spreading across AZs, but don't block if only one AZ available
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: nginx
          topologyKey: topology.kubernetes.io/zone   # Prefer different AZs
```

> **`topologyKey` explained:** This is the node label used as the "spread domain." `kubernetes.io/hostname` spreads across nodes. `topology.kubernetes.io/zone` spreads across availability zones. The scheduler ensures no two matching pods share the same value of this label.

### Full Example Deployment with Affinity + Anti-Affinity

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: nginx
spec:
  replicas: 3    # 3 replicas, each forced to a different node
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
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: nginx
              topologyKey: kubernetes.io/hostname   # Each replica on a different node
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deployment.yml

# NODE column confirms each pod is on a different node
kubectl get pods -n nginx -o wide

# Check why a pod is (or isn't) on a particular node
kubectl describe pod/<pod-name> -n nginx
# Look for: "Node-Selectors" and "Tolerations" sections
```

### Operator Reference

| Operator | Meaning |
|----------|---------|
| `In` | Label value is in the provided list |
| `NotIn` | Label value is NOT in the provided list |
| `Exists` | Label key exists (any value) |
| `DoesNotExist` | Label key does not exist |
| `Gt` / `Lt` | Label value is greater / less than (numeric strings) |

---

## Quick Reference

```bash
# Resource usage
kubectl top nodes
kubectl top pods -n <namespace>
kubectl describe resourcequota -n <namespace>

# HPA
kubectl apply -f hpa.yml
kubectl get hpa -n <namespace>
kubectl describe hpa <name> -n <namespace>

# VPA
kubectl get vpa -n <namespace>
kubectl describe vpa <name> -n <namespace>    # Shows recommendations

# KEDA
kubectl get scaledobject -n <namespace>
kubectl get triggerauthentication -n <namespace>

# Taints
kubectl taint node <node> <key>=<value>:<effect>
kubectl taint node <node> <key>=<value>:<effect>-   # Remove (trailing dash)
kubectl describe node <node> | grep Taints

# Node Labels (for affinity)
kubectl label node <node> <key>=<value>
kubectl get nodes --show-labels
kubectl get nodes -l <key>=<value>

# Pods — scheduling info
kubectl get pods -o wide              # Shows which node each pod is on
kubectl describe pod <pod> | grep -A5 "Node-Selectors\|Tolerations\|Affinity"
```
