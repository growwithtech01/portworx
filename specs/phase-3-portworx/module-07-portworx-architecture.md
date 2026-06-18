# Module 07 — Portworx Architecture

```
Phase       : 3 — Portworx
Module      : 07
Status      : draft
Prereqs     : Module 06 — Stateful Applications
Estimated   : 5–6 hours
```

---

## Why This Module Exists

This module is the conceptual center of the entire course. Every component you have studied — block storage, Linux Device Mapper, Kubernetes PV/PVC, CSI — converges here into Portworx.

After this module, when you look at a Portworx cluster, you will not see a black box. You will see a distributed storage system with a control plane, data plane, CSI interface, and operator — and you will know exactly what each component does when something fails.

---

## Learning Objectives

After completing this module, you will be able to:

1. Describe the Portworx architecture stack from workload to disk
2. Explain the role of every Portworx component: Operator, PVC Controller, KVDB, Stork, API, data plane
3. Trace the full write path: Application → PVC → Portworx → Local Disk + Remote Replica
4. Trace the full failure recovery path: Disk failure → Detection → Rebuild
5. Explain what KVDB is and why it matters for quorum
6. Explain Stork and what storage-aware scheduling means
7. Identify which component to check first for any given failure symptom

---

## Prerequisites

- All Phase 1 and Phase 2 modules
- Portworx-licensed cluster or Portworx Essentials (free tier) is helpful but not required for this module

---

## Concepts to Study

### 1. The Full Architecture Stack

```
┌──────────────────────────────────────────────────────┐
│  WORKLOADS (Pods, StatefulSets, Deployments)         │
├──────────────────────────────────────────────────────┤
│  PVC (PersistentVolumeClaim)                         │
├──────────────────────────────────────────────────────┤
│  StorageClass  →  PV (PersistentVolume)              │
├──────────────────────────────────────────────────────┤
│  CSI LAYER                                           │
│  ├─ CSI Controller (CreateVolume, AttachVolume)      │
│  └─ CSI Node Plugin (MountVolume)                    │
├──────────────────────────────────────────────────────┤
│  PORTWORX CONTROL PLANE                              │
│  ├─ Portworx Operator (lifecycle management)         │
│  ├─ Portworx PVC Controller (volume claims)          │
│  ├─ KVDB (cluster metadata store)                    │
│  ├─ Stork (storage-aware scheduler)                  │
│  └─ Portworx API (pxctl, REST API)                   │
├──────────────────────────────────────────────────────┤
│  PORTWORX DATA PLANE                                 │
│  ├─ Volume Manager                                   │
│  ├─ Replication Engine                               │
│  ├─ I/O Path (read/write routing)                    │
│  └─ Journal (write ordering, crash recovery)         │
├──────────────────────────────────────────────────────┤
│  LOCAL DISKS (NVMe, SSD, HDD)                        │
└──────────────────────────────────────────────────────┘
```

Each layer has distinct responsibilities. Failures in different layers produce different symptoms.

---

### 2. Portworx Operator

The Portworx Operator (also called StorageCluster Operator or PX-Operator) is a Kubernetes Operator that manages the Portworx cluster lifecycle:

```
Responsibilities:
├─ Install Portworx (create DaemonSet, RBAC, services)
├─ Upgrade Portworx (rolling upgrade of PX pods)
├─ Monitor PX pod health (restart failed pods)
├─ Manage configuration changes
└─ Report cluster status via StorageCluster CRD
```

**Key CRD**: `StorageCluster`

```yaml
apiVersion: core.libopenstorage.org/v1
kind: StorageCluster
metadata:
  name: portworx
  namespace: kube-system
spec:
  image: portworx/oci-monitor:3.x.x
  kvdb:
    internal: true              # use built-in KVDB
  storage:
    useAll: true                # use all available disks
    journalDevice: auto         # auto-detect NVMe for journal
  network:
    dataInterface: eth0
    mgmtInterface: eth0
  stork:
    enabled: true
  autopilot:
    enabled: true
```

```bash
# Check Portworx Operator
kubectl get pod -n kube-system | grep portworx-operator

# Check cluster status via CRD
kubectl get storagecluster -n kube-system
kubectl describe storagecluster portworx -n kube-system
```

**Why the Operator pattern?** Portworx is complex stateful infrastructure. Managing it manually (editing DaemonSets, handling upgrades) is error-prone. The Operator encodes operational knowledge: how to upgrade safely, how to handle node additions, how to recover from failed upgrades.

---

### 3. Portworx DaemonSet (PX Pod)

The core Portworx process runs as a DaemonSet — one pod per node:

```
portworx-<nodename> pod
├─ portworx container      ← the main PX daemon
│   ├─ CSI controller service (on one elected node)
│   ├─ CSI node service (on all nodes)
│   ├─ Data plane: I/O path, replication engine, journal
│   └─ Portworx API server
├─ node-driver-registrar   ← CSI registration with kubelet
└─ liveness-probe          ← health check
```

```bash
# List all PX pods
kubectl get pod -n kube-system -l name=portworx -o wide

# Check a specific PX pod
kubectl describe pod portworx-<nodename> -n kube-system

# PX daemon logs
kubectl logs -n kube-system portworx-<nodename> -c portworx

# SSH into a node and check PX status
pxctl status
```

---

### 4. KVDB — The Cluster Brain

KVDB (Key-Value DataBase) is Portworx's distributed metadata store. Portworx uses an embedded etcd by default (called "internal KVDB") or can use an external etcd.

```
KVDB stores:
├─ Volume metadata (which volumes exist, their size, replication factor)
├─ Node membership (which nodes are in the cluster)
├─ Volume placement (which replica is on which node)
├─ Cluster configuration
└─ Alert history
```

**KVDB quorum**:

```
KVDB nodes = 3 (default for clusters with 3+ nodes)
Quorum = majority = 2

If 2 of 3 KVDB nodes are alive → cluster operational
If 2 of 3 KVDB nodes are down → cluster read-only / degraded
```

**Why KVDB matters during incidents**:
- If KVDB is unhealthy, Portworx cannot create/delete volumes
- Volume metadata lives in KVDB — a corrupt KVDB can make volumes appear "missing"
- KVDB split-brain causes cluster partition

```bash
# Check KVDB health
pxctl service kvdb status
pxctl service kvdb members

# Check KVDB logs
kubectl logs -n kube-system portworx-<nodename> -c portworx | grep kvdb
```

**Internal vs External KVDB**:

| | Internal KVDB | External etcd |
|--|--------------|---------------|
| Setup | Automatic | Manual |
| Scaling | Limited | Unlimited |
| Risk | KVDB co-located with PX | Separate failure domain |
| Production | Acceptable for small clusters | Recommended for large |

---

### 5. Portworx PVC Controller

The Portworx PVC Controller watches for PVCs with Portworx StorageClasses and handles provisioning:

```
PVC created with storageClassName: portworx-sc
        ↓
PVC Controller detects unbound PVC
        ↓
Calls Portworx API: create volume with specified parameters
        ↓
Portworx data plane creates volume on storage pool
        ↓
PVC Controller creates PV object
        ↓
Binds PVC to PV
```

This is the CSI external-provisioner role, but Portworx also has its own PVC controller for non-CSI paths and legacy compatibility.

```bash
# Check PVC controller logs
kubectl logs -n kube-system portworx-<nodename> -c portworx | grep pvc-controller
```

---

### 6. Stork — Storage-Aware Scheduling

Stork (Storage Orchestration Runtime for Kubernetes) is a scheduler extender that makes Kubernetes volume-placement aware:

**Problem without Stork**:

```
Kubernetes scheduler places pod on node3
But the Portworx volume's primary replica is on node1
Result: I/O must traverse the network (node3 → node1 for every read)
Latency increases dramatically
```

**With Stork**:

```
Stork watches pod scheduling decisions
Stork knows which node has the volume's primary replica
Stork hints to Kubernetes: "prefer node1 for this pod"
Result: pod runs on node1 (hyperconverged = local I/O)
Latency is local disk speed, not network speed
```

**Additional Stork capabilities**:

```
Volume Health Monitoring  → detects unhealthy volumes, triggers pod rescheduling
Snapshot scheduling       → create snapshots on schedule
Migration (STORK CRDs)    → migrate workloads between clusters (Metro DR)
Application clone         → clone full application (app + PVCs)
```

```bash
# Check Stork status
kubectl get pod -n kube-system -l name=stork

# Stork scheduler name
kubectl get pod -n kube-system -l name=stork -o jsonpath='{.items[0].spec.schedulerName}'
# output: stork

# Use Stork scheduler for a pod
spec:
  schedulerName: stork      # ← use stork-aware scheduling
```

---

### 7. Portworx API

Portworx exposes a gRPC and REST API for all storage operations:

```
Portworx API covers:
├─ Volume CRUD (create, read, update, delete)
├─ Snapshot operations
├─ Cloud backup operations
├─ Node management
├─ Storage pool management
├─ Alert management
└─ Cluster configuration
```

**CLI**: `pxctl` — the Portworx command-line tool that wraps the REST API

```bash
# Volume operations
pxctl volume list
pxctl volume inspect <volume-id>
pxctl volume create --size=10 --repl=3 myvol

# Node operations
pxctl status
pxctl node list

# Storage pool operations
pxctl service pool show

# Alert viewing
pxctl alerts show
```

**Portworx Lighthouse / PX-Central**: Web UI that wraps the same API.

---

### 8. The Data Plane — How Writes Actually Work

This is the most important part for understanding Portworx performance and reliability.

**Volume internals**:

```
A Portworx volume is:
├─ A virtual block device (appears as /dev/pxd/* to the OS)
├─ Backed by one or more storage pools (local disks)
└─ Replicated across N nodes (replication factor)
```

**Write path with replication factor 3**:

```
Application writes to /var/lib/mysql/data
        ↓
Write goes to /dev/pxd/pxd<volume-id>  (virtual PX block device)
        ↓
Portworx data plane intercepts the write
        ↓
Portworx writes to local replica (current node)
        ↓
Portworx simultaneously replicates to 2 remote nodes
        ↓
Wait for acknowledgment from quorum (2 of 3 replicas)
        ↓
Acknowledge write to application
```

**Synchronous replication**:
- Write is not acknowledged until quorum is satisfied
- If remote replica is slow or down, the write is blocked until timeout
- This is why network latency between Portworx nodes directly impacts application write latency

**Journal**:
- Portworx maintains a write-ahead journal on a dedicated fast disk (NVMe)
- Journal ensures write ordering across replicas
- On node crash, journal is replayed to recover consistent state

---

### 9. Storage Pools

A storage pool is a collection of disks on one node that Portworx manages as a single pool:

```
Node 1:
├─ /dev/sda (SSD 500GB)
├─ /dev/sdb (SSD 500GB)
└─ /dev/nvme0n1 (NVMe 200GB) → journal disk
     ↓
Storage Pool: 1000GB total capacity
```

Multiple pools can exist per node (e.g., fast pool on NVMe, slow pool on HDD):

```bash
# View storage pools
pxctl service pool show

# Output:
# Pool  0 Medium    RAID_1  online  1.0 TB  online  default,journal
#   IO_Priority      : HIGH
#   Labels           : ...
#   UUID             : ...
```

---

### 10. Telemetry

Portworx ships telemetry that sends cluster health metrics to Pure Storage:

```
Telemetry collects:
├─ Volume health metrics
├─ Node health metrics
├─ IOPS and throughput data
├─ Error rates
└─ Capacity utilization
```

Telemetry enables:
- Proactive support (Pure can alert you before you notice issues)
- Automatic diag collection during support cases
- License tracking

```bash
# Check telemetry status
pxctl service diags --status

# Check telemetry pod
kubectl get pod -n kube-system | grep telemetry
```

---

### 11. Complete Request Flow

**PVC Creation → Application Write → Disk**:

```
1. Developer: kubectl apply -f pvc.yaml
        ↓
2. PVC Controller detects new PVC
3. Calls Portworx API: CreateVolume(size=10Gi, repl=3, ioProfile=db)
        ↓
4. Portworx API layer creates volume in KVDB:
   - assigns volume ID
   - selects 3 nodes for replicas (using placement rules)
   - stores metadata in KVDB
        ↓
5. Portworx data plane on each selected node:
   - allocates blocks from storage pool
   - creates /dev/pxd/pxd<volume-id> virtual device
        ↓
6. PVC Controller creates PV object in Kubernetes
7. PVC bound to PV
        ↓
8. Pod scheduled (Stork ensures pod lands on node with primary replica)
        ↓
9. CSI NodeStageVolume: mkfs.xfs on /dev/pxd/pxd<id>, mount at staging path
10. CSI NodePublishVolume: bind mount to pod directory
        ↓
11. Application writes to /var/lib/mysql
12. Portworx data plane: local write + replicate to 2 other nodes
13. Quorum ack → application write confirmed
```

**Disk Failure → Recovery**:

```
1. Disk on node2 fails (physical failure)
        ↓
2. Portworx on node2 detects I/O errors on /dev/sdb
3. Marks replica on node2 as degraded
4. Updates KVDB: volume <id> replica node2 = degraded
        ↓
5. Alert generated: "Volume <id> degraded, replica on node2 lost"
        ↓
6. Portworx control plane: volume is below replication factor
7. Selects a healthy node (node4) to host new replica
        ↓
8. Rebuild begins: copies data from node1 replica to node4
   (bandwidth throttled to not impact running workloads)
        ↓
9. Rebuild complete: KVDB updated, node4 is new replica
10. Volume back to full replication factor
11. Alert: "Volume <id> fully replicated"
```

---

## Exercises

### Exercise 1 — Component → Failure Mapping

For each symptom, identify which Portworx component to check first:

| Symptom | First Component to Check |
|---------|--------------------------|
| PVC stuck in Pending | |
| Pod scheduled on wrong node (not local to volume) | |
| Volume creation fails with API error | |
| Node removed from cluster, volumes not rebuilding | |
| Cluster cannot accept writes | |
| Volume shows degraded for 2 days | |
| pxctl commands return timeout | |

### Exercise 2 — Write Path Tracing

A MySQL pod is running on node1. Portworx replication factor is 3. Replicas are on node1, node2, node3. Trace exactly what happens (which components, which calls, which logs) when:

1. MySQL writes a 4KB row
2. node2 loses network connectivity
3. MySQL writes another 4KB row
4. node2 comes back online

---

## Interview Questions

1. Explain the Portworx architecture in 3 minutes — components, their roles, how they connect.
2. What is KVDB? What happens to the cluster if KVDB loses quorum?
3. What does Stork do that Kubernetes scheduler doesn't?
4. A Portworx volume has replication factor 3. One node dies. Walk me through what Portworx does automatically.
5. What is a storage pool? How does Portworx use multiple disks on a node?
6. Why does Portworx recommend a separate NVMe disk for the journal?
7. What is the difference between the Portworx control plane and data plane?
8. How does the Portworx Operator differ from just running Portworx as a DaemonSet?

---

## Production Takeaways

1. **KVDB is the most critical component** — monitor it constantly. KVDB quorum loss = cluster degraded. Keep 3 KVDB members minimum.

2. **Stork scheduler is not optional for databases** — without Stork, your database pods may not co-locate with their volume replicas. Every read becomes a network I/O.

3. **Replication factor does not protect against KVDB loss** — you can have 3 data replicas but if KVDB is gone, Portworx cannot manage them.

4. **Journal disk wear is a real concern** — NVMe journal disks see very high write volumes. Monitor S.M.A.R.T. data on journal disks.

5. **Storage pools must be on dedicated disks** — do not share disks between Portworx and the OS. OS disk fills up → Portworx cannot write journal.

6. **The Operator controls PX upgrades** — never manually edit the PX DaemonSet. Use the StorageCluster CRD. The Operator upgrades one node at a time.

---

## Success Criteria

You have completed this module when you can:

- [ ] Draw the full Portworx architecture stack from memory (workload to disk)
- [ ] Explain every component's role in one sentence
- [ ] Trace the complete write path for a database write with replication factor 3
- [ ] Trace the disk failure → rebuild recovery path
- [ ] Explain KVDB quorum and what happens when it is lost
- [ ] Explain what Stork does and why it matters for latency
- [ ] Complete Exercise 1 (component → failure mapping) without guessing

---

## Next Module

[Module 08 — Portworx Installation](./module-08-portworx-installation.md)
