# Module 04 — Persistent Volumes Deep Dive

```
Phase       : 2 — Kubernetes Storage
Module      : 04
Status      : draft
Prereqs     : Module 03 — Containers and Persistent Storage
Estimated   : 4–5 hours
```

---

## Why This Module Exists

PV and PVC are the core Kubernetes storage abstractions. Everything Portworx does lives on top of them. When a Portworx incident happens — volume stuck in Pending, pod not starting, data inaccessible — the debugging path runs through PVC → PV → StorageClass.

If you don't know this model cold, you will not know where to start debugging.

---

## Learning Objectives

After completing this module, you will be able to:

1. Explain the PV/PVC separation and why it exists
2. Trace the full lifecycle: PVC → StorageClass → Provisioner → PV → Pod Mount
3. Distinguish static vs dynamic provisioning
4. Explain all access modes and when each applies
5. Explain all reclaim policies and their production implications
6. Debug PVC stuck in Pending, PV stuck in Released, pod stuck in ContainerCreating
7. Read StorageClass parameters and explain what each does

---

## Prerequisites

- Module 03 — Why Containers Need Persistent Storage
- Access to a Kubernetes cluster (Kind, Minikube, or cloud)

---

## Concepts to Study

### 1. The Abstraction Model

Kubernetes separates storage into three objects:

```
StorageClass      ← HOW to provision (which provisioner, which parameters)
     ↓
PVC               ← WHAT the app needs (size, access mode)
     ↓
PV                ← WHAT exists (actual storage resource)
```

**Why the separation?**

- **PV** is cluster-scoped — admins manage it (or provisioners create it)
- **PVC** is namespace-scoped — developers request it (they don't need to know about backends)
- **StorageClass** is cluster-scoped — defines provisioning policy

This separation is the same principle as Kubernetes nodes vs pods: admins manage nodes, developers manage pods. Admins manage PVs, developers manage PVCs.

---

### 2. Persistent Volume (PV)

A PV is a cluster resource representing a piece of storage:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: portworx-sc
  hostPath:                        # or csi, nfs, awsElasticBlockStore, etc.
    path: /data/pv-example
```

**PV Phases**:

```
Available   → PV exists, not yet bound to any PVC
Bound       → PV is bound to a PVC
Released    → PVC was deleted, PV still exists, data still there
Failed      → PV reclaim failed (provisioner error)
```

**PV is cluster-scoped** — no namespace. One PV can only be bound to one PVC at a time.

---

### 3. Persistent Volume Claim (PVC)

A PVC is a namespace-scoped request for storage:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: portworx-sc
```

**PVC Phases**:

```
Pending   → waiting for PV to be bound (provisioner running, or no matching PV)
Bound     → PVC is bound to a PV
Lost      → bound PV was deleted (data may be gone)
```

**PVC is namespace-scoped** — must be in the same namespace as the pod using it.

---

### 4. Access Modes

```
ReadWriteOnce (RWO)
  → One node can mount the volume read-write
  → All pods on that node can use it
  → Most block storage (EBS, Portworx, etc.)

ReadWriteMany (RWX)
  → Multiple nodes can mount read-write simultaneously
  → Requires shared filesystem (NFS, CephFS, Portworx SharedV4)
  → Used for: shared web assets, ML training datasets across workers

ReadOnlyMany (ROX)
  → Multiple nodes can mount read-only
  → Used for: shared configuration, read-only reference data

ReadWriteOncePod (RWOP)   [Kubernetes 1.22+]
  → Exactly one pod (not just one node) can mount read-write
  → Stricter than RWO
```

**Common mistake**: Thinking RWO means only one pod. RWO means only one **node**. Multiple pods on the same node can share an RWO volume.

**Portworx note**:
- `ReadWriteOnce` → standard Portworx volume
- `ReadWriteMany` → Portworx SharedV4 volume (NFS-based sharing layer)

---

### 5. Reclaim Policies

What happens to the PV when the PVC is deleted?

```
Retain    → PV stays, data stays, PV moves to Released state
            Admin must manually reclaim or delete
            Safe for production (no accidental data loss)

Delete    → PV and underlying storage are deleted automatically
            Convenient for dev/test, dangerous in production
            Default for dynamically provisioned PVs

Recycle   → Deprecated. Was: delete files, make PV Available again
            Do not use.
```

**Production rule**: Use `Retain` for any stateful production workload. Use `Delete` only in dev/test environments.

**Portworx default**: `Delete` — meaning when you delete a PVC, the Portworx volume is also deleted. Change to `Retain` in your StorageClass for production.

```yaml
# Production StorageClass with Retain
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: portworx-db-prod
provisioner: pxd.portworx.com
parameters:
  repl: "3"
  io_profile: db
reclaimPolicy: Retain           # ← protect production data
allowVolumeExpansion: true
```

---

### 6. StorageClass

StorageClass defines the provisioning recipe:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: portworx-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # ← default SC
provisioner: pxd.portworx.com       # ← which CSI driver to use
parameters:
  repl: "3"                         # ← Portworx-specific: replica count
  io_profile: "db"                  # ← Portworx-specific: I/O profile
  group: "mysql-vg"                 # ← Portworx-specific: volume group
  priority_io: "high"               # ← Portworx-specific: QoS priority
reclaimPolicy: Delete
allowVolumeExpansion: true          # ← allow PVC resize
volumeBindingMode: WaitForFirstConsumer  # ← don't create PV until pod scheduled
```

**`volumeBindingMode`**:

```
Immediate             → PV created as soon as PVC is created
                        Volume may be in wrong zone for the pod
                        (problem in multi-zone clusters)

WaitForFirstConsumer  → PV created only when pod is scheduled
                        Respects pod topology (zone, node affinity)
                        Recommended for production
```

**Portworx note**: Always use `WaitForFirstConsumer` with Portworx. This ensures the Portworx volume is created on a node where the pod can actually run.

---

### 7. Dynamic Provisioning Lifecycle

When `volumeBindingMode: Immediate` (or pod is scheduled with `WaitForFirstConsumer`):

```
Developer creates PVC
        ↓
kube-controller-manager detects unbound PVC
        ↓
Finds matching StorageClass
        ↓
Calls StorageClass provisioner (via CSI: CreateVolume RPC)
        ↓
Provisioner creates storage on backend (e.g., creates Portworx volume)
        ↓
Provisioner creates PV object in Kubernetes, bound to PVC
        ↓
PVC transitions: Pending → Bound
        ↓
Pod can now be scheduled and started
        ↓
kubelet mounts volume into pod
```

**Static Provisioning** (for pre-existing storage):

```
Admin creates storage manually (e.g., existing EBS volume, existing Portworx volume)
        ↓
Admin creates PV with storageClassName: ""
        ↓
Developer creates PVC with matching size, access mode, storageClassName: ""
        ↓
Kubernetes binds PVC to PV (matching algorithm)
        ↓
No provisioner involved — no dynamic creation
```

---

### 8. PVC-PV Binding Algorithm

When is a PVC bound to a PV?

```
1. StorageClass matches (both must have same storageClassName)
2. Access modes match (PVC's mode must be in PV's accessModes)
3. Capacity: PV.capacity >= PVC.requests.storage
4. PV must be in Available state
5. PVC selectors must match PV labels (if selector specified)
```

**Important**: Kubernetes may bind a PVC to a larger PV if no exact match exists. A PVC for 1Gi can bind to a 100Gi PV. The 99Gi is wasted unless the PVC is resized.

---

## Hands-On Labs

### Lab 4.1 — Static Provisioning

```bash
kind create cluster

# Create a PV manually (using hostPath for simplicity)
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-static-001
  labels:
    tier: database
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /data/pv-static-001
    type: DirectoryOrCreate
EOF

# Check PV state
kubectl get pv
# STATUS should be: Available

# Create PVC that claims it
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  selector:
    matchLabels:
      tier: database
EOF

# Check binding
kubectl get pv,pvc
# PV STATUS: Bound
# PVC STATUS: Bound

# Deploy a pod using the PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-pvc
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo 'Persistent data' > /data/test.txt && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
EOF

kubectl wait --for=condition=Ready pod/app-with-storage --timeout=60s

# Write and verify
kubectl exec app-with-storage -- cat /data/test.txt

# Delete pod — data should survive
kubectl delete pod app-with-storage

# Check PV/PVC are still Bound
kubectl get pv,pvc

# Recreate pod — verify data persists
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-pvc
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "cat /data/test.txt && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
EOF

kubectl wait --for=condition=Ready pod/app-with-storage --timeout=60s
kubectl exec app-with-storage -- cat /data/test.txt  # Data persists!

# Cleanup
kubectl delete pod app-with-storage
kubectl delete pvc app-pvc
kubectl get pv  # PV is Released (not deleted) because reclaimPolicy: Retain
```

### Lab 4.2 — Dynamic Provisioning with StorageClass

```bash
# Use the default StorageClass (Kind includes rancher.io/local-path)
kubectl get storageclass

# Create PVC — provisioner will automatically create PV
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Watch PV get created automatically
watch kubectl get pv,pvc

# Create pod to use it
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-app
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: dynamic-pvc
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo 'Dynamic PV!' > /data/test.txt && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
EOF

kubectl wait --for=condition=Ready pod/dynamic-app --timeout=60s
kubectl exec dynamic-app -- cat /data/test.txt

# Cleanup — with reclaimPolicy: Delete, PV is deleted with PVC
kubectl delete pod dynamic-app
kubectl delete pvc dynamic-pvc
kubectl get pv  # PV should be gone

kind delete cluster
```

### Lab 4.3 — Debug PVC Stuck in Pending

```bash
kind create cluster

# Create a PVC requesting a non-existent StorageClass
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: broken-pvc
spec:
  storageClassName: non-existent-class
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF

# Observe Pending state
kubectl get pvc broken-pvc

# Step 1: Describe the PVC
kubectl describe pvc broken-pvc
# Look for Events section — why is it pending?

# Step 2: Check StorageClasses
kubectl get storageclass

# Step 3: Check events
kubectl get events --sort-by='.lastTimestamp' | grep -i pvc

# Fix: update PVC to use valid StorageClass
kubectl delete pvc broken-pvc
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: broken-pvc
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pvc broken-pvc  # should become Bound

# Cleanup
kubectl delete pvc broken-pvc
kind delete cluster
```

---

## Debugging Reference

### PVC Stuck in Pending

```bash
# Step 1: Describe PVC — read the Events section
kubectl describe pvc <pvc-name> -n <namespace>

# Common causes:
# a) No matching StorageClass
kubectl get storageclass

# b) No matching PV (static provisioning)
kubectl get pv

# c) Provisioner not running (dynamic provisioning)
kubectl get pods -n kube-system | grep csi
kubectl logs -n kube-system <csi-controller-pod>

# d) Topology constraints (WaitForFirstConsumer)
# PVC stays Pending until pod is scheduled — this is normal
kubectl get pod -o wide   # is pod scheduled?
```

### PV Stuck in Released

```bash
# PV was previously bound, PVC deleted, PV not Available
kubectl get pv
# STATUS: Released

# For Retain policy — must manually reclaim:
# Option 1: Delete PV and re-create (data is on storage backend)
kubectl delete pv <pv-name>

# Option 2: Remove the claimRef to make PV Available again
kubectl patch pv <pv-name> -p '{"spec":{"claimRef": null}}'
```

### Pod Stuck in ContainerCreating (volume mount issue)

```bash
# Describe pod — read Events
kubectl describe pod <pod-name> -n <namespace>

# Common events:
# "Unable to attach or mount volumes"
# "Timeout expired waiting for volumes to attach"
# "failed to sync ... Volume is already exclusively attached"

# Check if PVC is Bound
kubectl get pvc -n <namespace>

# Check volume attachment
kubectl get volumeattachment

# Check CSI node plugin logs
kubectl logs -n kube-system <csi-node-pod> -c <csi-container>
```

---

## Interview Questions

1. What is the difference between a PV and a PVC? Why does Kubernetes separate them?
2. A PVC is in Pending state. Walk me through how you debug it.
3. What is the difference between `Retain` and `Delete` reclaim policies? Which do you use in production and why?
4. What is `WaitForFirstConsumer` and why is it important in multi-zone clusters?
5. A developer deleted a PVC by mistake. The reclaimPolicy is Retain. Is the data gone?
6. What is the difference between `ReadWriteOnce` and `ReadWriteOncePod`?
7. Can two pods in different namespaces share the same PVC?
8. What happens to StatefulSet PVCs when you `kubectl delete statefulset`?

---

## Production Takeaways

1. **Always use `reclaimPolicy: Retain` for databases** — a `kubectl delete pvc` by mistake with `Delete` policy means data is gone. No undo.

2. **`WaitForFirstConsumer` prevents zone mismatches** — in multi-zone clusters, `Immediate` binding can create a PV in zone A but schedule the pod in zone B.

3. **PVC size is a minimum, not an exact match** — a PVC for 5Gi may bind to a 100Gi PV. Use `selector: matchLabels` in static provisioning to prevent this.

4. **StatefulSet PVCs are not deleted with the StatefulSet** — this is intentional. Protect against accidental deletion. You must delete PVCs manually after verifying data is backed up.

5. **VolumeExpansion must be enabled in StorageClass** — add `allowVolumeExpansion: true` to all production StorageClasses. Portworx supports online resize.

6. **Lost PVC means gone data** — if a PVC shows as `Lost` (bound PV was deleted), the data is likely gone. This is why monitoring PV/PVC state matters.

---

## Success Criteria

You have completed this module when you can:

- [ ] Draw the PVC → StorageClass → Provisioner → PV → Pod flow from memory
- [ ] Create a static PV and bind a PVC to it (Lab 4.1)
- [ ] Create a dynamic PVC and explain what the provisioner does (Lab 4.2)
- [ ] Debug a PVC stuck in Pending (Lab 4.3)
- [ ] Explain all four access modes and when to use each
- [ ] Explain `Retain` vs `Delete` reclaim policy and choose correctly for any scenario
- [ ] Explain `WaitForFirstConsumer` and why it matters

---

## Next Module

[Module 05 — CSI Architecture](./module-05-csi-architecture.md)
