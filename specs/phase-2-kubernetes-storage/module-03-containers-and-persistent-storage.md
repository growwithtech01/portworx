# Module 03 — Why Containers Need Persistent Storage

```
Phase       : 2 — Kubernetes Storage
Module      : 03
Status      : draft
Prereqs     : Module 02 — Linux Storage
Estimated   : 3–4 hours
```

---

## Why This Module Exists

Containers were designed to be ephemeral. Everything on the container filesystem is lost when the container dies. This is a feature — not a bug — for stateless apps. But stateful apps (databases, message queues, caches with persistence) need data to outlive the container.

This module explains **why** — not just what commands to run. Every PVC and PV you create later is solving the problem defined in this module.

---

## Learning Objectives

After completing this module, you will be able to:

1. Explain what the container filesystem is and why it is ephemeral
2. Describe the overlay filesystem and how layers work
3. Explain why pod restarts lose data without persistent storage
4. Compare `emptyDir`, `hostPath`, and persistent volumes — pick the right one
5. Describe the Kubernetes volume lifecycle
6. Trace what happens to data when a pod is deleted vs rescheduled vs node dies

---

## Prerequisites

- Module 01 — Storage From First Principles
- Module 02 — Linux Storage
- Basic Kubernetes familiarity (pods, deployments, namespaces)

---

## Concepts to Study

### 1. The Container Filesystem

A container's filesystem is **not a real disk**. It is a union of layers stacked on top of each other using the Linux overlay filesystem:

```
Layer 4: Container write layer    ← Read/Write (ephemeral)
Layer 3: App layer (FROM image)   ← Read Only
Layer 2: Base OS layer            ← Read Only
Layer 1: Scratch layer            ← Read Only
```

When a container writes a file, it goes into the **write layer** (Layer 4). This layer exists only in memory/on the host's local disk — it is tied to that specific container instance.

When the container dies:
- The write layer is destroyed
- All writes are gone
- The next container starts with a fresh, empty write layer

```bash
# See overlay mounts on a running container
mount | grep overlay

# Typical output:
# overlay on /var/lib/docker/overlay2/abc123/merged type overlay
#   (rw,lowerdir=...,upperdir=...,workdir=...)
#
# upperdir = the write layer (ephemeral)
# lowerdir = the read-only image layers
```

---

### 2. The Overlay Filesystem

The overlay filesystem is a Linux kernel feature (OverlayFS) that merges multiple directories into a single mount:

```
Container sees:
/etc/nginx.conf    ← from image layer (read-only)
/var/log/nginx/    ← write layer (read-write)

Internally:
lowerdir = image layers (stacked, read-only)
upperdir = container's write layer
merged   = union of lowerdir + upperdir
```

**Copy-on-Write (CoW)**:
- When a container modifies a read-only file, OverlayFS copies it to `upperdir` first
- All subsequent writes go to `upperdir`
- Reads from `upperdir` shadow the original in `lowerdir`

**Performance implication**:
- First write to any file in the image = CoW copy (can be slow for large files)
- Subsequent writes = fast (already in `upperdir`)
- This is why databases should never use the container overlay filesystem for data

---

### 3. What Happens When a Pod Restarts

```
Pod scheduled → Container starts → Write layer created (empty)
                      ↓
              Application writes /data/db/*
                      ↓
              Pod crashes (OOM, error, eviction)
                      ↓
              Container dies → Write layer DESTROYED
                      ↓
              Kubernetes restarts container
                      ↓
              New container starts → NEW empty write layer
                      ↓
              /data/db/* is GONE
```

This is why running MySQL, PostgreSQL, MongoDB, etcd without a PVC results in data loss on every pod restart.

**Kubernetes restarts vs reschedules**:

| Event | Pod stays on same node? | Data survives (no PV)? |
|-------|------------------------|----------------------|
| Container crash + restart | Yes | No — write layer recreated |
| Pod deleted and recreated | Maybe | No |
| Node failure, pod rescheduled | No (different node) | No |

---

### 4. Volume Types in Kubernetes

#### emptyDir

```yaml
volumes:
  - name: cache
    emptyDir: {}
```

```
Created when Pod is assigned to a Node
Deleted when Pod is removed from Node
Shared between all containers in the Pod
Lives on the Node's local disk (or RAM if medium: Memory)
```

**Use cases**:
- Shared scratch space between containers in a pod (sidecar pattern)
- Cache that can be rebuilt (not source of truth)
- Temporary files during processing

**Not for**: persistent data, database files, anything that must survive pod deletion

```bash
# emptyDir backed by RAM (tmpfs — very fast, limited to node RAM)
volumes:
  - name: ramdisk
    emptyDir:
      medium: Memory
      sizeLimit: 512Mi
```

#### hostPath

```yaml
volumes:
  - name: host-data
    hostPath:
      path: /data/myapp
      type: DirectoryOrCreate
```

```
Mounts a specific path from the HOST node's filesystem
Data persists across pod restarts (as long as pod stays on same node)
Data is NOT portable — tied to specific node
```

**Use cases**:
- DaemonSets that need to read node-level resources (`/proc`, `/sys`, cgroups)
- Single-node development clusters
- Log collection agents reading `/var/log`

**Not for**: production stateful applications — pod rescheduled to different node = data inaccessible

**Security warning**: `hostPath` gives the pod access to the host filesystem. A misconfigured `hostPath: /` gives full host access. Never use in multi-tenant clusters.

#### Persistent Volumes (PVC/PV)

```yaml
volumes:
  - name: database-data
    persistentVolumeClaim:
      claimName: mysql-pvc
```

```
Data persists across pod restarts
Data is portable — pod can reschedule to any node
Backed by real storage (cloud disk, NFS, Portworx, etc.)
Lifecycle independent of pod
```

**This is what we use for everything stateful in production.**

---

### 5. The Kubernetes Volume Lifecycle

```
Pod spec references volume name
        ↓
kubelet identifies the volume source
        ↓
If PVC: look up PVC → look up bound PV → determine storage backend
        ↓
kubelet calls CSI node plugin: NodeStageVolume
(attach disk to node and format/mount at staging path)
        ↓
kubelet calls CSI node plugin: NodePublishVolume
(bind mount from staging path into pod's /var/lib/kubelet/pods/<uid>/volumes/)
        ↓
Container starts with volume mounted at specified mountPath
```

**On pod deletion**:

```
Container stops
        ↓
kubelet calls CSI: NodeUnpublishVolume (remove bind mount from pod dir)
        ↓
kubelet calls CSI: NodeUnstageVolume (unmount from staging path)
        ↓
CSI Controller: ControllerUnpublishVolume (detach disk from node)
        ↓
PV remains, PVC remains, data is safe on storage backend
```

---

### 6. Container Storage vs Persistent Storage

```
Container Storage:                  Persistent Storage:
─────────────────                   ──────────────────
OverlayFS (CoW)                     Real filesystem (XFS/ext4)
Tied to container                   Independent of container
Fast for reads (cached layers)      Consistent performance
Slow for writes (CoW)               Direct block I/O
No data portability                 Node-portable (with CSI)
Deleted with container              Survives pod deletion
```

**Rule of thumb**: Any data you would be upset about losing goes in a PVC.

---

## Hands-On Labs

### Lab 3.1 — Observe Data Loss Without PVC

```bash
# Start a local cluster (Kind or Minikube)
kind create cluster

# Deploy MySQL WITHOUT a PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: mysql-no-pvc
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    - name: MYSQL_DATABASE
      value: "testdb"
EOF

# Wait for it to be running
kubectl wait --for=condition=Ready pod/mysql-no-pvc --timeout=120s

# Write data
kubectl exec mysql-no-pvc -- mysql -uroot -ppassword -e \
  "CREATE TABLE testdb.users (id INT, name VARCHAR(50)); \
   INSERT INTO testdb.users VALUES (1, 'Alice'), (2, 'Bob');"

# Verify data
kubectl exec mysql-no-pvc -- mysql -uroot -ppassword -e \
  "SELECT * FROM testdb.users;"

# Kill the container (simulate crash)
kubectl exec mysql-no-pvc -- kill 1
# Wait for restart (restartPolicy: Always by default)
sleep 10

# Try to read data again
kubectl exec mysql-no-pvc -- mysql -uroot -ppassword -e \
  "SELECT * FROM testdb.users;" 2>&1
# Expected: ERROR — table doesn't exist. Data is gone.

# Cleanup
kubectl delete pod mysql-no-pvc
```

### Lab 3.2 — Observe the Overlay Filesystem

```bash
# Run a container and observe its overlay mount
docker run -d --name overlay-test nginx:alpine

# Find the overlay mount
CONTAINER_ID=$(docker inspect overlay-test --format '{{.Id}}')
docker inspect overlay-test --format '{{.GraphDriver.Data}}'

# Access the upper (write) layer
UPPER=$(docker inspect overlay-test --format '{{.GraphDriver.Data.UpperDir}}')
ls $UPPER   # empty initially

# Write a file in the container
docker exec overlay-test sh -c "echo 'hello' > /tmp/test.txt"

# Now look at the upper layer on the host
ls $UPPER/tmp/   # test.txt appears here!

# Kill and remove the container
docker rm -f overlay-test

# Upper layer is gone
ls $UPPER  # no longer exists

# Cleanup
kind delete cluster
```

### Lab 3.3 — emptyDir Shared Between Containers

```bash
kind create cluster

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-demo
spec:
  volumes:
  - name: shared
    emptyDir: {}
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "while true; do date >> /data/log.txt; sleep 5; done"]
    volumeMounts:
    - name: shared
      mountPath: /data
  - name: reader
    image: busybox
    command: ["sh", "-c", "while true; do tail -5 /data/log.txt; sleep 5; done"]
    volumeMounts:
    - name: shared
      mountPath: /data
EOF

kubectl wait --for=condition=Ready pod/shared-volume-demo --timeout=60s

# Watch reader container see writer's output
kubectl logs shared-volume-demo -c reader -f &

# After 30 seconds, delete the pod
sleep 30
kubectl delete pod shared-volume-demo

# Recreate and check: log.txt is gone
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-demo
spec:
  volumes:
  - name: shared
    emptyDir: {}
  containers:
  - name: reader
    image: busybox
    command: ["sh", "-c", "ls /data/"]
    volumeMounts:
    - name: shared
      mountPath: /data
EOF

kubectl logs shared-volume-demo -c reader
# Output: (empty) — data is gone with emptyDir

# Cleanup
kubectl delete pod shared-volume-demo
kind delete cluster
```

---

## Exercises

### Exercise 1 — Volume Type Selection

For each scenario, choose the right volume type and explain why:

| Scenario | Volume Type | Reason |
|----------|-------------|--------|
| PostgreSQL primary in production | | |
| Nginx temp upload directory | | |
| Log shipping sidecar reading /var/log/pods | | |
| Redis with AOF persistence enabled | | |
| Prometheus metrics (ephemeral, rebuilt on restart) | | |
| Shared config between app container and config-reload sidecar | | |
| etcd cluster member data | | |

### Exercise 2 — Data Survival Matrix

Complete this matrix:

| Event | emptyDir | hostPath | PVC (same node) | PVC (Portworx) |
|-------|----------|----------|-----------------|----------------|
| Container crash + restart | | | | |
| Pod deleted + recreated same node | | | | |
| Pod deleted + recreated different node | | | | |
| Node failure | | | | |
| Kubernetes upgrade | | | | |

### Exercise 3 — Overlay Filesystem Reasoning

Answer without running the commands:

1. A container reads `/etc/nginx.conf` from the image. Then it writes a new `/etc/nginx.conf`. What happens at the overlay level?
2. Two containers share the same image. Container A modifies `/etc/hosts`. Does Container B see the change?
3. A container writes 1GB of data to `/var/lib/mysql`. Where does this data actually live on the host?

---

## Interview Questions

1. Explain why a MySQL pod loses its data every time it restarts if no PVC is attached.
2. What is OverlayFS? How does copy-on-write work in containers?
3. When would you use `emptyDir` and when would you use a PVC?
4. What is the difference between a pod being restarted vs rescheduled in terms of storage?
5. A developer says "hostPath works fine, it persists across restarts." What is the flaw in this argument?
6. What happens to a PVC when the pod using it is deleted?
7. Why do databases perform poorly when run on container overlay storage without a persistent volume?
8. What is the volume lifecycle — from pod creation to pod deletion?

---

## Production Takeaways

1. **All stateful workloads need PVCs, no exceptions** — emptyDir and hostPath are not production storage solutions.

2. **`restartPolicy: Always` is the default** — Kubernetes will restart crashed containers. Without a PVC, every restart means fresh state.

3. **StatefulSets manage PVCs automatically** — when using StatefulSets (the right way to run databases), each replica gets its own PVC. The PVC is NOT deleted when the StatefulSet is scaled down.

4. **Overlay CoW slows down large file writes** — if your app writes large files into the container filesystem, it copies the whole file first. Always mount database data directories as volumes.

5. **Pod IP changes on reschedule, volume data does not** — with a properly configured PVC, the pod gets a new IP but the same data. Applications must tolerate IP changes; they should not need to tolerate data loss.

6. **`hostPath` is a cluster security risk** — in multi-tenant clusters, `hostPath` allows container escape. Audit all `hostPath` volumes in production.

---

## Success Criteria

You have completed this module when you can:

- [ ] Explain the overlay filesystem and copy-on-write without notes
- [ ] Demonstrate data loss on pod restart without PVC (Lab 3.1)
- [ ] Explain exactly when emptyDir, hostPath, and PVCs are appropriate
- [ ] Complete the Data Survival Matrix exercise without guessing
- [ ] Describe the Kubernetes volume lifecycle from pod creation to deletion
- [ ] Explain why overlay CoW hurts database write performance

---

## Next Module

[Module 04 — Persistent Volumes Deep Dive](./module-04-persistent-volumes.md)
