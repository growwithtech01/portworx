# Module 12 — Portworx Production Debugging

```
Phase       : 3 — Portworx
Module      : 12 (Final)
Status      : draft
Prereqs     : All previous modules
Estimated   : 6–8 hours (scenario-based deep dives)
```

---

## Why This Module Exists

This is the module that turns knowledge into operational competence. Every scenario is a real production incident pattern. Work through each one — symptom, investigation, commands, root cause, resolution — until you can execute the investigation without looking at the guide.

A Portworx engineer who knows architecture but panics during incidents is not production-ready. This module is how you become production-ready.

---

## Learning Objectives

After completing this module, you will be able to:

1. Execute a systematic investigation for any Portworx incident
2. Know which commands to run at each stage of investigation
3. Distinguish between Kubernetes-layer failures and Portworx-layer failures
4. Read Portworx logs and identify the failing component
5. Write a postmortem for any storage incident

---

## Prerequisites

- All 11 previous modules completed
- `pxctl` access on a Portworx cluster

---

## Investigation Framework

Before every incident, ask these questions in order:

```
1. SCOPE:   How many volumes/pods/nodes are affected?
2. TIMING:  When did it start? What changed before?
3. LAYER:   Is this Kubernetes (PVC/PV/CSI) or Portworx (data plane/KVDB)?
4. HEALTH:  Are all Portworx nodes healthy? Is KVDB healthy?
5. EVENTS:  What do kubectl events and pxctl alerts say?
6. LOGS:    What do PX logs say on the affected node?
```

---

## Diagnostic Command Reference

```bash
# ─── Cluster Health ───────────────────────────────────────────
pxctl status                                    # cluster-level status
pxctl node list                                 # all nodes and status
pxctl service kvdb status                       # KVDB quorum health
pxctl alerts show                               # all current alerts

# ─── Volume Investigation ─────────────────────────────────────
pxctl volume list                               # all volumes and health
pxctl volume list | grep -v Up                  # degraded/offline volumes
pxctl volume inspect <volume-id>                # deep volume detail
pxctl volume inspect <volume-id> | grep -A 20 "Replication"

# ─── Node Investigation ───────────────────────────────────────
pxctl service pool show                         # storage pools
pxctl service pool show --json | jq '.'         # pool detail in JSON
pxctl node list --json | jq '.[] | {id, status, used, available}'

# ─── CSI / Kubernetes ─────────────────────────────────────────
kubectl get pvc -A | grep -v Bound              # unbound PVCs
kubectl get volumeattachment                    # volume attachments
kubectl describe pvc <name> -n <ns>             # PVC events
kubectl describe pod <name> -n <ns>             # pod mount events
kubectl get events -A --sort-by='.lastTimestamp' | grep -i volume

# ─── Portworx Logs ────────────────────────────────────────────
# Logs on specific node
kubectl logs -n kube-system portworx-<node> -c portworx | tail -200
# Filter for errors
kubectl logs -n kube-system portworx-<node> -c portworx | grep -iE "error|fail|panic|fatal"
# Filter for specific volume
kubectl logs -n kube-system portworx-<node> -c portworx | grep <volume-id>

# ─── Diag Collection ──────────────────────────────────────────
pxctl service diags -a                          # collect all node diags
pxctl service diags --output /tmp/px-diags.tgz  # save to file
```

---

## Scenario 1 — PVC Stuck in Pending

```
Symptom: kubectl get pvc shows STATUS: Pending for > 5 minutes
Impact:  Pod cannot start, application is down
```

### Investigation

```bash
# Step 1: Get PVC details
kubectl describe pvc <pvc-name> -n <namespace>
# Read the Events section carefully

# Step 2: Check StorageClass
SC=$(kubectl get pvc <pvc-name> -n <namespace> -o jsonpath='{.spec.storageClassName}')
kubectl get storageclass $SC
kubectl describe storageclass $SC
# Does it exist? Is provisioner correct?

# Step 3: Check if WaitForFirstConsumer is the cause
kubectl get storageclass $SC -o jsonpath='{.volumeBindingMode}'
# WaitForFirstConsumer: PVC waits until pod is scheduled. Is pod running?
kubectl get pod -n <namespace> | grep <app-name>

# Step 4: Check CSI provisioner logs
kubectl get pod -n kube-system | grep portworx
# Find the PX pod on any node
kubectl logs -n kube-system portworx-<node> -c portworx | grep -i "provision\|createvolume" | tail -50

# Step 5: Check Portworx cluster health
pxctl status
# If cluster is degraded, provisioning may be paused
```

### Root Causes

| Finding | Root Cause | Fix |
|---------|------------|-----|
| "no StorageClass found" | StorageClass name typo | Correct storageClassName in PVC |
| "WaitForFirstConsumer" + no pod | Normal behavior | Create the pod first |
| PX status shows degraded | Cluster below quorum | Fix PX cluster first |
| "no available nodes match volume placement" | VPS constraints too strict | Relax VolumePlacementStrategy |
| "not enough storage in pool" | Pool is full | Expand pool or add disks |

---

## Scenario 2 — Volume Not Mounting (Pod Stuck in ContainerCreating)

```
Symptom: Pod shows ContainerCreating for > 5 minutes, PVC is Bound
Impact:  Application is completely down
```

### Investigation

```bash
# Step 1: Describe pod — read Events section
kubectl describe pod <pod-name> -n <namespace>
# Map event to CSI call:
# "AttachVolume.Attach failed"     → ControllerPublishVolume
# "MountVolume.MountDevice failed" → NodeStageVolume
# "MountVolume.SetUp failed"       → NodePublishVolume

# Step 2: Check volume attachment
kubectl get volumeattachment | grep <pv-name>
# Is it there? Is it Attached?

# Step 3: Get the volume ID and check PX
PV_NAME=$(kubectl get pvc <pvc-name> -n <namespace> -o jsonpath='{.spec.volumeName}')
VOLUME_ID=$(kubectl get pv $PV_NAME -o jsonpath='{.spec.csi.volumeHandle}')
pxctl volume inspect $VOLUME_ID

# Step 4: Check node where pod is scheduled
NODE=$(kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.nodeName}')
echo "Pod is on: $NODE"

# Is PX running on that node?
kubectl get pod -n kube-system -l name=portworx -o wide | grep $NODE

# Step 5: Check PX logs on the failing node
PX_POD=$(kubectl get pod -n kube-system -l name=portworx -o wide | grep $NODE | awk '{print $1}')
kubectl logs -n kube-system $PX_POD -c portworx | grep -iE "mount|attach|stage" | tail -100

# Step 6: Check if volume is already attached to another node
pxctl volume inspect $VOLUME_ID | grep "Attached On"
# If attached to node A but pod is on node B = force-detach needed
```

### Root Causes

| Finding | Root Cause | Fix |
|---------|------------|-----|
| "Volume already exclusively attached" | Pod rescheduled but volume still attached | Wait for detach (up to 6 min) or force detach |
| "NodeStageVolume failed: device not found" | PX not running on target node | Fix PX DaemonSet pod |
| "permission denied on mount" | Wrong fsGroup in pod securityContext | Fix securityContext in pod spec |
| VolumeAttachment stuck Attaching | CSI controller not running | Check CSI controller pod |
| "timeout waiting for volume" | Slow disk or disk failure | Check disk health |

### Force Detach (Last Resort)

```bash
# Only if: pod is gone, volume is stuck attached, node is unreachable
pxctl host detach <volume-id>
# Then delete VolumeAttachment manually
kubectl delete volumeattachment <va-name>
```

---

## Scenario 3 — Volume Degraded After Node Failure

```
Symptom: pxctl alerts shows "Volume degraded", volume replication below factor
Impact:  Data still accessible but unprotected (one more node failure = potential loss)
```

### Investigation

```bash
# Step 1: Assess scope
pxctl volume list | grep -i degraded
# How many volumes? Just one application or cluster-wide?

# Step 2: Inspect the degraded volume
pxctl volume inspect <volume-id>
# Which node's replica is missing?
# What is the current replication status?

# Step 3: Is rebuild in progress or stuck?
pxctl volume inspect <volume-id> | grep -A 5 "Replication Status"
# "Replicating" = rebuild in progress (progress shows %)
# "Degraded" with no progress = stuck

# Step 4: Check available nodes for rebuild target
pxctl node list
# Is there a node with enough space?

# Step 5: Check rebuild speed / throttle
pxctl service options
# Look for: replication_throttle_mb
```

### Resolution

```bash
# If rebuild stuck: manually trigger
pxctl volume ha-update --repl 3 <volume-id>

# If no node has space: expand a pool
# (Requires cloud disk addition or physical disk)
pxctl service pool update --add /dev/sdc <pool-id>

# Monitor rebuild progress
watch pxctl volume inspect <volume-id>
# Wait for all replicas to show "Online"

# Check node that failed is decommissioned (if permanently gone)
pxctl node list
pxctl service node-decommission --node <failed-node-id>
```

---

## Scenario 4 — Disk Full / Pool Exhausted

```
Symptom: Application writes failing, pxctl shows pool near 100% usage
Impact:  All applications using this pool cannot write new data
```

### Investigation

```bash
# Step 1: Find the full pool
pxctl service pool show
# Look for Used close to TotalSize

# Step 2: Identify which volumes are largest
pxctl volume list --json | jq 'sort_by(.Usage) | reverse | .[0:10] | .[] | {name, usage: .Usage, capacity: .Spec.size}'

# Step 3: Check for snapshots consuming space
pxctl volume snapshot list
# Old snapshots can consume significant space

# Step 4: Check for detached volumes (not in use but consuming space)
pxctl volume list --status detached

# Step 5: Check node-level disk usage
kubectl debug node/<node> -it --image=ubuntu -- bash
# nsenter -t 1 -m -u -i -n -p -- df -h
# nsenter -t 1 -m -u -i -n -p -- lsblk
```

### Resolution (in order of risk)

```bash
# Option 1: Delete old snapshots (low risk)
pxctl volume snapshot delete <snap-id>

# Option 2: Delete unused detached volumes (verify first!)
pxctl volume list --status detached
# Confirm these are not needed
pxctl volume delete <volume-id>

# Option 3: Expand volumes that are undersized
# (reduces pool usage per-volume? No — this increases pool usage)
# Wrong direction — don't expand volumes when pool is full

# Option 4: Add a disk to the pool (correct fix)
pxctl service pool update --add /dev/sdc <pool-id>

# Option 5: Delete PVCs for deleted applications
kubectl get pvc -A | grep -v "Running\|Pending"  # find orphaned PVCs
```

---

## Scenario 5 — Node Down — Multiple Volumes Degraded

```
Symptom: Kubernetes node NotReady, pxctl shows 10+ volumes degraded simultaneously
Impact:  All volumes that had replicas on this node are degraded
```

### Investigation

```bash
# Step 1: Confirm the failure
kubectl get nodes
pxctl node list
# Identify the failed node

# Step 2: Assess recovery likelihood
# Is node rebooting or permanently failed?
# Check cloud provider console or hardware logs

# Step 3: Count affected volumes
pxctl volume list | grep -i degraded | wc -l

# Step 4: Check remaining replicas for each volume
pxctl volume list --json | jq '.[] | select(.replicationStatus == "Down") | {name, replicationStatus}'

# Step 5: Is the cluster still accepting writes?
# If repl=3 and only 1 node down, writes should still work (quorum=2)
# Test: create a small test volume
pxctl volume create --size=1 --repl=3 test-quorum-check
pxctl volume delete test-quorum-check
```

### Resolution

```bash
# Scenario A: Node will come back (rebooting)
# Wait up to 30 minutes before taking action
# Portworx will wait before triggering rebuild (configurable timeout)
watch pxctl node list
# When node returns: STATUS=OK, volumes auto-resync

# Scenario B: Node permanently failed
# Decommission immediately to trigger rebuild
pxctl node list  # get NODE_ID of failed node
pxctl service node-decommission --node <NODE_ID>
# Watch rebuild progress
watch "pxctl volume list | grep -i degraded | wc -l"
# Wait for count to reach 0

# Scenario C: Many rebuilds happening simultaneously
# Rebuilds are throttled to protect production I/O
# If you need faster rebuild (accepting I/O impact):
pxctl service options update replication_throttle_mb=500  # increase throttle
# After rebuild: reset to default
pxctl service options update replication_throttle_mb=100
```

---

## Scenario 6 — Network Partition / Split Brain

```
Symptom: Cluster splits into two groups, some nodes unreachable from others
Impact:  Potential split-brain: both halves think they are authoritative
```

### Investigation

```bash
# Step 1: Identify the partition
pxctl node list
# Nodes in same "partition" can communicate; nodes across cannot

# Step 2: Check KVDB quorum
pxctl service kvdb status
# If below quorum: cluster is read-only

# Step 3: Identify which partition has quorum (majority)
pxctl service kvdb members
# The partition with more KVDB members wins

# Step 4: Check volume state
pxctl volume list
# Volumes on the minority partition may show errors
```

### Resolution

```bash
# The minority partition (without quorum) must yield to majority

# Step 1: Stop PX on minority partition nodes
# (Prevents both halves from accepting writes)
kubectl -n kube-system delete pod portworx-<minority-node>  # emergency stop

# Step 2: Fix network partition
# (Physical network issue, firewall rule, cloud security group)

# Step 3: Verify network is healed
# Test connectivity between all nodes:
for node in <node1> <node2> <node3>; do
  kubectl debug node/$node -it --image=ubuntu -- \
    bash -c "ping -c 3 <other-node-ip>"
done

# Step 4: Restart PX on minority nodes
# (DaemonSet will restart automatically when pods are deleted)
kubectl get pod -n kube-system -l name=portworx -o wide

# Step 5: Verify KVDB quorum restored
pxctl service kvdb status

# Step 6: Check all volumes healthy
pxctl volume list | grep -v Up
```

---

## Scenario 7 — StorageClass Misconfiguration

```
Symptom: Volumes created with wrong parameters (wrong replication, wrong pool)
Impact:  Data durability lower than expected, performance not as designed
```

### Investigation

```bash
# Step 1: Find misconfigured volumes
# Example: volumes with repl=1 that should be repl=3
pxctl volume list --json | jq '.[] | select(.replicationFactor < 3) | {name, replicationFactor}'

# Step 2: Check the StorageClass
kubectl get storageclass | grep portworx
kubectl describe storageclass <sc-name>
# Read Parameters section

# Step 3: Check when volumes were created
pxctl volume list --json | jq '.[] | {name, created: .CreatedAt}'
# Compare to when SC was changed
```

### Resolution

```bash
# Fix existing misconfigured volumes:
# Update replication factor (online, no downtime)
pxctl volume ha-update --repl 3 <volume-id>

# Fix StorageClass for future volumes:
kubectl edit storageclass portworx-db
# Update repl: "3"

# Fix I/O profile (online update):
pxctl volume update --io_profile db <volume-id>
```

---

## Scenario 8 — CSI Failure — All New PVCs Pending

```
Symptom: All new PVC creations fail (Pending), existing volumes working fine
Impact:  No new storage can be provisioned
```

### Investigation

```bash
# Step 1: Check CSI driver
kubectl get csidrivers
kubectl get pod -n kube-system | grep -iE "csi|portworx"

# Step 2: Check CSI controller (external-provisioner)
CSI_CONTROLLER=$(kubectl get pod -n kube-system -l app=portworx -o wide | head -2 | tail -1 | awk '{print $1}')
kubectl logs -n kube-system $CSI_CONTROLLER -c external-provisioner | tail -100
# Look for: "Error provisioning volume", "Failed to call CSI"

# Step 3: Check PX API health
kubectl exec -n kube-system portworx-<any> -c portworx -- \
  /opt/pwx/bin/pxctl volume list
# If this fails, PX API is down

# Step 4: Check KVDB (provisioning requires KVDB write)
pxctl service kvdb status

# Step 5: Check events on a pending PVC
kubectl describe pvc <pending-pvc> -n <namespace>
```

### Root Causes

| Finding | Root Cause | Fix |
|---------|------------|-----|
| external-provisioner logs: "connection refused" | PX API down on leader node | Restart PX pod on leader node |
| "KVDB timeout" | KVDB degraded | Fix KVDB quorum first |
| "license expired" | PX license expired | Renew license |
| "grpc: context deadline exceeded" | PX overloaded | Check node resources |
| No events on PVC | external-provisioner not watching | Check external-provisioner pod |

---

## Postmortem Template

Every incident should produce a postmortem:

```markdown
# Postmortem: [Incident Title]

**Date**: 2026-06-18
**Duration**: 14:00 – 15:30 (1h30m)
**Severity**: P1 / P2 / P3
**Affected**: [list volumes, namespaces, applications]

## Summary

One paragraph describing what happened, what was impacted, and how it was resolved.

## Timeline

| Time  | Event |
|-------|-------|
| 14:00 | PagerDuty alert: PortworxVolumeDegraded |
| 14:05 | First responder: confirmed node failure via kubectl get nodes |
| 14:10 | Portworx decommission triggered |
| 14:30 | Rebuild in progress, volumes replicating |
| 15:30 | All volumes back to full replication |

## Root Cause

[What caused the incident? Be specific: "kernel bug on node3 caused OOM → PX process killed → node left cluster"]

## Impact

- Applications affected: [list]
- Data loss: None / [details]
- Recovery time: [duration]

## Contributing Factors

1. No alert on node memory pressure (allowed OOM to happen undetected)
2. Replication factor 2 instead of 3 (gave no headroom for one node loss)

## Action Items

| Action | Owner | Due |
|--------|-------|-----|
| Increase replication factor to 3 for all production volumes | Platform team | 2026-06-25 |
| Add OOM alert for Portworx nodes | SRE | 2026-06-20 |
| Runbook: node decommission procedure | Platform team | 2026-06-22 |

## Lessons Learned

1. What we did well:
2. What we could improve:
3. What we'll add to the runbook:
```

---

## Exercises

### Exercise 1 — Incident Simulation

Set up this scenario in your cluster and debug it:

```bash
# Setup: create 5 volumes with repl=3
for i in 1 2 3 4 5; do
  pxctl volume create --size=1 --repl=3 sim-vol-$i
done

# "Break" the cluster: pause one node
docker pause <worker-node>

# Now: diagnose and recover without looking at the guide
# Goal: get all 5 volumes back to full replication
```

### Exercise 2 — Log Reading

Given this PX log excerpt, identify what happened and what to do next:

```
time="2026-06-18T14:23:01Z" level=error msg="Failed to connect to KVDB" endpoint="etcd:http://10.0.0.5:2379" error="context deadline exceeded"
time="2026-06-18T14:23:01Z" level=warning msg="KVDB quorum lost, entering read-only mode"
time="2026-06-18T14:23:05Z" level=error msg="CreateVolume failed: KVDB unavailable"
time="2026-06-18T14:23:10Z" level=info msg="Volume vol-abc123 entering degraded state, replica on node-5 unreachable"
```

Questions:
1. What is the root cause?
2. Which scenario from this module does this match?
3. What is your first action?

### Exercise 3 — Write Three Alerts

Write Prometheus alerting rules for:
1. Any volume degraded for more than 10 minutes
2. Pool usage exceeds 80%
3. KVDB below quorum

---

## Interview Questions

1. A production database pod cannot start. It shows "MountVolume.MountDevice failed." Walk me through the next 10 minutes.
2. You are paged at 3AM. `pxctl alerts show` shows 15 volumes degraded. What do you do first?
3. A customer says "disk is full but `df` shows only 60% used." What is your diagnosis?
4. What is the first command you run when you get a Portworx PagerDuty alert?
5. A volume has been in "Replicating" state for 6 hours at 43%. What do you investigate?
6. What is the difference between a force-detach and a normal detach? When do you use force-detach?
7. How do you distinguish between a Kubernetes storage problem and a Portworx problem?
8. A PVC was accidentally deleted with reclaim policy Delete. What is your recovery path?

---

## Production Takeaways

1. **Start with scope, not commands** — before running any command, determine: is this one volume or many? One node or all nodes? This tells you where to look.

2. **`pxctl status` takes 3 seconds and saves 30 minutes** — always run it first. It tells you if the cluster is healthy before you chase symptoms.

3. **Events expire** — Kubernetes events expire after 1 hour by default. Run `kubectl describe pod/pvc` immediately. Don't wait.

4. **Force-detach is a last resort** — only use it when the node is confirmed dead and gone. Force-detach on a live node risks data corruption.

5. **Collect diags before you change anything** — `pxctl service diags -a` takes 2 minutes and captures the state before you start making changes. If your fix makes things worse, you have the original state.

6. **Postmortems prevent repeat incidents** — every P1/P2 incident needs a postmortem with action items. Without it, the same incident happens in 6 months.

---

## Success Criteria

You are production-ready when you can:

- [ ] Execute Scenario 1–8 investigations without referring to this guide
- [ ] Write the correct `pxctl` command for any situation from memory
- [ ] Distinguish Kubernetes-layer vs Portworx-layer failures in under 60 seconds
- [ ] Read PX log output and identify the failing component (Exercise 2)
- [ ] Write a complete postmortem after a simulated incident
- [ ] Answer all 8 interview questions cold

---

## Course Complete

You have finished all 12 modules.

```
What you can now do:

Disk fails       → you know what PX does and what you do
Node fails       → you know the rebuild path and timing
Volume Pending   → you know the 5-step investigation
Mount fails      → you map the event to the CSI call
Pool full        → you know the safe resolution order
Split brain      → you know which partition has quorum
Upgrade needed   → you use the Operator, not DaemonSet
DR required      → you choose async/sync/metro correctly
Backup required  → you design snapshot + PX-Backup strategy
```

Every Portworx incident now has a path. Follow the path.
