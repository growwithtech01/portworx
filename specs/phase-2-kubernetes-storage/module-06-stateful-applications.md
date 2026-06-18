# Module 06 — Stateful Applications in Kubernetes

```
Phase       : 2 — Kubernetes Storage
Module      : 06
Status      : draft
Prereqs     : Module 05 — CSI Architecture
Estimated   : 4–5 hours
```

---

## Why This Module Exists

StatefulSets are the Kubernetes primitive for running databases. Every Portworx use case — MySQL, Postgres, MongoDB, Kafka, Cassandra — runs in a StatefulSet or a StatefulSet-derivative (Operator pattern). Understanding StatefulSets makes Portworx storage placement and failure recovery obvious.

---

## Learning Objectives

After completing this module, you will be able to:

1. Explain why Deployments are wrong for stateful workloads
2. Describe StatefulSet guarantees: stable network identity, stable storage, ordered operations
3. Explain `volumeClaimTemplates` and how StatefulSets manage PVCs
4. Describe all access modes and match them to real workload patterns
5. Explain volume reclaim policy implications for StatefulSets
6. Run MySQL, Postgres, and MongoDB in Kubernetes with proper persistent storage
7. Trace what happens to PVCs when a StatefulSet is scaled down, deleted, or upgraded

---

## Prerequisites

- Module 05 — CSI Architecture
- Kubernetes cluster (Kind or cloud)

---

## Concepts to Study

### 1. Why Deployments Fail for Stateful Workloads

A Deployment manages replicas that are **interchangeable**:

```
Deployment (nginx):
  Pod-abc123 ←→ Pod-def456 ←→ Pod-xyz789   (identical, any can serve traffic)
```

For stateless apps, this is perfect. For stateful apps, it fails:

**Problem 1 — No stable identity**
- Each Deployment pod gets a random name (nginx-5d9f-abc123)
- Pod restarts get new random names
- Databases use their hostname in replication config — a hostname change breaks replication

**Problem 2 — No per-pod storage**
- Deployments share one PVC across all replicas (if configured)
- Two database replicas writing to the same disk = corruption
- Or each pod gets its own PVC via anti-affinity tricks — very complicated

**Problem 3 — No ordering guarantees**
- All pods start simultaneously
- For databases with primary/replica topology, the primary must start before replicas
- Deployments provide no startup ordering

---

### 2. StatefulSet Guarantees

StatefulSets provide three guarantees that Deployments do not:

**Guarantee 1: Stable Network Identity**

```
StatefulSet name: mysql
Replicas: 3

Pods:
  mysql-0    ← always named mysql-0
  mysql-1    ← always named mysql-1
  mysql-2    ← always named mysql-2

DNS (with headless service):
  mysql-0.mysql.namespace.svc.cluster.local
  mysql-1.mysql.namespace.svc.cluster.local
  mysql-2.mysql.namespace.svc.cluster.local
```

Pod `mysql-0` always has the same hostname. After restart, it is still `mysql-0`. Replication config pointing to `mysql-0.mysql.default.svc.cluster.local` always reaches the primary — regardless of pod restarts.

**Guarantee 2: Stable Storage (volumeClaimTemplates)**

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: portworx-sc
      resources:
        requests:
          storage: 10Gi
```

Each pod gets its own PVC:
```
mysql-0 → PVC: data-mysql-0 → PV: portworx-volume-0
mysql-1 → PVC: data-mysql-1 → PV: portworx-volume-1
mysql-2 → PVC: data-mysql-2 → PV: portworx-volume-2
```

When `mysql-0` is restarted, it is ALWAYS re-attached to `data-mysql-0`. Its data is always accessible.

**Guarantee 3: Ordered Operations**

```
Scale up:   mysql-0 starts first → healthy → mysql-1 starts → healthy → mysql-2 starts
Scale down: mysql-2 terminates first → gone → mysql-1 terminates → gone → mysql-0 terminates
Restart:    Same ordering (podManagementPolicy: OrderedReady)
```

This allows the database operator to configure: "pod 0 is always primary, pods 1,2 are always replicas."

---

### 3. The Headless Service

StatefulSets require a headless service (ClusterIP: None):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None          # ← headless: no virtual IP
  selector:
    app: mysql
  ports:
  - port: 3306
```

A normal service gives a single virtual IP (load balanced). A headless service gives DNS records for each individual pod:

```
Normal service:  mysql.default → 10.96.0.100  (one VIP, load balanced)
Headless:        mysql-0.mysql.default → 10.0.0.5  (direct to pod)
                 mysql-1.mysql.default → 10.0.0.6
                 mysql-2.mysql.default → 10.0.0.7
```

Databases need direct pod addressing (primary vs replica routing). The headless service provides this.

---

### 4. volumeClaimTemplates Lifecycle

```
StatefulSet created (replicas: 3)
        ↓
PVCs created: data-mysql-0, data-mysql-1, data-mysql-2
        ↓
StatefulSet scaled down to 1
        ↓
Pods mysql-2 and mysql-1 deleted
        ↓
PVCs data-mysql-2 and data-mysql-1: NOT deleted (data preserved!)
        ↓
StatefulSet scaled back to 3
        ↓
mysql-1 re-attaches to data-mysql-1 (same data)
mysql-2 re-attaches to data-mysql-2 (same data)
```

**StatefulSet deletion**:

```
kubectl delete statefulset mysql
        ↓
Pods: deleted
PVCs: NOT deleted (you must delete manually)
PVs: NOT deleted (depends on reclaim policy)
```

This is by design. Kubernetes protects you from accidentally deleting database data.

```bash
# After deleting StatefulSet, PVCs remain:
kubectl get pvc -n default
# NAME           STATUS   VOLUME         CAPACITY   ...
# data-mysql-0   Bound    portworx-vol   10Gi       ...
# data-mysql-1   Bound    portworx-vol   10Gi       ...
# data-mysql-2   Bound    portworx-vol   10Gi       ...

# You must manually delete PVCs when you're sure:
kubectl delete pvc data-mysql-0 data-mysql-1 data-mysql-2
```

---

### 5. Access Modes in Practice

```
ReadWriteOnce (RWO) — block storage
┌────────────────────────────────────────────┐
│ One node mounts read-write                 │
│ Multiple pods on SAME node can use it      │
│ Best for: databases, single-node workloads │
│ Portworx: standard volume                  │
└────────────────────────────────────────────┘

ReadWriteMany (RWX) — shared filesystem
┌────────────────────────────────────────────┐
│ Multiple nodes mount read-write            │
│ Best for: shared web assets, ML datasets   │
│ Portworx: SharedV4 volume (NFS-based)      │
│ Higher latency than RWO                    │
└────────────────────────────────────────────┘

ReadOnlyMany (ROX) — shared readonly
┌────────────────────────────────────────────┐
│ Multiple nodes mount read-only             │
│ Best for: shared reference data, config    │
│ Portworx: supported                        │
└────────────────────────────────────────────┘
```

**StatefulSet + RWO**: Each pod gets its own RWO PVC. No sharing. This is correct for databases.

**Deployment + RWX**: All replicas share one PVC via NFS. This is correct for stateless apps that need shared files (web servers reading the same static assets).

---

### 6. Init Containers for Database Initialization

StatefulSets often need init containers for:
- Setting permissions on mounted volumes
- Cloning data from primary before replica starts
- Waiting for dependencies

```yaml
initContainers:
- name: init-permissions
  image: busybox
  command: ["sh", "-c", "chown -R 999:999 /var/lib/mysql"]
  volumeMounts:
  - name: data
    mountPath: /var/lib/mysql
- name: clone-primary
  image: gcr.io/google-samples/xtrabackup:1.0
  command:
  - bash
  - -c
  - |
    # If we're not mysql-0 (not primary), clone from mysql-0
    [[ $(hostname) =~ -([0-9]+)$ ]] && ordinal=${BASH_REMATCH[1]}
    if [[ $ordinal -eq 0 ]]; then
      exit 0  # Primary: no cloning needed
    fi
    # Clone from primary
    ncat --recv-only mysql-0.mysql 3307 | xbstream -x -C /var/lib/mysql
```

---

## Hands-On Labs

### Lab 6.1 — MySQL StatefulSet with Persistent Storage

```bash
kind create cluster

# Create headless service
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - name: mysql
    port: 3306
EOF

# Create StatefulSet
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-permissions
        image: busybox
        command: ["sh", "-c", "mkdir -p /var/lib/mysql && chown -R 999:999 /var/lib/mysql"]
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
        - name: MYSQL_DATABASE
          value: "testdb"
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-h", "localhost", "-ppassword"]
          initialDelaySeconds: 30
          periodSeconds: 10
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF

kubectl wait --for=condition=Ready pod/mysql-0 --timeout=180s

# Verify PVC was created
kubectl get pvc
# NAME          STATUS   VOLUME   CAPACITY
# data-mysql-0  Bound    ...      1Gi

# Write data
kubectl exec mysql-0 -- mysql -uroot -ppassword -e \
  "CREATE TABLE testdb.users (id INT, name VARCHAR(50)); \
   INSERT INTO testdb.users VALUES (1, 'Alice');"

# Simulate pod failure
kubectl delete pod mysql-0

# Watch pod restart with SAME name and SAME PVC
kubectl get pod -w &
WATCHPID=$!
sleep 30
kill $WATCHPID

# Verify data persisted
kubectl wait --for=condition=Ready pod/mysql-0 --timeout=120s
kubectl exec mysql-0 -- mysql -uroot -ppassword -e "SELECT * FROM testdb.users;"

# Cleanup
kubectl delete statefulset mysql
kubectl delete service mysql
kubectl get pvc  # PVCs remain — must delete manually
kubectl delete pvc data-mysql-0
kind delete cluster
```

### Lab 6.2 — Scale StatefulSet and Observe PVC Behavior

```bash
kind create cluster

# Create simple StatefulSet
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  clusterIP: None
  selector:
    app: app
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: app
spec:
  selector:
    matchLabels:
      app: app
  serviceName: app
  replicas: 3
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: busybox
        command: ["sh", "-c", "echo $(hostname) > /data/hostname.txt && sleep 3600"]
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Mi
EOF

kubectl wait --for=condition=Ready pod/app-0 pod/app-1 pod/app-2 --timeout=120s

# Each pod wrote its own hostname
kubectl exec app-0 -- cat /data/hostname.txt   # app-0
kubectl exec app-1 -- cat /data/hostname.txt   # app-1
kubectl exec app-2 -- cat /data/hostname.txt   # app-2

# Scale down
kubectl scale statefulset app --replicas=1
kubectl wait --for=delete pod/app-2 --timeout=60s
kubectl wait --for=delete pod/app-1 --timeout=60s

# PVCs still exist!
kubectl get pvc

# Scale back up
kubectl scale statefulset app --replicas=3
kubectl wait --for=condition=Ready pod/app-0 pod/app-1 pod/app-2 --timeout=120s

# Data from before scale-down is still there
kubectl exec app-1 -- cat /data/hostname.txt   # app-1 (from before!)
kubectl exec app-2 -- cat /data/hostname.txt   # app-2 (from before!)

# Cleanup
kubectl delete statefulset app
kubectl delete service app
kubectl delete pvc data-app-0 data-app-1 data-app-2
kind delete cluster
```

---

## Exercises

### Exercise 1 — StatefulSet Design

Design the StatefulSet spec (including service, volumeClaimTemplates, and storage requirements) for:

1. **PostgreSQL primary with 2 read replicas** — 100GB data, synchronous replication
2. **Kafka cluster (3 brokers)** — 500GB logs per broker, high throughput
3. **MongoDB replica set (3 members)** — 50GB data, WiredTiger storage engine

For each, specify:
- Access mode
- Storage class requirements (IOPS, replication factor)
- headless service vs regular service needs
- Init container requirements

### Exercise 2 — Failure Scenario Analysis

For each scenario, describe what happens to:
- The pods
- The PVCs
- The data

| Scenario | Pods | PVCs | Data |
|----------|------|------|------|
| `kubectl delete pod mysql-0` (StatefulSet exists) | | | |
| `kubectl delete statefulset mysql` | | | |
| `kubectl delete statefulset mysql --cascade=orphan` | | | |
| Node running mysql-0 reboots (node comes back) | | | |
| Node running mysql-0 dies permanently | | | |

---

## Interview Questions

1. Why can't you use a Deployment for MySQL with multiple replicas?
2. What three guarantees does a StatefulSet provide that a Deployment does not?
3. A StatefulSet pod is deleted. What is its new name after restart?
4. You delete a StatefulSet. What happens to its PVCs?
5. What is a headless service and why do StatefulSets require one?
6. A developer wants to run Postgres with 3 read replicas sharing one volume for efficiency. Is this correct? What would you recommend?
7. What is `podManagementPolicy: Parallel` and when would you use it?
8. A StatefulSet is scaled from 3 to 5 replicas. Walk me through how pods 3 and 4 get their storage.

---

## Production Takeaways

1. **Never use Deployments for databases** — the absence of stable identity and stable storage causes data corruption in split-brain scenarios.

2. **StatefulSet PVCs survive StatefulSet deletion by design** — always have a cleanup runbook that verifies data before deleting PVCs.

3. **`podManagementPolicy: Parallel` is for stateless-ish workloads** — Kafka sometimes uses it. For databases with primary/replica, keep `OrderedReady`.

4. **Headless service DNS is how replicas find each other** — `mysql-0.mysql.default.svc.cluster.local` resolves directly to the pod IP. Replicas use this for replication configuration.

5. **init containers prevent data directory permission issues** — MySQL, Postgres, and MongoDB are sensitive about file permissions on their data directories. Use an init container to `chown` before the main container starts.

6. **`volumeClaimTemplates` is immutable** — you cannot change the storage size in a volumeClaimTemplate. To resize, you must resize each PVC individually using `kubectl edit pvc`.

---

## Success Criteria

You have completed this module when you can:

- [ ] Explain all three StatefulSet guarantees from memory
- [ ] Write a complete StatefulSet YAML (headless service + volumeClaimTemplates) for MySQL
- [ ] Demonstrate PVC persistence across pod deletion and restart (Lab 6.1)
- [ ] Demonstrate PVC survival across StatefulSet scale-down and scale-up (Lab 6.2)
- [ ] Complete the failure scenario matrix exercise without looking at answers
- [ ] Explain what happens to PVCs when a StatefulSet is deleted

---

## Next Module

[Module 07 — Portworx Architecture](../phase-3-portworx/module-07-portworx-architecture.md)

> You have completed Phase 2. You now understand the full Kubernetes storage stack from containers to CSI to StatefulSets. Phase 3 begins with Portworx — and every concept you have built now makes its architecture immediately readable.
