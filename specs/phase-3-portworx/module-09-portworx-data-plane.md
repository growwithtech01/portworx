# Module 09 — Portworx Data Plane

```
Phase       : 3 — Portworx
Module      : 09
Status      : draft
Prereqs     : Module 08 — Portworx Installation
Estimated   : 4–5 hours
```

---

## Why This Module Exists

The data plane is where Portworx earns its existence. This is the code path that runs on every write your database makes — replication, quorum, journaling, I/O routing. Understanding the data plane explains:

- Why replication factor 3 costs 3× storage but gives you resilience to 2 node failures
- Why write latency is bounded by network speed between nodes
- Why Portworx recommends hyperconvergence (pod on same node as volume replica)
- Why a journal disk reduces write latency
- What happens during a node failure — and why you care about quorum

---

## Learning Objectives

After completing this module, you will be able to:

1. Explain how Portworx exposes a virtual block device to the OS
2. Trace the write path from application syscall to disk bits on all replica nodes
3. Explain replication factor, quorum, and the tradeoff between durability and performance
4. Explain the role of the journal and why it must be on fast storage
5. Describe I/O profiles and how they tune the data plane for different workloads
6. Explain volume placement rules (NUMA, rack, zone awareness)
7. Benchmark a Portworx volume and interpret results

---

## Prerequisites

- Module 08 — Portworx Installation
- Working Portworx cluster (from Lab 8.1)

---

## Concepts to Study

### 1. The Virtual Block Device

When Portworx creates a volume, it exposes a virtual block device to the kernel:

```
pxctl volume create --size=10 myvol
        ↓
Portworx creates /dev/pxd/pxd<volume-id>
        ↓
Application or kubelet mounts /dev/pxd/pxd<volume-id>
        ↓
All I/O to this device goes through the Portworx data plane
```

**How it works**:

```
/dev/pxd/pxd<id>
        ↓
Portworx kernel module or FUSE layer intercepts all reads/writes
        ↓
Portworx decides: local write + replicate to N-1 nodes
```

Portworx uses a kernel module (`px.ko`) or user-space FUSE driver (depending on environment) to intercept block I/O at the kernel level. This is below the filesystem — Portworx sees raw blocks, not files.

```bash
# List Portworx virtual devices on a node
ls /dev/pxd/

# Show Portworx block device details
lsblk | grep pxd

# Inspect which physical pool backs a volume
pxctl volume inspect <volume-id>
# Look for: "Replica Sets on Nodes"
```

---

### 2. The Write Path — Step by Step

Assuming: Portworx volume with replication factor 3 (node1 = primary, node2, node3 = replicas)

```
Application (running on node1) writes 4KB to /var/lib/mysql/data/ibdata1
        ↓
[1] Filesystem (XFS) calls block layer write
        ↓
[2] Block layer routes to /dev/pxd/pxd<id>
        ↓
[3] Portworx kernel module intercepts write
        ↓
[4] Write goes to journal (NVMe) — fast, ordered, crash-safe
        ↓
[5] Portworx writes to local storage pool (primary replica on node1)
     AND simultaneously sends to node2 and node3
        ↓
[6] node2 receives data, writes to its local replica, sends ACK
    node3 receives data, writes to its local replica, sends ACK
        ↓
[7] Portworx waits for quorum ACKs:
    - repl=3, quorum=2: needs 2 ACKs (including local)
    - node1 local write = ACK #1
    - node2 ACK = ACK #2 → QUORUM REACHED
        ↓
[8] Write acknowledged to application
    (node3 ACK arrives async, before next write)
        ↓
[9] Journal entry marked committed
```

**Key insight**: Write latency is bounded by the SLOWEST of:
- Local disk write (NVMe → very fast)
- Network RTT to nearest replica + remote disk write

If replication traffic goes over a slow network, write latency increases proportionally.

---

### 3. Read Path

Reads are simpler:

```
Application reads from /var/lib/mysql/data/ibdata1
        ↓
Portworx checks: is pod on same node as primary replica?
        ↓
YES (hyperconverged — Stork did its job):
  Read from local disk directly
  Latency: local NVMe/SSD speed (µs range)

NO (pod is on different node):
  Read fetches data from primary replica node over network
  Latency: network RTT + remote disk read (ms range)
```

This is why Stork matters. Without Stork, reads can traverse the network unnecessarily.

**Read from replica**: Portworx can optionally read from any replica (not just primary), useful for read scaling.

---

### 4. Replication Factor

Replication factor determines how many copies of data exist across nodes:

```
repl=1  → 1 copy (no redundancy)
repl=2  → 2 copies on 2 different nodes
repl=3  → 3 copies on 3 different nodes
```

**Quorum for writes**:

```
repl=2: need 1 of 2 nodes alive for writes (quorum = ceil(repl/2))
repl=3: need 2 of 3 nodes alive for writes (quorum = 2)
```

**What replication factor means for failure tolerance**:

```
repl=1  → 0 node failures tolerated before data loss
repl=2  → 1 node failure tolerated
repl=3  → 1 node failure tolerated for writes, 2 for reads
           (2 nodes down = below quorum = reads only)
```

**Cost**:

```
10GB volume × repl=3 = 30GB consumed from storage pools
10GB volume × repl=2 = 20GB consumed
```

**StorageClass parameter**:

```yaml
parameters:
  repl: "3"
```

```bash
# Check replica placement for a volume
pxctl volume inspect <volume-id> | grep -A 10 "Replica Sets"

# Change replication factor on existing volume (online operation!)
pxctl volume ha-update --repl 2 <volume-id>   # reduce to 2
pxctl volume ha-update --repl 3 <volume-id>   # increase to 3
```

---

### 5. The Journal

The journal is a write-ahead log for the Portworx data plane:

```
Purpose:
  - Ensures write ordering across replicas
  - Provides crash recovery (replay journal on restart)
  - Reduces write latency (journal write is fast; background flush to main disk)

Location:
  - Dedicated disk (separate from data disks)
  - Must be NVMe for best performance
  - Auto-selected by Portworx if journalDevice: auto in StorageCluster

Size:
  - Typically 10–20GB (Portworx manages automatically)
```

**Why journal on separate NVMe?**

```
Without separate journal:
  write → wait for data disk (SSD: ~0.1ms per write)

With NVMe journal:
  write → journal (NVMe: ~0.02ms) → acknowledge
          data flush happens in background

Result: write latency reduced by ~5× for write-heavy workloads
```

```bash
# Check which disk is the journal
pxctl service pool show
# Look for: type=journal in pool list

# In StorageCluster spec:
storage:
  journalDevice: /dev/nvme0n1   # explicit
  # OR
  journalDevice: auto           # Portworx selects fastest available
```

---

### 6. I/O Profiles

I/O profiles tune the data plane's internal buffering, scheduling, and caching behavior for different workload types:

```
none        → no specific optimization (default)
db          → optimize for random I/O (databases: MySQL, Postgres, MongoDB)
db_remote   → optimize for databases with remote volumes (cross-zone)
sync_shared → optimize for shared volumes with synchronous writes
sequential  → optimize for sequential workloads (ETL, analytics)
auto        → Portworx selects based on observed workload
```

**StorageClass parameter**:

```yaml
parameters:
  io_profile: "db"
```

**db profile internals**:
- Increases write buffer (absorbs burst writes)
- Adjusts read-ahead size for small random reads
- Prioritizes low latency over throughput

```bash
# Check volume's I/O profile
pxctl volume inspect <volume-id> | grep "IO Profile"

# Update I/O profile on existing volume
pxctl volume update --io_profile db <volume-id>
```

---

### 7. Storage Pools and Disk Tiers

Portworx can manage multiple storage pools per node with different performance characteristics:

```
Node 1:
├─ Pool 0: HIGH priority (NVMe SSDs) — for databases
│   ├─ /dev/nvme0n1 (500GB)
│   └─ /dev/nvme1n1 (500GB)
└─ Pool 1: MEDIUM priority (SATA SSDs) — for general workloads
    ├─ /dev/sdb (2TB)
    └─ /dev/sdc (2TB)
```

```yaml
# StorageClass targeting specific priority pool
parameters:
  priority_io: "high"    # only use HIGH priority pool
  # OR
  priority_io: "medium"
  # OR
  priority_io: "low"
```

```bash
# Show all storage pools
pxctl service pool show

# Show pool utilization
pxctl service pool show --json | jq '.[] | {id: .ID, used: .Used, capacity: .TotalSize}'
```

---

### 8. Volume Placement Strategies

Portworx places replicas according to placement rules to ensure replicas are on different failure domains:

**Default placement**: Portworx ensures replicas are on different nodes (never 2 replicas on same node).

**Rack awareness**: Portworx can be told about physical rack topology:

```bash
# Label nodes with rack info
kubectl label nodes node1 node2 node3 failure-domain.beta.kubernetes.io/zone=rack-a
kubectl label nodes node4 node5 node6 failure-domain.beta.kubernetes.io/zone=rack-b
```

**Volume Placement Strategies (VPS)**:

```yaml
# Force replicas into different zones
apiVersion: portworx.io/v1beta2
kind: VolumePlacementStrategy
metadata:
  name: db-placement
spec:
  replicaAffinity:
  - enforcement: required
    topologyKey: topology.kubernetes.io/zone
---
# Use this in StorageClass
parameters:
  placement_strategy: "db-placement"
```

---

## Hands-On Labs

### Lab 9.1 — Observe Write Replication

```bash
# Requires working PX cluster from Module 8

# Create a volume
pxctl volume create --size=5 --repl=3 test-repl-vol

# Inspect replica placement
pxctl volume inspect test-repl-vol | grep -A 10 "Replica"
# Note which 3 nodes hold replicas

# Write data using a pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: repl-pvc
spec:
  storageClassName: portworx-db
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: repl-test
spec:
  schedulerName: stork
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: repl-pvc
  containers:
  - name: fio
    image: ljishen/fio
    command:
    - fio
    - --name=write-test
    - --rw=randwrite
    - --bs=4k
    - --size=500M
    - --numjobs=4
    - --iodepth=32
    - --runtime=60
    - --filename=/data/test
    volumeMounts:
    - name: data
      mountPath: /data
EOF

kubectl wait --for=condition=Ready pod/repl-test --timeout=120s

# Watch replication stats while fio runs
watch pxctl volume inspect $(kubectl get pvc repl-pvc -o jsonpath='{.spec.volumeName}' | xargs kubectl get pv -o jsonpath='{.spec.csi.volumeHandle}')

# Cleanup
kubectl delete pod repl-test
kubectl delete pvc repl-pvc
```

### Lab 9.2 — Simulate Node Failure and Volume Rebuild

```bash
# Create a replicated volume
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ha-pvc
spec:
  storageClassName: portworx-db
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 2Gi
EOF

kubectl wait --for=jsonpath='{.status.phase}'=Bound pvc/ha-pvc --timeout=60s

VOLUME_ID=$(kubectl get pvc ha-pvc -o jsonpath='{.spec.volumeName}' | \
  xargs kubectl get pv -o jsonpath='{.spec.csi.volumeHandle}')

# Check initial replica placement
pxctl volume inspect $VOLUME_ID | grep "Replica Set"

# Write some data
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: ha-test
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: ha-pvc
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "dd if=/dev/urandom of=/data/test.bin bs=1M count=100 && echo DONE && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
EOF

kubectl wait --for=condition=Ready pod/ha-test --timeout=60s

# Find a node holding a replica (not the one with the pod)
POD_NODE=$(kubectl get pod ha-test -o jsonpath='{.spec.nodeName}')
REPLICA_NODE=$(pxctl volume inspect $VOLUME_ID | grep "Online" | grep -v $POD_NODE | head -1 | awk '{print $2}')

echo "Pod node: $POD_NODE"
echo "Removing replica from node: $REPLICA_NODE"

# Simulate node failure by stopping Portworx on that node
# (in Kind, we cordon and drain to simulate)
kubectl cordon $REPLICA_NODE
kubectl drain $REPLICA_NODE --ignore-daemonsets --delete-emptydir-data

# Watch volume health — should show degraded then rebuild
watch pxctl volume inspect $VOLUME_ID

# Verify pod is still running (writes continue with repl=2)
kubectl get pod ha-test

# Uncordon the node — Portworx will rebuild the replica
kubectl uncordon $REPLICA_NODE
watch pxctl volume inspect $VOLUME_ID
# Wait for "Up, Attached On: <node>" to show 3 replicas again

# Cleanup
kubectl delete pod ha-test
kubectl delete pvc ha-pvc
```

### Lab 9.3 — Benchmark Portworx Volume

```bash
# Create a PVC for benchmarking
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: bench-pvc
spec:
  storageClassName: portworx-db
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
EOF

kubectl wait --for=jsonpath='{.status.phase}'=Bound pvc/bench-pvc --timeout=60s

# Run fio benchmark
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: fio-bench
spec:
  schedulerName: stork
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: bench-pvc
  containers:
  - name: fio
    image: ljishen/fio
    command:
    - bash
    - -c
    - |
      echo "=== Random Read IOPS ==="
      fio --name=rand-read --rw=randread --bs=4k --size=1G \
          --numjobs=4 --iodepth=32 --runtime=30 --filename=/data/fio-test
      echo "=== Random Write IOPS ==="
      fio --name=rand-write --rw=randwrite --bs=4k --size=1G \
          --numjobs=4 --iodepth=32 --runtime=30 --filename=/data/fio-test
      echo "=== Sequential Read Throughput ==="
      fio --name=seq-read --rw=read --bs=1M --size=4G \
          --numjobs=1 --iodepth=1 --runtime=30 --filename=/data/fio-test
      sleep 999999
    volumeMounts:
    - name: data
      mountPath: /data
EOF

kubectl wait --for=condition=Ready pod/fio-bench --timeout=120s

# View results
kubectl logs fio-bench -f

# Cleanup
kubectl delete pod fio-bench
kubectl delete pvc bench-pvc
```

---

## Interview Questions

1. What is `/dev/pxd/`? How does Portworx intercept I/O on it?
2. Walk me through what happens at the data plane level when a pod writes 4KB to a Portworx volume with replication factor 3.
3. What is quorum in Portworx? If you have replication factor 3 and 2 nodes go down, what happens?
4. Why does write latency increase when replication traffic goes over a high-latency network?
5. What is the Portworx journal? Why must it be on a separate NVMe disk?
6. What does `io_profile: db` do? How is it different from `io_profile: sequential`?
7. What is hyperconvergence? How does Stork achieve it?
8. A volume shows 20,000 write IOPS on `pxctl volume inspect` but the application is only seeing 5,000. What might explain this?

---

## Production Takeaways

1. **Write latency = max(local write, slowest replica ACK)** — monitor per-node network latency between Portworx nodes. One slow node degrades all volume writes.

2. **Reduce replication factor only under maintenance, not permanently** — replication factor 1 has no redundancy. Factor 2 means one node failure = unprotected. Factor 3 is the production minimum.

3. **Pool utilization alerts** — alert at 70% pool utilization, page at 85%. Portworx cannot provision new volumes or rebuild replicas when the pool is full.

4. **I/O profile cannot fix a slow disk** — if underlying disk IOPS are saturated, tuning the I/O profile won't help. Profile tuning helps with latency variance, not raw throughput ceiling.

5. **Journal disk health is critical** — if the journal disk fills or fails, the entire node loses the ability to acknowledge writes. Monitor journal disk separately.

6. **Replica rebuild traffic is throttled but real** — during rebuild, replication traffic increases. In WAN-stretched clusters, a rebuild can saturate the link. Plan maintenance windows.

---

## Success Criteria

You have completed this module when you can:

- [ ] Explain the write path from application syscall to 3 replica disks
- [ ] Explain quorum and calculate it for any replication factor
- [ ] Explain why journal must be on NVMe (not the data disk)
- [ ] Run a fio benchmark on a Portworx volume and interpret IOPS/latency output
- [ ] Simulate a node failure and observe volume rebuild (Lab 9.2)
- [ ] Explain what I/O profiles do and choose correctly for a given workload

---

## Next Module

[Module 10 — Portworx Operations](./module-10-portworx-operations.md)
