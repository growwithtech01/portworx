# Module 10 — Portworx Operations (Day-2)

```
Phase       : 3 — Portworx
Module      : 10
Status      : draft
Prereqs     : Module 09 — Portworx Data Plane
Estimated   : 5–6 hours
```

---

## Why This Module Exists

Day-1 is installing Portworx. Day-2 is everything that happens after — node failures, disk failures, cluster expansion, rebalancing, upgrades, maintenance. This is where you earn your Kubernetes Platform Engineer title.

Every production incident has a runbook. This module builds your runbook library.

---

## Learning Objectives

After completing this module, you will be able to:

1. Respond to node failure — know what Portworx does automatically vs what you must do
2. Respond to disk failure — detect, remove, replace, rebuild
3. Respond to volume degradation — identify cause, trigger rebuild, verify completion
4. Expand cluster — add nodes, add disks, rebalance pools
5. Shrink cluster — safely decommission nodes without data loss
6. Perform rolling Portworx upgrade using the Operator
7. Perform planned maintenance — drain node without losing volume access
8. Migrate volumes between nodes
9. Monitor cluster health proactively

---

## Prerequisites

- Module 09 — Portworx Data Plane
- Working Portworx cluster

---

## Concepts to Study

### 1. What Portworx Does Automatically (vs What You Do)

Understanding the automation boundary prevents both panic (it's already handled) and complacency (you must intervene):

```
Automatic (Portworx handles without intervention):
├─ Detects node failure (via KVDB heartbeat timeout)
├─ Marks replicas on failed node as degraded
├─ Triggers replica rebuild to surviving nodes
├─ Rebuilds replicas as fast as network/disk allows (throttled)
└─ Updates KVDB with new replica locations

Manual (you must do):
├─ Replace failed hardware
├─ Re-add replacement node to cluster
├─ Decommission permanently failed nodes from KVDB
├─ Monitor rebuild progress and alert on rebuild failures
├─ Trigger manual rebalance if pools become uneven
└─ Approve Portworx upgrades via StorageCluster update
```

---

### 2. Node Failure Runbook

```
Symptom: Portworx node disappears from cluster / node is unreachable
```

**Step 1: Verify the failure**

```bash
# Check node status in Kubernetes
kubectl get nodes
# Node shows NotReady

# Check Portworx sees the node as down
pxctl status
# Node <id>: STATUS=Down, STATUS_DESC=Timeout

# Check volume health
pxctl alerts show | grep -i degraded
```

**Step 2: Assess impact**

```bash
# How many volumes have replicas on the failed node?
pxctl volume list | grep -i degraded

# For each degraded volume, check replication status
pxctl volume inspect <volume-id>
# "Replication Status: Degraded" with replica count showing < repl factor
```

**Step 3: Decide action**

```
Is node coming back?
├─ YES (rebooting, maintenance): Wait up to 30 minutes for auto-recovery
│   Portworx waits before rebuilding to avoid unnecessary rebuild on short outage
└─ NO (hardware failure, node deleted): Decommission immediately
   → proceed to Step 4
```

**Step 4: Decommission permanently failed node**

```bash
# Get node ID of failed node
pxctl node list
# Note the NODE_ID of the failed node

# Decommission node from Portworx cluster
pxctl service node-decommission --node <NODE_ID>

# Portworx will:
# 1. Remove node from KVDB membership
# 2. Trigger rebuild of all volumes that had replicas on this node
# 3. Select new nodes for replacement replicas

# Watch rebuild progress
watch pxctl volume list
# Volumes move from Degraded to Replicating to OK
```

**Step 5: Verify full recovery**

```bash
# All volumes should return to replication factor
pxctl volume list
# No REPLICATION STATUS: Degraded entries

# Cluster should show original minus 1 node
pxctl node list
```

---

### 3. Disk Failure Runbook

```
Symptom: pxctl alerts show disk error | dmesg shows I/O errors on a disk
```

**Step 1: Identify the failed disk**

```bash
# Check alerts
pxctl alerts show | grep -i disk

# Check system dmesg for disk errors (on affected node)
dmesg | grep -iE "i/o error|disk failure|ata error|nvme error" | tail -20

# Check disk health via SMART
smartctl -a /dev/sdb
# Look for: "overall-health: FAILED" or reallocated sectors > 0

# Identify which pool the disk belongs to
pxctl service pool show
```

**Step 2: Remove failed disk from pool**

```bash
# Drain pool to allow disk removal (online operation)
pxctl service pool update --full-drain <pool-id>
# Wait for all data to be evacuated from this pool

# Check pool is drained
pxctl service pool show
# Pool should show: data migrated off

# Remove disk from Portworx
pxctl service drive delete /dev/sdb
```

**Step 3: Replace disk**

```bash
# Physical replacement (hardware team)
# OR
# Add replacement disk to node

# Rescan for new disk
pxctl service drive add --type=auto /dev/sdc

# Verify new disk added to pool
pxctl service pool show

# Rebalance volumes to use new disk
pxctl service pool rebalance <pool-id>
```

---

### 4. Volume Degraded Runbook

```
Symptom: pxctl volume list shows Degraded status
```

**Step 1: Diagnose cause**

```bash
# Inspect degraded volume
pxctl volume inspect <volume-id>
# Look for:
# - Which nodes have replicas
# - Which replica is degraded
# - Replication status detail

# Check alerts for this volume
pxctl alerts show --type volume | grep <volume-id>
```

**Step 2: Check if auto-rebuild is in progress**

```bash
# Volume should show Replicating status if rebuild is in progress
pxctl volume inspect <volume-id> | grep "Replication Status"
# "Replicating" = rebuild in progress (wait for it)
# "Degraded" = rebuild not started or stuck

# Check rebuild progress percentage
watch pxctl volume inspect <volume-id>
```

**Step 3: Trigger manual rebuild if stuck**

```bash
# Manually trigger rebuild to a specific node
pxctl volume ha-update --repl 3 <volume-id>

# Or force rebuild
pxctl volume update --repl 3 <volume-id>
```

**Step 4: Investigate if rebuild keeps failing**

```bash
# Check available space for rebuild target
pxctl service pool show
# Target node must have space >= volume size

# Check network connectivity between nodes
pxctl service node-check

# Check PX logs on target node
kubectl logs -n kube-system portworx-<node> -c portworx | grep <volume-id> | tail -50
```

---

### 5. Cluster Expansion — Adding Nodes

```bash
# Step 1: Provision new node with required disk
# Install OS, join Kubernetes cluster:
kubeadm join <control-plane> --token ...

# Step 2: Label node for Portworx if using node selector
kubectl label node new-node portworx.io/node=true

# Step 3: Portworx Operator auto-detects new node and deploys PX DaemonSet pod
# Watch for new PX pod:
kubectl get pod -n kube-system -l name=portworx -o wide -w
# New pod appears on the new node

# Step 4: PX starts on new node and joins cluster automatically
pxctl node list
# New node appears with status: OK

# Step 5: Rebalance if desired (volumes not auto-migrated by default)
# Optional: move some volumes to distribute load
pxctl service pool rebalance
```

---

### 6. Cluster Shrink — Decommissioning a Node

> Never remove a node while it holds the only surviving replicas of a volume.

```bash
# Step 1: Check which volumes have replicas on the target node
NODE_TO_REMOVE=node3
pxctl volume list | grep $NODE_TO_REMOVE

# Step 2: Drain Portworx volumes off the node
# Option A: Move replicas to other nodes
pxctl service node-decommission --node <NODE_ID>
# Portworx moves all replicas off this node to other nodes

# Option B: Manual move of specific volumes
pxctl volume ha-update --repl 3 --node <other-node-id> <volume-id>

# Step 3: Verify no volumes on the target node
pxctl volume inspect $(pxctl volume list -l | awk 'NR>1 {print $1}') | \
  grep -A 5 "Replica" | grep $NODE_TO_REMOVE
# Should return nothing

# Step 4: Remove from Kubernetes and Portworx
kubectl drain $NODE_TO_REMOVE --ignore-daemonsets --delete-emptydir-data
kubectl delete node $NODE_TO_REMOVE
pxctl node-decommission <NODE_ID>    # removes from KVDB
```

---

### 7. Portworx Upgrade (via Operator)

```bash
# Step 1: Check current version
pxctl version

# Step 2: Review release notes for the target version
# https://docs.portworx.com/release-notes/

# Step 3: Update the StorageCluster image (Operator handles rolling upgrade)
kubectl patch storagecluster portworx -n kube-system --type='json' \
  -p='[{"op": "replace", "path": "/spec/image", "value": "portworx/oci-monitor:3.2.0"}]'

# Step 4: Watch rolling upgrade progress
watch kubectl get pods -n kube-system -l name=portworx
# Operator upgrades one node at a time
# Each node: old pod terminates → new pod starts → health check passes → next node

# Step 5: Verify upgrade complete
pxctl version
kubectl describe storagecluster portworx -n kube-system | grep -A 10 "Conditions"
```

**Never manually update the PX DaemonSet** — always use the Operator. Manual updates break the Operator's state machine.

---

### 8. Planned Maintenance — Drain Node Safely

Before taking a node down for maintenance:

```bash
NODE=node2

# Step 1: Check current volume health
pxctl alerts show | grep -i degraded
# Ensure no volumes are already degraded before adding more risk

# Step 2: Put Portworx into maintenance mode on this node
pxctl service maintenance --enter

# Step 3: Kubernetes drain (stops pods, but volumes stay on Portworx)
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data

# Step 4: Perform maintenance
# (reboot, disk replacement, kernel upgrade, etc.)

# Step 5: Bring node back
kubectl uncordon $NODE

# Step 6: Exit Portworx maintenance mode
pxctl service maintenance --exit

# Step 7: Verify node health
pxctl status
pxctl node list | grep $NODE
```

---

### 9. Volume Migration

Move a volume's primary replica from one node to another:

```bash
# Check current replica placement
pxctl volume inspect <volume-id> | grep "Attached On"

# Migrate primary replica to a specific node
pxctl volume update --attach-node <target-node-id> <volume-id>

# For running pods: use Stork migration CRD (triggers pod reschedule too)
kubectl apply -f - <<EOF
apiVersion: stork.libopenstorage.org/v1alpha1
kind: VolumeSnapshotRestore
metadata:
  name: migrate-vol
spec:
  sourceName: <pvc-name>
  sourceNamespace: <namespace>
  destinationName: <new-pvc-name>
  destinationNamespace: <namespace>
EOF
```

---

### 10. Monitoring and Alerting

**Prometheus metrics** (Portworx ships these natively):

```
px_volume_iops_total          → volume IOPS
px_volume_throughput_bytes    → volume throughput
px_volume_depth_io            → I/O depth
px_volume_read_latency_seconds → read latency
px_volume_write_latency_seconds→ write latency
px_node_status                → node health (1=OK, 0=down)
px_pool_stats_avail_bytes     → available pool capacity
px_cluster_replicas_ready     → replication health
```

**Key alerts to configure**:

```yaml
# Volume degraded
alert: PortworxVolumeDegraded
expr: px_cluster_replicas_ready < 1
for: 5m
labels:
  severity: critical

# Pool utilization high
alert: PortworxPoolCapacityHigh
expr: (px_pool_stats_used_bytes / px_pool_stats_total_bytes) > 0.80
for: 10m
labels:
  severity: warning

# Node down
alert: PortworxNodeDown
expr: px_node_status == 0
for: 2m
labels:
  severity: critical
```

---

## Hands-On Labs

### Lab 10.1 — Simulate and Recover from Node Failure

```bash
# In Kind cluster: simulate node failure by pausing the container
REPLICA_NODE=$(kubectl get nodes --no-headers | grep worker | tail -1 | awk '{print $1}')
CONTAINER_NAME="${REPLICA_NODE}"  # in Kind, node name = container name

# Pause the node (simulate failure)
docker pause $CONTAINER_NAME

# Observe Portworx detecting failure
watch pxctl status
# Node should show down, volumes should show degraded

# After 5 minutes (PX timeout): observe auto-rebuild starting
watch pxctl volume list

# Resume the node
docker unpause $CONTAINER_NAME

# Observe node rejoining and replicas resyncing
watch pxctl node list
```

### Lab 10.2 — Planned Maintenance

```bash
NODE=$(kubectl get nodes --no-headers | grep worker | head -1 | awk '{print $1}')

# Put Portworx in maintenance
kubectl exec -n kube-system portworx-<node-suffix> -c portworx -- \
  /opt/pwx/bin/pxctl service maintenance --enter

# Drain node
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --timeout=300s

# Verify pods rescheduled, volumes still accessible
pxctl volume list

# Exit maintenance
kubectl uncordon $NODE
kubectl exec -n kube-system portworx-<node-suffix> -c portworx -- \
  /opt/pwx/bin/pxctl service maintenance --exit

pxctl status
```

---

## Interview Questions

1. A Portworx node goes down at 2AM. Walk me through what you check first and your escalation decision.
2. A disk on a Portworx node fails. What are the exact steps to replace it?
3. What is the difference between `pxctl service maintenance` and `kubectl drain`?
4. Before removing a node from the cluster, what do you verify to ensure no data loss?
5. A Portworx upgrade is running. One node gets stuck in "Upgrading" for 30 minutes. What do you do?
6. A volume shows "Replicating" but the percentage is not moving. What do you check?
7. How do you verify that a volume has been fully rebuilt after a node failure?
8. An SRE wants to set up alerting for Portworx. Name the three most critical alerts.

---

## Production Takeaways

1. **Portworx rebuild timeout is ~30 minutes** — by default, Portworx waits before triggering rebuild (assumes the node is rebooting). For permanent failures, decommission immediately.

2. **Never remove 2 nodes simultaneously** — with replication factor 3, losing 2 nodes means some volumes have 1 replica. They are unprotected. Remove one node, wait for rebuild, then remove the next.

3. **Pool capacity at 75% = stop adding data** — Portworx needs headroom for rebuilds. A full pool cannot receive rebuild data. Alert at 70%, page at 80%.

4. **Maintenance mode prevents volume eviction** — when a PX node is in maintenance mode, Kubernetes does not evict pods from that node (Stork prevents it). This is intentional.

5. **Upgrades are safe when done via Operator** — the Operator waits for each node to be healthy before moving to the next. Never rush upgrades by manually updating pods.

6. **`pxctl service diags -a` is your first call to support** — always collect diags before opening a ticket. Support will ask for them first.

---

## Success Criteria

You have completed this module when you can:

- [ ] Execute the node failure runbook without notes
- [ ] Execute the disk failure runbook without notes
- [ ] Safely drain a node for maintenance and bring it back (Lab 10.2)
- [ ] Trigger a Portworx upgrade via StorageCluster patch
- [ ] Add a new node to the cluster and verify it joins
- [ ] Configure the three most critical Prometheus alerts for Portworx
- [ ] Explain what Portworx does automatically vs what requires manual intervention

---

## Next Module

[Module 11 — Portworx Enterprise Features](./module-11-portworx-enterprise-features.md)
