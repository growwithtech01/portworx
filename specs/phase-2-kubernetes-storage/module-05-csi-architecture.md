# Module 05 — CSI Architecture

```
Phase       : 2 — Kubernetes Storage
Module      : 05
Status      : draft
Prereqs     : Module 04 — Persistent Volumes Deep Dive
Estimated   : 4–5 hours
```

---

## Why This Module Exists

CSI (Container Storage Interface) is the bridge between Kubernetes and Portworx. Every Portworx volume operation — create, attach, mount, snapshot, expand — is a CSI call. When a volume is stuck, you are debugging a CSI call.

If you don't understand CSI, you cannot debug Portworx incidents at depth. You will be guessing at log messages instead of knowing exactly which RPC failed and why.

---

## Learning Objectives

After completing this module, you will be able to:

1. Explain why CSI was created and what problem it solves
2. Describe the three CSI components: Identity, Controller, Node
3. Trace the full request path for PVC creation, pod scheduling, and pod deletion
4. Identify which CSI gRPC call is failing from log output
5. Inspect CSI driver registration and health in a cluster
6. Explain how Portworx implements each CSI operation
7. Debug stuck volume attachments and failed mounts using CSI logs

---

## Prerequisites

- Module 04 — Persistent Volumes Deep Dive
- Kubernetes cluster (Kind or cloud)

---

## Concepts to Study

### 1. Before CSI — The Problem

Before CSI (before Kubernetes 1.13), storage drivers were built into the Kubernetes core binary (`in-tree` plugins):

```
Kubernetes binary
├── AWS EBS driver code
├── GCE PD driver code
├── Azure Disk driver code
├── Ceph driver code
├── GlusterFS driver code
└── ... 30+ storage drivers
```

**Problems**:
- Adding a new storage system required a PR to Kubernetes core
- Bug fixes required a new Kubernetes release
- Storage vendors were at the mercy of Kubernetes release cycles
- Security vulnerabilities in one driver affected all clusters
- Testing burden on Kubernetes maintainers

**Portworx pre-CSI**: Portworx had an in-tree driver that had to be submitted to and maintained in the Kubernetes repo.

---

### 2. After CSI — The Solution

CSI defines a standard gRPC API between the orchestrator (Kubernetes) and the storage driver:

```
Kubernetes (orchestrator)          Storage Driver (out-of-tree)
─────────────────────────          ────────────────────────────
      gRPC calls          →        CSI Plugin (e.g., Portworx)
   CreateVolume           →        Create Portworx volume
   AttachVolume           →        Attach to node
   MountVolume            →        Mount into pod
```

**Benefits**:
- Storage vendors ship their own driver, independent of Kubernetes releases
- Kubernetes only calls the standard API — doesn't care about the implementation
- Drivers can be updated without cluster downtime
- Portworx ships as a separate container, not baked into Kubernetes

---

### 3. CSI Components

Every CSI driver has three services:

#### Identity Service

```
RPCs:
  GetPluginInfo         → driver name and version
  GetPluginCapabilities → what this driver can do (snapshots? expansion? etc.)
  Probe                 → is the driver healthy?
```

- Every CSI plugin implements this
- Kubernetes uses it to verify the driver is alive and discover capabilities

#### Controller Service

```
RPCs:
  CreateVolume           → create a new volume on the storage backend
  DeleteVolume           → delete a volume from the storage backend
  ControllerPublishVolume → attach volume to a node (like EBS attach)
  ControllerUnpublishVolume → detach volume from a node
  ValidateVolumeCapabilities → verify access modes are supported
  ListVolumes            → list all volumes
  CreateSnapshot         → create a snapshot
  DeleteSnapshot         → delete a snapshot
  ExpandVolume           → expand volume capacity
```

- Runs as a **Deployment** (one replica, or a few with leader election)
- Does not need to run on every node
- Talks to the storage backend API (e.g., Portworx API)
- Kubernetes components that talk to it: `external-provisioner`, `external-attacher`, `external-snapshotter`

#### Node Service

```
RPCs:
  NodeStageVolume    → format and mount volume at a global staging path
  NodeUnstageVolume  → unmount from staging path
  NodePublishVolume  → bind mount from staging path into pod directory
  NodeUnpublishVolume → unmount from pod directory
  NodeGetCapabilities → what node operations are supported
  NodeGetInfo        → node topology (zone, region, etc.)
```

- Runs as a **DaemonSet** — one pod per node
- Must run on every node because mounts happen locally
- Does not talk to storage backend API — operates on local kernel mounts
- Talks to: `kubelet` (via gRPC socket at `/var/lib/kubelet/plugins/<driver-name>/`)

---

### 4. CSI Sidecar Containers

CSI drivers ship with Kubernetes-maintained sidecar containers that translate Kubernetes API calls into CSI gRPC calls:

```
external-provisioner    → watches for PVCs, calls CreateVolume
external-attacher       → watches for VolumeAttachment, calls ControllerPublishVolume
external-snapshotter    → watches for VolumeSnapshots, calls CreateSnapshot
external-resizer        → watches for PVC resize, calls ControllerExpandVolume
node-driver-registrar   → registers CSI driver with kubelet
liveness-probe          → calls Probe() to check driver health
```

**Portworx CSI architecture**:

```
portworx-api pod (Controller)
├── portworx container          ← the Portworx daemon (CSI controller service)
├── external-provisioner        ← creates/deletes volumes (calls Portworx)
├── external-attacher           ← attaches/detaches volumes
└── external-snapshotter        ← creates/deletes snapshots

portworx-<nodename> pod (Node — DaemonSet)
├── portworx container          ← Portworx daemon (CSI node service + data plane)
├── node-driver-registrar       ← registers with kubelet
└── liveness-probe              ← checks health
```

---

### 5. Full Request Trace — PVC Creation to Pod Mount

```
Step 1: Developer applies PVC
        kubectl apply -f pvc.yaml
                ↓
Step 2: external-provisioner (sidecar) watches for unbound PVCs
        Detects new PVC → calls CSI Controller: CreateVolume(size=10Gi, params={repl:3})
                ↓
Step 3: Portworx controller service receives CreateVolume
        Creates Portworx volume on storage backend
        Returns volume ID
                ↓
Step 4: external-provisioner creates PV object in Kubernetes
        PV bound to PVC
        PVC status: Bound
                ↓
Step 5: Developer applies Pod referencing PVC
        Scheduler assigns pod to node X
                ↓
Step 6: external-attacher watches VolumeAttachment object
        Calls CSI Controller: ControllerPublishVolume(volumeId, nodeId=X)
        Portworx: sets node X as the "primary" for this volume
                ↓
Step 7: kubelet on node X detects pod needs volume
        Calls CSI Node: NodeStageVolume(volumeId, stagingPath=/var/lib/kubelet/plugins/kubernetes.io/csi/pv/<pv-name>/globalmount/)
        Portworx node plugin: formats volume if needed, mounts at staging path
                ↓
Step 8: kubelet calls CSI Node: NodePublishVolume(volumeId, targetPath=/var/lib/kubelet/pods/<uid>/volumes/kubernetes.io~csi/<pv-name>/mount/)
        Portworx: bind mounts from staging path to pod-specific path
                ↓
Step 9: Container starts with volume at mountPath: /data
```

**On pod deletion (reverse flow)**:

```
Container stops
        ↓
kubelet calls CSI Node: NodeUnpublishVolume (remove bind mount from pod dir)
        ↓
kubelet calls CSI Node: NodeUnstageVolume (unmount from staging path)
        ↓
external-attacher calls CSI Controller: ControllerUnpublishVolume (detach from node)
        ↓
VolumeAttachment object deleted
        ↓
PV remains, PVC remains, Portworx volume remains
```

---

### 6. CSI Topology

CSI topology tells Kubernetes where volumes can be placed:

```
NodeGetInfo returns topology labels:
{
  "topology.kubernetes.io/zone": "us-east-1a",
  "topology.portworx.io/node": "node1"
}
```

With `WaitForFirstConsumer` StorageClass, Kubernetes:
1. Schedules pod to a node
2. Gets node's topology
3. Calls `CreateVolume` with topology hint: "create this volume accessible from zone us-east-1a"
4. Portworx creates the volume with replicas in that zone

This is how zone-awareness works — the CSI driver, not Kubernetes, understands zone topology.

---

### 7. CSI Driver Registration

The `node-driver-registrar` sidecar registers the CSI plugin with kubelet:

```
Registration process:
1. node-driver-registrar starts on node
2. Calls GetPluginInfo() on the CSI driver socket
3. Creates registration socket at /var/lib/kubelet/plugins_registry/<driver-name>/
4. kubelet detects the registration socket
5. kubelet knows: "driver pxd.portworx.com lives at /var/lib/kubelet/plugins/pxd.portworx.com/csi.sock"
6. kubelet uses this socket for NodeStageVolume, NodePublishVolume calls
```

```bash
# Verify CSI driver is registered
kubectl get csidrivers
# NAME                    ATTACHREQUIRED   PODINFOONMOUNT   STORAGECAPACITY   ...
# pxd.portworx.com        true             false            false

# Check CSI driver capabilities
kubectl describe csidriver pxd.portworx.com
```

---

## Hands-On Labs

### Lab 5.1 — Inspect CSI Drivers in a Cluster

```bash
kind create cluster

# Kind comes with a local-path provisioner, not a full CSI driver
# Let's install a real CSI driver (democratic-csi with hostPath for lab)

# First, see what CSI drivers exist
kubectl get csidrivers

# View the CSI node sockets (on the kind node)
docker exec -it kind-control-plane ls /var/lib/kubelet/plugins/

# Install NFS CSI driver (for demonstration)
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system \
  --set controller.replicas=1

# Observe the DaemonSet (node plugin)
kubectl get daemonset -n kube-system | grep nfs

# Observe the Deployment (controller plugin)
kubectl get deployment -n kube-system | grep nfs

# Check CSI driver registered
kubectl get csidrivers

# Inspect controller pod sidecar containers
kubectl get pod -n kube-system -l app=csi-nfs-controller -o jsonpath='{.items[0].spec.containers[*].name}'
# Output: nfs external-provisioner external-snapshotter external-resizer liveness-probe

# Inspect node pod sidecar containers
kubectl get pod -n kube-system -l app=csi-nfs-node -o jsonpath='{.items[0].spec.containers[*].name}'
# Output: liveness-probe node-driver-registrar nfs

kind delete cluster
```

### Lab 5.2 — Trace a CSI Volume Creation

```bash
kind create cluster

# Enable CSI plugin logging (for local-path provisioner)
# Watch provisioner logs while creating a PVC
kubectl logs -n local-path-storage deployment/local-path-provisioner -f &
LOGPID=$!

# Create a PVC and watch the log
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-trace-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Observe provisioner log: you should see volume creation events
sleep 5
kill $LOGPID

# Describe the PVC
kubectl describe pvc csi-trace-pvc

# Find the PV it created
kubectl get pv

# Look at VolumeAttachment (created when pod mounts the volume)
kubectl get volumeattachment

# Create a pod and watch attachment
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: csi-trace-pod
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: csi-trace-pvc
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: data
      mountPath: /data
EOF

# Watch VolumeAttachment object appear
kubectl get volumeattachment -w &
WATCHPID=$!
kubectl wait --for=condition=Ready pod/csi-trace-pod --timeout=60s
sleep 5
kill $WATCHPID

# Check mount inside the node
docker exec -it kind-control-plane \
  ls /var/lib/kubelet/pods/$(kubectl get pod csi-trace-pod -o jsonpath='{.metadata.uid}')/volumes/

# Cleanup
kubectl delete pod csi-trace-pod
kubectl delete pvc csi-trace-pvc
kind delete cluster
```

### Lab 5.3 — Debug a CSI Node Mount Failure (Simulation)

```bash
kind create cluster

# Create PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: debug-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

kubectl wait --for=jsonpath='{.status.phase}'=Bound pvc/debug-pvc --timeout=60s

# Create pod with wrong mount path permissions (simulate issue)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: debug-pvc
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls /data && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
EOF

# Observe events
kubectl describe pod debug-pod
kubectl get events --sort-by='.lastTimestamp'

# Check what happened on the node
docker exec -it kind-control-plane dmesg | tail -20

# Cleanup
kubectl delete pod debug-pod
kubectl delete pvc debug-pvc
kind delete cluster
```

---

## Debugging Reference

### Identify the Failing CSI Call

```bash
# Pod stuck in ContainerCreating — check events first
kubectl describe pod <pod-name>

# Look for events like:
# "AttachVolume.Attach failed for volume..."
# "MountVolume.MountDevice failed..."
# "MountVolume.SetUp failed..."

# Map event to CSI call:
# AttachVolume.Attach → ControllerPublishVolume
# MountVolume.MountDevice → NodeStageVolume
# MountVolume.SetUp → NodePublishVolume

# Find the CSI controller pod
kubectl get pod -n <portworx-namespace> | grep -i csi-controller

# Check CSI controller logs
kubectl logs -n <portworx-namespace> <csi-controller-pod> -c portworx
kubectl logs -n <portworx-namespace> <csi-controller-pod> -c external-provisioner
kubectl logs -n <portworx-namespace> <csi-controller-pod> -c external-attacher

# Find the CSI node pod on the failing node
NODE=$(kubectl get pod <pod-name> -o jsonpath='{.spec.nodeName}')
kubectl get pod -n <portworx-namespace> -o wide | grep $NODE

# Check CSI node plugin logs
kubectl logs -n <portworx-namespace> <csi-node-pod> -c portworx
```

### Check CSI Driver Registration

```bash
# Is the CSI driver registered?
kubectl get csidrivers

# Is the node plugin running on every node?
kubectl get daemonset -n <portworx-namespace>
kubectl get pod -n <portworx-namespace> -o wide | grep -i node

# Check node plugin socket exists on the node
kubectl debug node/<node-name> -it --image=busybox -- \
  ls /host/var/lib/kubelet/plugins/pxd.portworx.com/

# Check kubelet can reach CSI socket
kubectl logs -n kube-system kube-apiserver-<node> | grep csi
```

---

## Interview Questions

1. What problem did CSI solve? What was the alternative before CSI?
2. What is the difference between the CSI Controller service and the CSI Node service? Where does each run?
3. Walk me through every CSI call made when a pod with a PVC is created.
4. What does `external-provisioner` do? Is it part of Portworx?
5. A pod is stuck in ContainerCreating with error "MountVolume.MountDevice failed." Which CSI call failed?
6. What is `node-driver-registrar` and why is it a sidecar rather than a separate deployment?
7. How does CSI topology enable zone-aware volume placement?
8. What is a VolumeAttachment object? Who creates it?

---

## Production Takeaways

1. **CSI controller runs once; CSI node runs everywhere** — if the CSI node DaemonSet has a failing pod on one node, only volumes on that node are affected. Check DaemonSet health regularly.

2. **`kubectl describe pod` maps directly to CSI calls** — "MountVolume.SetUp failed" = `NodePublishVolume` failed. Know this mapping; it saves 10 minutes per incident.

3. **VolumeAttachment stuck = ControllerPublishVolume stuck** — if a volume attachment is not completing, check the CSI controller logs for the `ControllerPublishVolume` call.

4. **CSI socket path is sacred** — `/var/lib/kubelet/plugins/<driver-name>/csi.sock` must exist for kubelet to call the node plugin. If this socket is missing, no volumes can mount on that node.

5. **Sidecar container versions matter** — `external-provisioner` v3.x has different capabilities than v2.x. Upgrading Portworx requires compatible sidecar versions.

6. **CSI driver logs are verbose at debug level** — in production, CSI logs at info level are usually sufficient. Don't set debug in production without log rotation.

---

## Success Criteria

You have completed this module when you can:

- [ ] Explain why CSI was created without looking at notes
- [ ] Draw the Controller + Node + Sidecar architecture from memory
- [ ] Trace every CSI call in the PVC create → pod mount → pod delete lifecycle
- [ ] Identify which CSI call is failing from `kubectl describe pod` events
- [ ] Complete Lab 5.2 and explain each event in the provisioner log
- [ ] Explain what happens to CSI calls when the CSI controller pod restarts

---

## Next Module

[Module 06 — Stateful Applications](./module-06-stateful-applications.md)
