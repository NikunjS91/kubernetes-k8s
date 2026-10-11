# Kubernetes Cluster Administration — RBAC, Dashboard & CRDs

---

## 1. RBAC (Role-Based Access Control)

### Why RBAC?

By default, anyone with `kubectl` access to your cluster can do anything — create, delete, modify every resource. RBAC lets you define *who* can do *what* on *which* resources, and then enforce it.

**Real scenario:** You hire an intern. They have `kubectl` access for debugging, but they accidentally run `kubectl delete deployment --all`. RBAC prevents this by giving them read-only access.

**Another scenario:** A monitoring tool (Prometheus) needs to list pods and services across the cluster. You create a ServiceAccount for it and grant only `get` and `list` on those resources — not `delete` or `create`. If the tool is ever compromised, the blast radius is limited.

### Core Concepts

| Concept | Scope | What it does |
|---------|-------|-------------|
| **Role** | Namespace | Defines allowed actions on resources within a namespace |
| **ClusterRole** | Cluster-wide | Same as Role but applies across all namespaces |
| **ServiceAccount** | Namespace | An identity for a pod or automated process |
| **User** | Cluster-wide | A human identity (managed externally — no K8s User resource) |
| **RoleBinding** | Namespace | Grants a Role to a User or ServiceAccount |
| **ClusterRoleBinding** | Cluster-wide | Grants a ClusterRole to a User or ServiceAccount |

### Useful Diagnostic Commands

```bash
kubectl auth whoami                             # Who am I right now?
kubectl auth can-i get pods                     # Can I list pods?
kubectl auth can-i delete deployments -n apache # Can I delete deployments in 'apache'?
kubectl auth can-i get pods --as=apache-user -n apache  # What can apache-user do?

# List all permissions a ServiceAccount has (useful for auditing)
kubectl auth can-i --list --as=system:serviceaccount:apache:apache-user -n apache
```

---

### Namespace-Level RBAC (ServiceAccount + Role + RoleBinding)

#### Step 1 — Create a Namespace & Deployment

```bash
kubectl create namespace apache
kubectl apply -f deployment.yml   # Your app deployment in the apache namespace
```

#### Step 2 — Define a Role (`role.yml`)

A Role lists the exact verbs (get, list, create, delete…) allowed on specific resources.

**Read-only role (for interns / support staff):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: apache-reader
  namespace: apache
rules:
  - apiGroups: [""]           # "" = core API group (pods, services, etc.)
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]   # Read-only — no create/delete
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
```

**Developer role (can deploy but not delete):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: apache-developer
  namespace: apache
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
    # No "delete" — developers can't remove deployments
  - apiGroups: [""]
    resources: ["pods/log", "pods/exec"]   # Can exec into pods and view logs
    verbs: ["get", "create"]
```

```bash
kubectl apply -f role.yml
kubectl get role -n apache
```

> **Verbs available:** `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection`, `impersonate`

#### Step 3 — Create a ServiceAccount (`service-account.yml`)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: apache-user
  namespace: apache
```

```bash
kubectl apply -f service-account.yml
kubectl get serviceaccount -n apache

# Test before binding — should return "no"
kubectl auth can-i get pods -n apache --as=system:serviceaccount:apache:apache-user
```

#### Step 4 — Bind Role to ServiceAccount (`role-binding.yml`)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: apache-reader-binding
  namespace: apache
subjects:
  - kind: ServiceAccount
    name: apache-user
    namespace: apache
roleRef:
  kind: Role
  name: apache-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f role-binding.yml
kubectl get rolebinding -n apache

# Now test — should return "yes"
kubectl auth can-i get pods -n apache --as=system:serviceaccount:apache:apache-user

# Still blocked — should return "no"
kubectl auth can-i delete pods -n apache --as=system:serviceaccount:apache:apache-user
```

#### Using a ServiceAccount Inside a Pod

When a pod needs to call the Kubernetes API (e.g., a custom controller or CI tool), assign it a ServiceAccount:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: apache
spec:
  serviceAccountName: apache-user   # Pod uses this identity for API calls
  containers:
    - name: app
      image: myapp:latest
```

> Inside the pod, credentials are automatically mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`. Libraries like the official Kubernetes Go/Python clients pick this up automatically.

---

### Cluster-Level RBAC (ClusterRole + ClusterRoleBinding)

Use ClusterRole when you need access across *all* namespaces — e.g., a monitoring agent that scrapes metrics from every namespace, or a logging tool that reads pod logs cluster-wide.

```yaml
# clusterrole-monitor.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-monitor
rules:
  - apiGroups: [""]
    resources: ["pods", "nodes", "namespaces", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "daemonsets", "statefulsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list"]           # Needed for kubectl top
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-monitor-binding
subjects:
  - kind: ServiceAccount
    name: monitor-user
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: cluster-monitor
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f clusterrole-monitor.yml

# Test that monitoring SA can list pods in any namespace
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:monitor-user -n kube-system
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:monitor-user -n production
```

---

## 2. Kubernetes Dashboard

The Kubernetes Dashboard is a web UI to visualise and manage your cluster — view pod status, logs, resource usage, and apply manifests — all in a browser.

### Install the Dashboard

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Verify it's running
kubectl get pods -n kubernetes-dashboard
```

### Create an Admin ServiceAccount (`dashboard-admin-user.yml`)

This file defines both the ServiceAccount and its ClusterRoleBinding in one file (separated by `---`).

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user-binding
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
roleRef:
  kind: ClusterRole
  name: cluster-admin           # Built-in superuser ClusterRole — full access to everything
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f dashboard-admin-user.yml

# Check what ClusterRoles are available
kubectl get clusterrole | grep -v system   # Filter out system roles
```

### Get an Access Token

```bash
kubectl -n kubernetes-dashboard create token admin-user
# Copy the output — this is your login token

# For a long-lived token (not recommended for production):
kubectl -n kubernetes-dashboard create token admin-user --duration=8760h
```

> **Token expiry:** Tokens created with `create token` expire in 1 hour by default. If your dashboard session drops, re-run this command to get a fresh token.

### Start the Proxy

```bash
# Run in foreground
kubectl proxy --port=8001 --address=0.0.0.0 --accept-hosts='.*'

# Run in background (appending & keeps the shell free)
kubectl proxy --port=8001 --address=0.0.0.0 --accept-hosts='.*' &
```

### Access the Dashboard

Open in your browser:

```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

Select **Token**, paste the token from above, and sign in.

**What you can do in the Dashboard:**
- View pods, deployments, services across all namespaces
- Check pod logs without running `kubectl logs`
- Scale deployments up/down with a slider
- View resource usage (CPU/memory) if metrics-server is installed
- Apply YAML manifests via a built-in editor

---

## 3. Custom Resource Definitions (CRDs)

### What Are CRDs?

Kubernetes has built-in resource types: `Pod`, `Deployment`, `Service`, `ConfigMap`, etc. Each has a defined structure (apiVersion, kind, spec fields).

**CRDs let you define your own resource types** — with your own schema — that Kubernetes then treats as first-class citizens. You can `kubectl apply`, `kubectl get`, and `kubectl describe` them just like any built-in resource.

### How K8s Knows About Its Own Resources

Every resource has an `apiVersion` and `kind`:
- `Pod` → `apiVersion: v1` (no group prefix = core group, version = v1)
- `Deployment` → `apiVersion: apps/v1` (`apps` = group, `v1` = version)

CRDs follow the same pattern but with a custom group name you define.

### Real-World CRD Examples You Already Use

| Tool | CRD Kind | What it represents |
|------|---------|-------------------|
| ArgoCD | `Application` | A GitOps deployment target |
| Cert-Manager | `Certificate` | A TLS certificate to auto-issue |
| Prometheus Operator | `ServiceMonitor` | Which services Prometheus should scrape |
| Istio | `VirtualService` | Traffic routing rules |

When you run `kubectl get application -n argocd`, that `Application` type is a CRD — not built into Kubernetes. This is how tools extend Kubernetes without forking it.

### Create a CRD (`devops-crd.yml`)

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: devopstools.mycompany.io    # Must be: plural.group
spec:
  group: mycompany.io
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["toolName", "environment"]   # Validation: these fields are mandatory
              properties:
                toolName:
                  type: string
                  description: "Name of the DevOps tool"
                version:
                  type: string
                  pattern: '^[0-9]+\.[0-9]+\.[0-9]+$'   # Validates semver format
                environment:
                  type: string
                  enum: ["development", "staging", "production"]   # Only these values allowed
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
  scope: Namespaced               # Or Cluster for cluster-wide resources
  names:
    plural: devopstools
    singular: devopstool
    kind: DevOpsTool
    shortNames:
      - dot                       # kubectl get dot
```

```bash
mkdir crd && cd crd
kubectl apply -f devops-crd.yml

# Verify the CRD was registered
kubectl get crd
kubectl get crd devopstools.mycompany.io
```

### Create an Instance of Your Custom Resource

Now that the CRD is registered, create a resource using your custom `kind`:

```yaml
# argocd-instance.yml
apiVersion: mycompany.io/v1
kind: DevOpsTool
metadata:
  name: argocd-prod
  namespace: default
spec:
  toolName: ArgoCD
  version: "2.9.0"
  environment: production
  replicas: 3
```

```bash
kubectl apply -f argocd-instance.yml

# Use the full kind or the shortname
kubectl get devopstools
kubectl get dot

kubectl describe devopstool argocd-prod

# Try an invalid value — K8s will reject it because of the enum validation
kubectl apply -f - <<EOF
apiVersion: mycompany.io/v1
kind: DevOpsTool
metadata:
  name: bad-tool
spec:
  toolName: BadTool
  environment: local   # Not in the enum — will fail
EOF
# Error: spec.environment: Unsupported value: "local"
```

### Controllers — The Other Half of the CRD Story

A CRD alone is just a data store — Kubernetes saves your resource but does nothing with it. The real power comes from writing a **Controller** (also called an Operator) that watches for your custom resources and acts on them.

```
You create a DevOpsTool resource
        ↓
Controller detects the new resource (via watch)
        ↓
Controller installs ArgoCD Helm chart
        ↓
Controller updates DevOpsTool status: { installed: true, endpoint: "http://..." }
```

This is how tools like the Prometheus Operator work: you create a `ServiceMonitor` CR, and the controller automatically updates Prometheus's scrape config. You never touch Prometheus config directly.

---

## Quick Reference

```bash
# RBAC
kubectl auth whoami
kubectl auth can-i <verb> <resource> -n <namespace>
kubectl auth can-i --list -n <namespace> --as=system:serviceaccount:<ns>:<sa>
kubectl get role -n <namespace>
kubectl get rolebinding -n <namespace>
kubectl get clusterrole
kubectl get clusterrolebinding

# Dashboard
kubectl -n kubernetes-dashboard create token admin-user
kubectl proxy --port=8001 --address=0.0.0.0 --accept-hosts='.*' &

# CRDs
kubectl get crd
kubectl describe crd <name>
kubectl get <custom-resource-plural>
kubectl explain <kind>           # Shows the schema of any resource including CRDs

# Bulk delete all resources in a directory
kubectl delete -f .
```
