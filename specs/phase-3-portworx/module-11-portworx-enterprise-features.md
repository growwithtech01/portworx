# Module 11 — Portworx Enterprise Features

```
Phase       : 3 — Portworx
Module      : 11
Status      : draft
Prereqs     : Module 10 — Portworx Operations
Estimated   : 3–4 hours (theory + architecture — no lab required)
```

---

## Why This Module Exists

This module covers the features that differentiate Portworx from a basic CSI driver. These are the features you will be asked about in interviews, in customer conversations, and in architecture reviews:

- Why do we pay for Portworx instead of using the cloud provider's native CSI?
- What is the difference between async DR and sync DR?
- What is PX-Backup and how is it different from etcd backup?

You will not lab these (they require cloud infrastructure and Enterprise licensing). But you will understand them deeply enough to architect solutions and debug them conceptually.

---

## Learning Objectives

After completing this module, you will be able to:

1. Explain the three disaster recovery models: Async DR, Sync DR, Metro DR
2. Explain PX-Backup: what it protects, how it works, RPO/RTO characteristics
3. Explain volume snapshots: local, cloud-based, schedule-based
4. Explain volume encryption and when to use it
5. Explain Autopilot: automatic capacity management
6. Explain volume placement strategies for multi-zone clusters
7. Explain SharedV4 volumes (ReadWriteMany)
8. Explain PX-Security (RBAC for Portworx)

---

## Prerequisites

- Module 10 — Portworx Operations

---

## Concepts to Study

### 1. Snapshots

A snapshot is a point-in-time copy of a volume. Portworx supports three snapshot types:

**Local Snapshot**

```
Volume A (live)  →  Volume A-snapshot-20260618
                     (read-only point-in-time copy)
                     (uses copy-on-write — only stores diffs)
```

- Stored on the same Portworx cluster
- Fast creation (copy-on-write — no data copy at creation time)
- Use: quick restore after configuration error, test environment clones
- NOT a backup (lives on same cluster — cluster failure = snapshot lost)

```bash
# Create local snapshot
pxctl volume snapshot create <volume-id> --name snap-before-upgrade

# List snapshots
pxctl volume snapshot list

# Restore from snapshot (creates new volume)
pxctl volume restore --snap snap-before-upgrade --name vol-restored

# Schedule snapshots
pxctl volume snap-schedule-update \
  --daily @08:00,keep=7 \
  --weekly Sunday@09:00,keep=4 \
  <volume-id>
```

**Cloud Snapshot (Cloudsnap)**

```
Volume A (on PX cluster)  →  Object Storage (S3, GCS, Azure Blob)
                              (full snapshot stored externally)
```

- Stored off-cluster in object storage
- True backup: cluster can be destroyed and data is still in S3
- First snapshot: full copy. Subsequent snapshots: incremental (only changed blocks)
- Restore: pulls data back from S3 to a new volume

```bash
# Configure cloud credentials
pxctl credentials create --provider s3 \
  --s3-access-key <key> \
  --s3-secret-key <secret> \
  --s3-region us-east-1 \
  --s3-endpoint s3.amazonaws.com \
  --bucket my-px-backup-bucket

# Trigger cloud snapshot
pxctl cloudsnap backup <volume-id> --cred-id <cred-id>

# List cloud snapshots
pxctl cloudsnap list

# Restore from cloud snapshot
pxctl cloudsnap restore --snap <snap-id> --cred-id <cred-id>
```

**Stork Volume Snapshots (Kubernetes-native)**

```yaml
# Create snapshot using Kubernetes VolumeSnapshot CRD
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-snap-20260618
spec:
  volumeSnapshotClassName: portworx-snapshot-class
  source:
    persistentVolumeClaimName: mysql-pvc
---
# Restore: create PVC from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-restored
spec:
  dataSource:
    name: mysql-snap-20260618
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 50Gi
```

---

### 2. PX-Backup (Application-Consistent Backup)

PX-Backup is a separate product from Portworx that provides:
- Application-consistent backups (not just disk snapshots)
- Backup to object storage (S3-compatible)
- Cross-cluster and cross-cloud restore
- Kubernetes namespace backup (PVCs + application manifests together)
- Scheduled backup policies

**How it differs from raw cloudsnap**:

```
cloudsnap (raw):
  - Backs up disk blocks
  - Does NOT quiesce the application
  - May capture inconsistent state (e.g., mid-transaction MySQL write)

PX-Backup:
  - Runs pre-backup hooks (quiesce DB, flush logs, lock tables)
  - Takes snapshot
  - Runs post-backup hooks (unquiesce)
  - Backs up app manifests alongside volume data
  - Result: a consistent backup you can restore to ANY cluster
```

**PX-Backup Architecture**:

```
PX-Backup Server (runs separately from PX cluster)
├─ Web UI (schedule, monitor, restore)
├─ Stork (handles snapshot + manifest capture)
├─ Object Storage (S3/GCS/Azure — backup destination)
└─ License server
```

**Key concepts**:

```
Backup Location   → S3 bucket (where backups go)
Backup Schedule   → cron expression + retention policy
Backup Rule       → pre/post hooks for app quiescing
Namespace Backup  → backup all PVCs + deployments in a namespace
```

**RPO and RTO**:

```
RPO (Recovery Point Objective) = backup frequency
  - Hourly backups → max 1 hour of data loss
  - Daily backups → max 24 hours of data loss

RTO (Recovery Time Objective) = time to restore
  - PX-Backup restore: 10–60 minutes depending on data size
  - Local snapshot restore: seconds (same cluster)
  - Cloud snapshot restore: minutes to hours (data transfer from S3)
```

---

### 3. Disaster Recovery Models

**Model 1: Async DR (Asynchronous Disaster Recovery)**

```
┌─────────────────────┐              ┌─────────────────────┐
│  PRIMARY CLUSTER    │              │  DR CLUSTER         │
│  (Active)           │              │  (Standby)          │
│                     │              │                     │
│  App → Volume       │ ────────→   │  Volume copy        │
│                     │  async      │  (hours behind)     │
│                     │  replication│                     │
└─────────────────────┘              └─────────────────────┘
```

- Replication happens asynchronously (data is sent after write is acknowledged)
- RPO: minutes to hours (depending on replication interval)
- RTO: manual failover required (typically 30–60 minutes with runbooks)
- Use: cross-region DR (latency too high for synchronous replication)
- Cost: no write latency impact on primary workload

```yaml
# AsyncDR requires Stork CRD: MigrationSchedule
apiVersion: stork.libopenstorage.org/v1alpha1
kind: MigrationSchedule
metadata:
  name: mysql-dr-schedule
spec:
  template:
    spec:
      clusterPair: primary-to-dr
      namespaces: [production]
      includeResources: true
      startApplications: false
  schedulePolicyName: every-15-minutes
```

**Model 2: Sync DR (Synchronous Disaster Recovery)**

```
┌─────────────────────┐              ┌─────────────────────┐
│  PRIMARY CLUSTER    │              │  DR CLUSTER         │
│  (Active)           │              │  (Active-Standby)   │
│                     │              │                     │
│  App → Volume       │ ←──────────→ │  Volume copy        │
│                     │  synchronous │  (zero RPO)         │
│                     │  replication │                     │
└─────────────────────┘              └─────────────────────┘
```

- Write is not acknowledged until BOTH clusters have written
- RPO: zero (no data loss on failover)
- RTO: automatic failover in 60–90 seconds (with health monitoring)
- Use: financial systems, medical records, regulatory requirements
- Cost: every write adds cross-cluster network RTT to write latency

**Model 3: Metro DR (Synchronous, Multi-Zone)**

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│  Zone A     │   │  Zone B     │   │  Zone C     │
│  (Active)   │   │  (Active)   │   │  (Witness)  │
│             │   │             │   │  (KVDB only)│
│  App        │   │  Volume     │   │  KVDB       │
│  Volume     │←──│  replica    │   │  quorum     │
└─────────────┘   └─────────────┘   └─────────────┘
     same metropolitan area (< 5ms RTT)
```

- All zones in same metro area (low latency, synchronous replication viable)
- Zone C is a witness (KVDB only — no data, just quorum vote)
- Survives complete zone failure with zero RPO and automatic RTO (~60s)
- Use: on-prem multi-datacenter, cloud multi-AZ enterprise workloads

---

### 4. Autopilot (Automatic Capacity Management)

Autopilot monitors storage metrics and automatically takes actions:

```
Monitors:
  Volume usage → auto-expand PVC when > 70% full
  Pool usage   → auto-rebalance when uneven
  Performance  → alert when latency threshold exceeded

Actions:
  Expand PVC   → calls CSI ExpandVolume
  Expand pool  → adds disks from cloud (if cloud-native)
  Alert        → sends alert via Alertmanager
```

```yaml
# AutopilotRule: expand volume when 70% full
apiVersion: autopilot.libopenstorage.org/v1alpha1
kind: AutopilotRule
metadata:
  name: volume-expand-when-70-percent
spec:
  selector:
    matchLabels:
      autopilot: "true"
  namespaceSelector:
    matchLabels:
      autopilot: "enabled"
  conditions:
    expressions:
    - key: "100 * (px_volume_usage_bytes / px_volume_capacity_bytes)"
      operator: Gt
      values: ["70"]
  actions:
  - name: openstorage.io.action.volume/resize
    params:
      scalepercentage: "100"     # double the size
      maxsize: "400Gi"           # never grow beyond 400Gi
```

```bash
# Check Autopilot status
kubectl get autopilot -n kube-system
kubectl get arp    # AutopilotRule Progress
```

---

### 5. SharedV4 Volumes (ReadWriteMany)

Standard Portworx volumes are ReadWriteOnce — one node mounts read-write. SharedV4 adds ReadWriteMany:

```
SharedV4 volume:
├─ Portworx volume (block, replicated)
└─ NFS server running inside Portworx (on the primary replica node)
    └─ All other nodes mount via NFS
    
Result: multiple pods on multiple nodes can mount read-write
```

```yaml
# StorageClass for SharedV4
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: portworx-rwx
provisioner: pxd.portworx.com
parameters:
  repl: "2"
  sharedv4: "true"
  sharedv4_svc_type: "ClusterIP"   # NFS service type
---
# PVC with ReadWriteMany
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  storageClassName: portworx-rwx
  accessModes: [ReadWriteMany]
  resources:
    requests:
      storage: 10Gi
```

**Latency warning**: SharedV4 adds NFS overhead. Write latency is higher than standard RWO volumes. Use for:
- Web server document roots (reads dominate)
- ML training datasets (read-heavy)
- Shared configuration files

Do NOT use for high-throughput databases — use separate RWO volumes per replica.

---

### 6. Volume Encryption

Portworx integrates with key management systems to encrypt volume data at rest:

```
Encryption backends:
├─ AWS KMS
├─ Google Cloud KMS
├─ Azure Key Vault
├─ HashiCorp Vault
└─ Kubernetes Secrets (for development)
```

```yaml
# StorageClass with encryption
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: portworx-encrypted
provisioner: pxd.portworx.com
parameters:
  repl: "3"
  secure: "true"
  secret_name: "px-secret"         # Kubernetes secret with encryption key
  secret_namespace: "kube-system"
```

**How it works**:

```
Volume created with encryption:
  Portworx generates a data encryption key (DEK)
  DEK is encrypted by the key management system (KMS)
  Encrypted DEK stored in KVDB
  All data written to disk is AES-256 encrypted

On mount:
  Portworx fetches encrypted DEK from KVDB
  Sends to KMS for decryption
  Uses decrypted DEK to decrypt data as it's read
```

**Per-volume vs Cluster-wide encryption**:
- Per-volume: each volume has its own key (rotation, access control per volume)
- Cluster-wide: one key for all volumes (simpler but coarser access control)

---

### 7. Volume Placement Strategies (VPS)

For multi-zone, multi-rack clusters — control where Portworx places replicas:

```yaml
# Ensure replicas span zones
apiVersion: portworx.io/v1beta2
kind: VolumePlacementStrategy
metadata:
  name: zone-spread
spec:
  replicaAffinity:
  - enforcement: required
    topologyKey: topology.kubernetes.io/zone
---
# Ensure replicas are on nodes with SSD labels
spec:
  replicaAffinity:
  - enforcement: preferred
    matchExpressions:
    - key: storage-tier
      operator: In
      values: ["ssd"]
---
# Anti-affinity: keep volume replicas away from specific nodes
spec:
  replicaAntiAffinity:
  - enforcement: required
    topologyKey: kubernetes.io/hostname
```

---

### 8. PX-Security (RBAC)

PX-Security provides role-based access control for Portworx:

```
Roles:
├─ system.admin    → full cluster access
├─ system.user     → create/manage own volumes
├─ system.view     → read-only cluster view
└─ custom roles    → fine-grained permissions

Token-based authentication:
  Portworx issues JWT tokens
  pxctl --auth-token <token> <command>
  Applications authenticate via ServiceAccount token
```

**When to use**: Multi-tenant clusters where different teams must not see each other's volumes.

---

## Exercises

### Exercise 1 — DR Model Selection

For each business requirement, select the correct DR model and justify:

| Requirement | DR Model | Justification |
|-------------|----------|---------------|
| Fintech: zero data loss, automatic failover, 5ms latency acceptable | | |
| E-commerce: up to 15 minutes data loss acceptable, primary in us-east, DR in eu-west | | |
| Healthcare: zero RPO, same city (2 datacenters, 1ms RTT) | | |
| SaaS startup: nightly backup is sufficient, restore within 4 hours | | |

### Exercise 2 — Snapshot Strategy Design

Design a snapshot strategy for:

1. **Production MySQL** (50GB, critical, RPO=1 hour, RTO=30 minutes)
   - Snapshot type?
   - Frequency?
   - Retention?
   - Backup location?
   - App quiescing needed?

2. **Dev/Test PostgreSQL** (10GB, non-critical, weekly backup acceptable)
   - Same questions

---

## Interview Questions

1. What is the difference between a local snapshot, a cloud snapshot, and PX-Backup? When do you use each?
2. A customer has two datacenters 500km apart. They need zero RPO. Which Portworx DR model do you recommend and why?
3. What is Metro DR? How is it different from Sync DR?
4. What is Autopilot? What happens if Autopilot expands a volume to its `maxsize` and it's still full?
5. Why does SharedV4 have higher latency than a standard Portworx RWO volume?
6. What is the difference between PX-Backup and PX-Cloudsnap?
7. A developer wants to use volume encryption but is worried about performance impact. What do you tell them?
8. What is a VolumePlacementStrategy and when would you use one?

---

## Production Takeaways

1. **Cloudsnap is not a backup if you don't test restore** — schedule monthly restore tests. An untested backup is not a backup.

2. **Sync DR doubles write latency** — because every write waits for cross-cluster ACK. Always benchmark before committing to sync DR in production.

3. **Autopilot maxsize prevents unbounded growth** — always set `maxsize` in AutopilotRule. Without it, a runaway application can fill your entire cluster.

4. **SharedV4 NFS server is a single point of failure** — if the node hosting the SharedV4 primary fails, NFS clients see an I/O error until Portworx migrates the NFS server. Plan for this in your RTO.

5. **Encryption keys must be backed up separately** — losing the KMS key means losing access to all encrypted volumes permanently. Back up KMS keys to offline storage.

6. **PX-Security tokens expire** — configure token rotation or use Kubernetes ServiceAccount tokens (which Portworx refreshes automatically).

---

## Success Criteria

You have completed this module when you can:

- [ ] Explain the three DR models and choose the right one for any scenario (Exercise 1)
- [ ] Design a snapshot and backup strategy for a production database (Exercise 2)
- [ ] Explain the difference between Cloudsnap and PX-Backup
- [ ] Explain SharedV4 architecture and its latency tradeoff
- [ ] Explain how volume encryption works (DEK, KMS, KVDB)
- [ ] Answer all 8 interview questions without notes

---

## Next Module

[Module 12 — Portworx Production Debugging](./module-12-portworx-production-debugging.md)
