# Kubernetes Storage — Persistent Volumes, Claims & StorageClasses

---

## Why Storage?

Pods are ephemeral — if a pod is deleted, any data stored inside it is lost. To persist data across pod restarts and deletions, Kubernetes uses **Persistent Volumes (PV)** and **Persistent Volume Claims (PVC)**.

**Think of it like this:**
- A **PV** is like a physical hard drive in a data centre — it exists independently of any app
- A **PVC** is like a ticket a pod submits saying "I need 1GB of disk" — Kubernetes finds a matching PV and binds them together
- **StorageClass** automates the whole process — no need to manually create PVs

---

## Storage Options Overview

| Type | Lifecycle | Use Case |
|------|-----------|----------|
| `emptyDir` | Lives as long as the pod | Temporary scratch space, shared between containers in a pod |
| `hostPath` | Lives as long as the node | Access host filesystem (logs, Docker socket) — avoid in production |
| `PersistentVolume` | Independent of pods | Databases, user uploads, anything that must survive pod restarts |
| `ConfigMap / Secret` | Managed by K8s | Inject config files or credentials as read-only volumes |

---

## Concepts

| Term | Description |
|------|-------------|
| **PersistentVolume (PV)** | A piece of storage provisioned in the cluster. Cluster-level resource — not namespaced. |
| **PersistentVolumeClaim (PVC)** | A namespaced request by a pod to use a PV. Bound to a matching PV. |
| **StorageClass** | A template that dynamically creates PVs on demand — no manual PV creation needed. |
| **VolumeMount** | Attaches the volume at a specific path inside a container. |

---

## Access Modes

Access modes define how many nodes can mount the volume simultaneously.

| Mode | Short | Meaning | Use Case |
|------|-------|---------|----------|
| `ReadWriteOnce` | RWO | One node can read/write | Databases (MySQL, Postgres, MongoDB) |
| `ReadOnlyMany` | ROX | Many nodes can read | Shared config, static assets |
| `ReadWriteMany` | RWX | Many nodes can read/write | Shared file systems (NFS, EFS) |

> **Most cloud block storage (AWS EBS, GCP PD) only supports RWO.** For RWX you need network file systems like AWS EFS or an NFS server.

---

## Reclaim Policies

What happens to the PV after the PVC is deleted?

| Policy | Behaviour | Use When |
|--------|-----------|----------|
| `Retain` | PV stays, data preserved — must be manually reclaimed | Production databases |
| `Delete` | PV and underlying storage deleted automatically | Cloud-managed storage where you want clean teardown |
| `Recycle` | Deprecated — data wiped, PV made available again | Avoid |

---

## Static Provisioning (Manual PV)

You create the PV yourself, then a PVC claims it. Used when the storage already exists (e.g., an NFS share, an existing EBS volume).

### Step 1 — Create a PersistentVolume (`persistentVolume.yml`)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
  labels:
    app: local
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  hostPath:
    path: /mnt/data    # Data stored on the node at this path (Minikube/Kind only)
```

```bash
kubectl apply -f persistentVolume.yml
kubectl get pv    # STATUS should show "Available"
```

> **`hostPath` is for local development only.** It ties the volume to a specific node — if your pod moves to a different node (e.g., after a crash), the data won't follow. Use cloud storage (EBS, GCP PD) or NFS for anything real.

### Step 2 — Create a PersistentVolumeClaim (`persistentVolumeClaim.yml`)

The PVC asks for storage matching the access mode and storageClassName. Kubernetes finds the best-fit PV and binds them.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
  namespace: nginx
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi       # Must be <= PV capacity
  storageClassName: local-storage   # Must match PV storageClassName
```

```bash
kubectl apply -f persistentVolumeClaim.yml
kubectl get pvc -n nginx   # STATUS should show "Bound"
kubectl get pv             # PV STATUS changes to "Bound", CLAIM shows the PVC name
```

### Step 3 — Mount the PVC in a Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: nginx
spec:
  replicas: 1
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
          volumeMounts:
            - mountPath: /var/www/html   # Path inside the container
              name: my-volume
      volumes:
        - name: my-volume
          persistentVolumeClaim:
            claimName: local-pvc         # Must match the PVC metadata.name
```

```bash
kubectl apply -f deployment.yml
kubectl get pods -n nginx

# Verify the volume is mounted inside the pod
kubectl exec -it <pod-name> -n nginx -- df -h
# You'll see /var/www/html listed with 1Gi capacity
```

---

## Dynamic Provisioning (StorageClass)

With static provisioning you have to manually create a PV before every PVC. **Dynamic provisioning** automates this — when a PVC is created, Kubernetes uses the StorageClass to automatically create a matching PV in the cloud.

**Why this matters:** In production on AWS/GCP, you never manually create PVs. You define a StorageClass once, and every PVC that references it gets its own EBS/PD volume created automatically.

### StorageClass Example (AWS EBS)

```yaml
# storageclass-ebs.yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com          # AWS EBS CSI driver
parameters:
  type: gp3                            # EBS volume type (gp3 is faster and cheaper than gp2)
  encrypted: "true"                    # Encrypt data at rest
reclaimPolicy: Delete                  # Delete EBS volume when PVC is deleted
volumeBindingMode: WaitForFirstConsumer  # Don't create EBS until a pod is scheduled
                                         # (ensures volume is in same AZ as pod)
allowVolumeExpansion: true             # Allow resizing PVCs without downtime
```

```bash
kubectl apply -f storageclass-ebs.yml
kubectl get storageclass
```

### PVC Using a StorageClass (No Manual PV Needed)

```yaml
# pvc-dynamic.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-storage
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc   # StorageClass handles PV creation automatically
  resources:
    requests:
      storage: 20Gi
```

```bash
kubectl apply -f pvc-dynamic.yml

# Watch the PVC — STATUS goes from Pending → Bound as EBS volume is created
kubectl get pvc -n production -w
```

> **`WaitForFirstConsumer`** is important on multi-AZ clusters. Without it, the EBS volume might be created in `us-east-1a` but your pod schedules to `us-east-1b` — and EBS volumes can't cross AZs.

### Default StorageClass

Most managed clusters (EKS, GKE, AKS) ship with a default StorageClass. If you create a PVC without specifying `storageClassName`, the default is used automatically.

```bash
# See which StorageClass is the default (marked with "(default)")
kubectl get storageclass

# Example output:
# NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      DEFAULT
# gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   true
# gp3             ebs.csi.aws.com         Delete          WaitForFirstConsumer   false
```

---

## emptyDir — Temporary Pod-Scoped Storage

`emptyDir` is created when a pod starts and deleted when the pod is removed. It's perfect for sharing data between containers in the same pod (like a log file shared between an app and a sidecar).

```yaml
# Two containers sharing a scratch directory
apiVersion: v1
kind: Pod
metadata:
  name: shared-storage-demo
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "while true; do echo $(date) >> /data/output.log; sleep 5; done"]
      volumeMounts:
        - name: shared-data
          mountPath: /data

    - name: reader
      image: busybox
      command: ["sh", "-c", "tail -f /data/output.log"]
      volumeMounts:
        - name: shared-data
          mountPath: /data    # Same volume — reads what writer produces

  volumes:
    - name: shared-data
      emptyDir: {}    # Created fresh when pod starts, deleted when pod dies
```

> **emptyDir survives container restarts** (within the same pod) but is gone when the pod itself is deleted or evicted.

---

## ConfigMap as a Volume (Inject Config Files)

Instead of environment variables, you can mount a ConfigMap as a file. Useful when your app reads a config file at startup.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: nginx
data:
  nginx.conf: |
    server {
        listen 80;
        location / {
            root /usr/share/nginx/html;
        }
        location /api {
            proxy_pass http://backend-service:5000;
        }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: nginx
spec:
  template:
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          volumeMounts:
            - name: nginx-conf
              mountPath: /etc/nginx/conf.d    # ConfigMap key becomes a file here
              readOnly: true
      volumes:
        - name: nginx-conf
          configMap:
            name: nginx-config
            items:
              - key: nginx.conf
                path: default.conf    # Mounted as /etc/nginx/conf.d/default.conf
```

---

## Real-World Example — MySQL with Persistent Storage

This is the full setup you'd use for a database in production: StorageClass (dynamic), PVC, and Deployment wired together.

```yaml
# mysql-pvc.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc   # Uses dynamic provisioning
  resources:
    requests:
      storage: 20Gi
---
# mysql-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: production
spec:
  replicas: 1    # Never run multiple MySQL replicas with a single PVC — use StatefulSet for HA
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql    # MySQL stores all databases here
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-pvc
```

---

## Troubleshooting

### Pod stuck in `Pending` state

```bash
kubectl describe pod/<pod-name> -n nginx
# Look for events like:
# "pod has unbound immediate PersistentVolumeClaims"
# "no persistent volumes available for this claim"
```

**Common causes:**

| Problem | Symptom | Fix |
|---------|---------|-----|
| PVC and PV in different namespaces | PVC stays Pending | PVs are cluster-scoped — only PVCs are namespaced. Remove `namespace` from PV metadata |
| `storageClassName` mismatch | PVC stays Pending | PVC and PV must have the same `storageClassName` |
| Access mode mismatch | PVC stays Pending | PVC `accessModes` must be a subset of PV's `accessModes` |
| PVC requests more than PV has | PVC stays Pending | PVC `storage` request must be <= PV `capacity.storage` |
| AZ mismatch (cloud) | PVC stays Pending | Use `volumeBindingMode: WaitForFirstConsumer` in StorageClass |

**Fix — delete and re-apply PV/PVC:**

```bash
kubectl delete pvc/<pvc-name> -n nginx
kubectl delete pv/<pv-name>

kubectl apply -f persistentVolume.yml
kubectl apply -f persistentVolumeClaim.yml
```

### Check PV/PVC binding status

```bash
kubectl get pv
# RECLAIM POLICY   STATUS    CLAIM
# Retain           Bound     nginx/local-pvc

kubectl get pvc -n nginx
# STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS
# Bound    local-pv   1Gi        RWO            local-storage

# Check what's mounted inside the pod
kubectl exec -it <pod-name> -n nginx -- df -h
kubectl exec -it <pod-name> -n nginx -- ls /var/www/html
```

---

## Quick Reference

```bash
# PV / PVC
kubectl get pv                          # All PVs (cluster-wide)
kubectl get pvc -n <namespace>          # PVCs in a namespace
kubectl describe pvc <name> -n <ns>     # Detailed status + events
kubectl get storageclass                # Available StorageClasses

# Volumes inside a pod
kubectl exec -it <pod> -n <ns> -- df -h        # Disk usage
kubectl exec -it <pod> -n <ns> -- ls <mount>   # Files in mounted volume

# Delete in correct order (PVC first, then PV)
kubectl delete pvc <name> -n <ns>
kubectl delete pv <name>
```
