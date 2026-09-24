# Kubernetes Storage — Persistent Volumes

---

## Why Storage?

Pods are ephemeral — if a pod is deleted, any data stored inside it is lost. To persist data across pod restarts and deletions, Kubernetes uses **Persistent Volumes (PV)** and **Persistent Volume Claims (PVC)**.

---

## Concepts

| Term | Description |
|---|---|
| **PersistentVolume (PV)** | A piece of storage provisioned on the host (e.g., `/mnt/data`). Cluster-level resource. |
| **PersistentVolumeClaim (PVC)** | A request by a pod to use a PV. Bound to a PV that satisfies the claim. |

---

## Step 1 — Create a PersistentVolume

`persistentVolume.yml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
  namespace: nginx
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
    path: /mnt/data
```

```bash
kubectl apply -f persistentVolume.yml
kubectl get pv
```

---

## Step 2 — Create a PersistentVolumeClaim

`persistentVolumeClaim.yml`
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
      storage: 1Gi
  storageClassName: local-storage
```

```bash
kubectl apply -f persistentVolumeClaim.yml
kubectl get pvc   # status should show "Bound"
kubectl get pv    # also shows the bound state
```

---

## Step 3 — Mount the PVC in a Deployment

Add `volumeMounts` and `volumes` to your deployment spec:

```yaml
# Inside spec.template.spec.containers
volumeMounts:
  - mountPath: /var/www/html
    name: my-volume

# Inside spec.template.spec
volumes:
  - name: my-volume
    persistentVolumeClaim:
      claimName: local-pvc
```

---

## Troubleshooting

**Pod stuck in `Pending` state?**
```bash
kubectl describe pod/<pod-name> -n nginx
```
Common cause: missing `namespace` field in the PV or PVC YAML.

**Fix — delete and re-apply PV/PVC after adding namespace:**
```bash
kubectl delete pvc/<pvc-name> -n nginx
kubectl delete pv/<pv-name>

kubectl apply -f persistentVolume.yml
kubectl apply -f persistentVolumeClaim.yml
```

After re-applying, the pod should transition to `Running`.
