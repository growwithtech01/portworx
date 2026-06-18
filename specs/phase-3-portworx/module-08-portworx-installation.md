# Module 08 — Portworx Installation

```
Phase       : 3 — Portworx
Module      : 08
Status      : draft
Prereqs     : Module 07 — Portworx Architecture
Estimated   : 4–6 hours (hands-on heavy)
```

---

## Why This Module Exists

Installation is where architecture becomes real. You will see exactly how Portworx integrates with Kubernetes — the Operator, the DaemonSet, the StorageClasses, the CSI registration. When you finish this module, you will have a working Portworx cluster and the ability to debug any installation that is not working.

---

## Learning Objectives

After completing this module, you will be able to:

1. Install the Portworx Operator using the spec-based approach
2. Configure a StorageCluster with correct disk and network settings
3. Verify the installation using `pxctl status` and Kubernetes health checks
4. Create StorageClasses for different workload profiles
5. Create and use a Portworx-backed PVC
6. Troubleshoot failed installations (common first-time failures)
7. Uninstall Portworx cleanly

---

## Prerequisites

- Module 07 — Portworx Architecture
- `kubectl` configured for a cluster
- Minimum cluster requirements: 3 worker nodes, 8GB RAM each, one available disk per node
- Portworx Essentials license (free) or PX-Developer license

---

## Concepts to Study

### 1. Installation Approaches

**Approach 1: Portworx Central / Spec Generator**

The primary supported installation method is the Portworx Spec Generator at `central.portworx.com`:
- Select Kubernetes version, cloud provider, disk configuration
- Downloads a generated YAML spec
- Apply with `kubectl apply -f <spec.yaml>`

**Approach 2: Operator + StorageCluster**

Manual installation via Helm or YAML — deploy the Operator first, then create the StorageCluster CR.

**Approach 3: OLM (Operator Lifecycle Manager)**

For OpenShift clusters, Portworx is installed via OLM.

---

### 2. Minimum Requirements

```
Nodes:
  - Minimum 3 worker nodes (for KVDB quorum)
  - 4 vCPUs per node
  - 8GB RAM per node
  - One or more raw (unformatted) block devices per node
    OR empty directories that Portworx can use

Disk:
  - Portworx storage disks: SSD recommended (NVMe best)
  - Journal disk: NVMe recommended (can be auto-selected)
  - Do NOT use the OS disk

Network:
  - Management network: Portworx API, control plane
  - Data network: replication traffic (high bandwidth)
  - Recommend separate NICs for data traffic

Kubernetes:
  - 1.18+ (1.24+ recommended)
  - etcd healthy (Portworx uses KVDB, not Kubernetes etcd, but cluster must be stable)

Ports:
  - 9001–9022: Portworx API and replication
  - 17001–17020: internal communication
```

---

### 3. Portworx Operator Installation

```bash
# Step 1: Apply the Portworx Operator
kubectl apply -f 'https://install.portworx.com/2.x/px-operator.yaml'

# Verify operator is running
kubectl get deployment -n kube-system portworx-operator

# Step 2: Create the StorageCluster
# (generated from Portworx Spec Generator or manual below)
```

---

### 4. StorageCluster Configuration

Key fields in StorageCluster spec:

```yaml
apiVersion: core.libopenstorage.org/v1
kind: StorageCluster
metadata:
  name: portworx
  namespace: kube-system
spec:
  image: portworx/oci-monitor:3.1.x

  # KVDB — where cluster metadata is stored
  kvdb:
    internal: true                    # use built-in etcd (3 KVDB nodes auto-selected)
    # external:                       # OR use external etcd
    #   endpoints:
    #   - etcd:http://etcd-svc:2379

  # Storage configuration
  storage:
    useAll: true                      # use all available unformatted disks
    # useAllWithPartitions: true      # even use disks with partitions
    # devices:                        # OR specify exact devices
    # - /dev/sdb
    # - /dev/sdc
    journalDevice: auto               # auto-select fastest disk for journal

  # Network interfaces
  network:
    dataInterface: eth0               # high-bandwidth interface for replication
    mgmtInterface: eth0               # management interface

  # Stork (storage-aware scheduler)
  stork:
    enabled: true
    args:
      verbose: "true"

  # Autopilot (automatic capacity management)
  autopilot:
    enabled: true

  # Node selector (install only on labeled nodes)
  # nodes:
  # - selector:
  #     matchLabels:
  #       storage: portworx
```

---

## Hands-On Labs

### Lab 8.1 — Install Portworx on Kind (Development)

> **Note**: Kind does not support Portworx fully in production mode. This lab uses the Portworx OCI-monitor in developer mode with loop devices. For real clusters, use Lab 8.2.

```bash
# Create a Kind cluster with multiple nodes
cat > /tmp/kind-config.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF

kind create cluster --config /tmp/kind-config.yaml --name px-lab

# Verify 4 nodes
kubectl get nodes

# Label worker nodes for Portworx
kubectl label nodes px-lab-worker px-lab-worker2 px-lab-worker3 \
  portworx.io/node=true

# Create disk images for Portworx (loop device backing)
# Do this on each worker node
for node in px-lab-worker px-lab-worker2 px-lab-worker3; do
  docker exec $node bash -c "
    mkdir -p /var/lib/portworx/disks
    dd if=/dev/zero of=/var/lib/portworx/disks/px-disk.img bs=1M count=10240
    losetup /dev/loop5 /var/lib/portworx/disks/px-disk.img
  "
done

# Verify loop devices
for node in px-lab-worker px-lab-worker2 px-lab-worker3; do
  echo "=== $node ==="; docker exec $node lsblk | grep loop5
done

# Install Portworx Operator (use PX Essentials - free tier)
# Get spec from: https://central.portworx.com
# Select: New Spec → Kubernetes → Portworx Essentials
# OR apply pre-built operator:
kubectl apply -f https://install.portworx.com/2.x/px-operator.yaml

kubectl wait --for=condition=Available deployment/portworx-operator \
  -n kube-system --timeout=120s

# Apply StorageCluster for Kind/dev environment
kubectl apply -f - <<EOF
apiVersion: core.libopenstorage.org/v1
kind: StorageCluster
metadata:
  name: portworx
  namespace: kube-system
  annotations:
    portworx.io/is-gke: "false"
spec:
  image: portworx/oci-monitor:3.1.4
  imagePullPolicy: Always
  kvdb:
    internal: true
  storage:
    devices:
    - /dev/loop5
    journalDevice: auto
  network:
    dataInterface: eth0
    mgmtInterface: eth0
  stork:
    enabled: true
  nodeSelector:
    matchLabels:
      portworx.io/node: "true"
EOF

# Watch installation progress
kubectl get pods -n kube-system -l name=portworx -w
# This takes 5-10 minutes on first install (image pull)
```

### Lab 8.2 — Verify Portworx Installation

```bash
# After all PX pods are Running (not just starting):

# Step 1: Check cluster status
kubectl exec -n kube-system portworx-<any-node> -c portworx -- /opt/pwx/bin/pxctl status

# Expected output:
# Status: PX is operational
# License: Trial/Essentials/Enterprise
# Node count: 3
# Cluster ID: ...
# Global storage pool:
#   Total capacity: 30 GiB (3 nodes × 10 GiB)
#   Used: 0 GiB

# Step 2: Check node status
kubectl exec -n kube-system portworx-<any-node> -c portworx -- /opt/pwx/bin/pxctl node list

# All nodes should show STATUS: OK

# Step 3: Check KVDB
kubectl exec -n kube-system portworx-<any-node> -c portworx -- \
  /opt/pwx/bin/pxctl service kvdb status

# Step 4: Check CSI driver registered
kubectl get csidrivers | grep portworx

# Step 5: Check Stork
kubectl get pod -n kube-system -l name=stork

# Step 6: Check StorageCluster status
kubectl get storagecluster -n kube-system
kubectl describe storagecluster portworx -n kube-system | grep -A 5 "Conditions:"
```

### Lab 8.3 — Create StorageClasses

```bash
# StorageClass 1: General purpose (replication 2)
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: portworx-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: pxd.portworx.com
parameters:
  repl: "2"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF

# StorageClass 2: Database (high IOPS, replication 3)
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: portworx-db
provisioner: pxd.portworx.com
parameters:
  repl: "3"
  io_profile: "db"
  priority_io: "high"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF

# StorageClass 3: Shared (ReadWriteMany — NFS-based)
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: portworx-shared
provisioner: pxd.portworx.com
parameters:
  repl: "2"
  sharedv4: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF

# Verify
kubectl get storageclass
```

### Lab 8.4 — Create and Use a Portworx PVC

```bash
# Create a PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: px-test-pvc
spec:
  storageClassName: portworx-sc
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
EOF

# Watch PVC become Bound
kubectl get pvc px-test-pvc -w

# Once Bound, inspect the PV
PV_NAME=$(kubectl get pvc px-test-pvc -o jsonpath='{.spec.volumeName}')
kubectl describe pv $PV_NAME

# Check Portworx volume details
VOLUME_ID=$(kubectl get pv $PV_NAME -o jsonpath='{.spec.csi.volumeHandle}')
kubectl exec -n kube-system portworx-<node> -c portworx -- \
  /opt/pwx/bin/pxctl volume inspect $VOLUME_ID

# Create a pod to use it
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: px-test-pod
spec:
  schedulerName: stork
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "dd if=/dev/urandom of=/data/test bs=1M count=100 && echo done && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: px-test-pvc
EOF

kubectl wait --for=condition=Ready pod/px-test-pod --timeout=120s

# Check which node the pod landed on
kubectl get pod px-test-pod -o wide

# Verify volume is on that same node (hyperconverged)
kubectl exec -n kube-system portworx-<node> -c portworx -- \
  /opt/pwx/bin/pxctl volume inspect $VOLUME_ID | grep -A 5 "Replication Status"

# Cleanup
kubectl delete pod px-test-pod
kubectl delete pvc px-test-pvc
```

### Lab 8.5 — Troubleshoot a Failed PX Pod

```bash
# Find a failed PX pod
kubectl get pod -n kube-system -l name=portworx | grep -v Running

# Step 1: Describe the pod
kubectl describe pod <failed-px-pod> -n kube-system

# Step 2: Check previous container logs (if restarting)
kubectl logs <failed-px-pod> -n kube-system -c portworx --previous

# Step 3: Check events
kubectl get events -n kube-system --sort-by='.lastTimestamp' | grep portworx

# Step 4: SSH to the node and check pxctl
NODE=$(kubectl get pod <failed-px-pod> -n kube-system -o jsonpath='{.spec.nodeName}')
kubectl debug node/$NODE -it --image=ubuntu -- bash
# Inside: nsenter -t 1 -m -u -i -n -p -- pxctl status

# Step 5: Check if disks are available
kubectl debug node/$NODE -it --image=ubuntu -- bash
# Inside: nsenter -t 1 -m -u -i -n -p -- lsblk

# Step 6: Check PX process
kubectl debug node/$NODE -it --image=ubuntu -- bash
# Inside: nsenter -t 1 -m -u -i -n -p -- ps aux | grep portworx
```

---

## Debugging Reference

### Common Installation Failures

**1. PX pod CrashLoopBackOff**

```bash
kubectl logs portworx-<node> -n kube-system -c portworx --previous | tail -50

# Common causes:
# a) No available disks
#    Look for: "No available disks found"
#    Fix: Check lsblk on node, ensure disk is unformatted

# b) KVDB startup failure
#    Look for: "KVDB failed to start"
#    Fix: Check if ports 9001-9022 are open between nodes

# c) License validation failure
#    Look for: "License not found" or "Failed to validate license"
#    Fix: Check portworx.io token in StorageCluster spec
```

**2. PVC stuck in Pending**

```bash
kubectl describe pvc <pvc-name>

# Common cause: WaitForFirstConsumer — PVC waits for pod
# Solution: Create pod first (not a bug!)

# Another cause: No free nodes in cluster
pxctl status | grep "Used"

# Another cause: StorageClass parameters invalid
kubectl get storageclass <name> -o yaml
```

**3. PX node shows "StorageDown"**

```bash
pxctl status
# Look for: "STATUS: StorageDown"

pxctl service diags -a
# Collect diags and check storage pool errors

# Check disk health
smartctl -a /dev/sdb
dmesg | grep -i error | tail -20
```

---

## Interview Questions

1. What are the minimum hardware requirements to install Portworx?
2. Why does Portworx need unformatted disks? What happens if you give it a formatted disk?
3. What is the difference between internal KVDB and external etcd for Portworx?
4. What does `WaitForFirstConsumer` do in a Portworx StorageClass?
5. How do you verify a Portworx installation is healthy? Name 5 checks.
6. A PX pod is in CrashLoopBackOff. Walk me through your first 5 debug steps.
7. What is the difference between the `portworx-sc` (general) and `portworx-db` StorageClasses you created?
8. What does `io_profile: db` do in a Portworx StorageClass?

---

## Production Takeaways

1. **Use Portworx Spec Generator for production installs** — it generates a spec tested against your specific Kubernetes version and cloud provider.

2. **Separate data and management networks** — in production, put replication traffic on a dedicated high-bandwidth network interface.

3. **Journal disk on NVMe** — use `journalDevice: /dev/nvme0n1` (not auto) in production so you know exactly which disk is the journal.

4. **Always use `Retain` reclaimPolicy in production StorageClasses** — mistake-proof your cluster against accidental data deletion.

5. **Monitor Portworx with Prometheus + Grafana** — Portworx ships Prometheus metrics. Set up alerting on node health, volume health, and capacity before going to production.

6. **Internal KVDB node count** — with 3 worker nodes, you get 3 KVDB members. Add more nodes to get 5 KVDB members for higher availability.

---

## Success Criteria

You have completed this module when you can:

- [ ] Install the Portworx Operator independently (no looking at notes)
- [ ] Create a StorageCluster spec with correct disk and network configuration
- [ ] Verify installation with all 5 health checks from Lab 8.2
- [ ] Create 3 different StorageClasses for different workload types
- [ ] Create a PVC, verify it is bound, and run a pod using it (Lab 8.4)
- [ ] Debug a CrashLoopBackOff PX pod using only logs and kubectl

---

## Next Module

[Module 09 — Portworx Data Plane](./module-09-portworx-data-plane.md)
