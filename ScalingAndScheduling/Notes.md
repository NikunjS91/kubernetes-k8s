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

readinessProbe:
  httpGet:
    path: /
    port: 8000
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
