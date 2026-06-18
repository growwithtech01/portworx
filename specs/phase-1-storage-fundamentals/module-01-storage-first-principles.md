# Module 01 — Storage From First Principles

```
Phase       : 1 — Storage Fundamentals
Module      : 01
Status      : draft
Prereqs     : None
Estimated   : 4–6 hours
```

---

## Why This Module Exists

Every Portworx incident eventually reduces to a storage fundamentals question:

- Why is this volume slow? → IOPS, latency, throughput
- Why did the node lose data? → No redundancy, RAID not configured
- Why is the replica rebuild taking hours? → Network throughput, disk speed
- Why did the write hang? → Quorum loss, disk failure

If you skip this module, you will memorize commands. When production breaks, you will not know which command to run or why.

---

## Prerequisites

None. This is the starting point.

---

## Learning Objectives

After completing this module, you will be able to:

1. Explain why storage exists separately from RAM
2. Compare HDD, SSD, and NVMe — pick the right one for a workload
3. Describe block, file, and object storage and when to use each
4. Define IOPS, throughput, and latency — explain the triangle
5. Explain RAID 0, 1, 5, 6, 10 — describe failure scenarios for each
6. Design a storage redundancy strategy for a production database
7. Trace a storage failure from disk → OS → application

---

## Concepts to Study

### 1. Why Storage Exists

```
CPU ← executes instructions
RAM ← holds data while running (volatile, fast, expensive, small)
Disk ← holds data permanently (non-volatile, slow, cheap, large)
```

**Key insight**: RAM loses all data when power is cut. Disk does not. Every persistent workload needs disk.

Study:
- Von Neumann architecture (memory hierarchy)
- Why memory is a cache of disk
- What happens to a database when a pod restarts with no persistent volume

**Production incident pattern**:
> A developer deploys MySQL without a PVC. The pod crashes. All data gone. Why? Container filesystem is ephemeral — it lives in RAM/overlay, not on disk.

---

### 2. RAM vs Disk

| Property | RAM | SSD | HDD |
|----------|-----|-----|-----|
| Speed | nanoseconds | microseconds | milliseconds |
| Persistence | No | Yes | Yes |
| Cost per GB | $10+ | $0.10 | $0.02 |
| Capacity | GBs | TBs | TBs |

Study:
- Access time: RAM ~100ns, SSD ~100µs, HDD ~10ms
- Why databases use memory-mapped files
- What happens when a node runs out of RAM (OOM → swap → disk)

---

### 3. HDD vs SSD vs NVMe

**HDD (Hard Disk Drive)**

```
┌─────────────────────┐
│  Spinning Platter   │
│    ↕ Seek Time      │
│  Read/Write Head    │
└─────────────────────┘
Latency: 5–10ms
IOPS: 100–200
Throughput: 100–200 MB/s
```

- Sequential reads are fast (head stays in place)
- Random reads are slow (head must seek to new position)
- Good for: cold data, backups, sequential workloads
- Bad for: databases, random I/O

**SSD (Solid State Drive)**

```
┌─────────────────────┐
│  NAND Flash Cells   │
│  No moving parts    │
│  Random I/O ≈ Seq   │
└─────────────────────┘
Latency: 0.1–1ms
IOPS: 10,000–100,000
Throughput: 500 MB/s – 3 GB/s
```

- Random I/O nearly as fast as sequential
- Good for: databases, Kubernetes etcd, general workloads
- Bad for: very high write endurance (cells wear out)

**NVMe (Non-Volatile Memory Express)**

```
┌─────────────────────┐
│  PCIe Direct        │
│  No SATA bottleneck │
│  Parallelism        │
└─────────────────────┘
Latency: 0.02–0.1ms
IOPS: 500,000–1,000,000+
Throughput: 3–7 GB/s
```

- PCIe interface bypasses SATA/SAS bottleneck
- Good for: etcd, high-performance databases, Portworx journal disk
- Portworx recommends NVMe for journal disk

Study:
- SATA vs SAS vs NVMe interface differences
- Why Portworx recommends a separate NVMe disk for the journal
- Write amplification in SSDs

---

### 4. Block Storage

```
┌───────────────────────┐
│  Raw Disk Blocks      │
│  No filesystem        │
│  OS manages layout    │
└───────────────────────┘
```

- Disk exposed as raw blocks (512B or 4KB blocks)
- No inherent structure — the OS/filesystem imposes structure
- Lowest latency (no metadata overhead)
- Examples: AWS EBS, GCP Persistent Disk, Portworx volumes, iSCSI LUNs

**When to use**: Databases, Kubernetes PVs, anything where you need full control

Study:
- What a block device is (`/dev/sda`)
- How a filesystem sits on top of block storage
- Why databases sometimes use raw block devices (skip the filesystem)

---

### 5. File Storage

```
┌───────────────────────┐
│  NAS / NFS            │
│  Filesystem included  │
│  Shared access        │
└───────────────────────┘
```

- Storage with a filesystem already — you mount a directory, not a block
- Multiple clients can read/write simultaneously (NFS, SMB)
- Higher latency than block (network + NFS protocol overhead)
- Examples: NFS, AWS EFS, Azure Files

**When to use**: Shared data between pods, ReadWriteMany PVCs, CI/CD artifact storage

---

### 6. Object Storage

```
┌───────────────────────┐
│  Key → Value Store    │
│  HTTP API (GET/PUT)   │
│  No filesystem tree   │
└───────────────────────┘
```

- No filesystem — you PUT and GET objects by key via HTTP
- Massively scalable, cheap, durable (11 nines)
- Cannot be mounted like a filesystem natively
- Examples: S3, GCS, MinIO, Portworx cloud backups target object storage

**When to use**: Backups, logs, images, Portworx PX-Backup destinations

---

### 7. IOPS, Throughput, Latency — The Triangle

```
IOPS        = operations per second (reads + writes)
Throughput  = data moved per second (MB/s or GB/s)
Latency     = time for one operation to complete (ms/µs)
```

**The relationship**:

```
Throughput = IOPS × Block Size

Example:
100,000 IOPS × 4KB blocks = 400 MB/s
100,000 IOPS × 64KB blocks = 6.4 GB/s
```

**The tradeoff**:
- Small block size → high IOPS, low throughput per IOPS
- Large block size → low IOPS count, high throughput per IOPS
- You cannot maximize all three simultaneously

**Production patterns**:

| Workload | Block Size | Priority |
|----------|------------|----------|
| Database (random) | 4–8KB | IOPS + Latency |
| Streaming (sequential) | 64–256KB | Throughput |
| etcd | 4KB | Latency (microseconds) |
| Portworx replication | 4KB | IOPS + Latency |

Study:
- `iostat -x 1` — read await, util%, r/s, w/s
- What does it mean when `await` spikes?
- Difference between `rrqm/s` and `r/s`

---

### 8. RAID Concepts

RAID = Redundant Array of Independent Disks

**RAID 0 — Striping**

```
Data A → Disk 1
Data B → Disk 2
Data C → Disk 3
```

- Performance: 2× read/write speed
- Redundancy: None — one disk failure = total data loss
- Use: scratch data, not production

**RAID 1 — Mirroring**

```
Data → Disk 1
Data → Disk 2 (exact copy)
```

- Performance: read from either, write to both
- Redundancy: survives 1 disk failure
- Use: OS disk, boot disk, simple redundancy

**RAID 5 — Striping with Parity**

```
Data A  | Data B  | Parity(A+B) → across 3 disks
Data C  | Parity  | Data D
```

- Performance: good reads, slower writes (parity calculation)
- Redundancy: survives 1 disk failure
- Minimum: 3 disks
- Risk: if a disk fails and another fails during rebuild → total loss

**RAID 6 — Striping with Double Parity**

- Survives 2 simultaneous disk failures
- Minimum: 4 disks
- Slower writes than RAID 5

**RAID 10 — Mirror + Stripe**

```
[Disk 1 + Disk 2 mirror] stripe [Disk 3 + Disk 4 mirror]
```

- Best performance + redundancy combo
- Survives multiple failures (as long as mirrors survive)
- Most expensive (50% usable capacity)
- Use: production databases, Portworx underlying disks

**Portworx connection**: Portworx implements its own software-defined replication that works like RAID 1 but across nodes (not disks). Understanding RAID 1 makes Portworx replication factor immediately intuitive.

---

### 9. Storage Redundancy

Layers of redundancy:

```
Level 1: Disk redundancy       → RAID
Level 2: Controller redundancy → Dual HBA / dual path
Level 3: Node redundancy       → Portworx replication across nodes
Level 4: Zone redundancy       → Portworx metro DR
Level 5: Region redundancy     → Portworx async DR
```

Study:
- Single point of failure analysis
- Recovery Time Objective (RTO) vs Recovery Point Objective (RPO)
- How Portworx replication factor 3 means data lives on 3 nodes

---

### 10. Storage Failures

**Failure taxonomy**:

```
Disk failure      → bit rot, head crash, controller failure
Node failure      → hardware, kernel panic, power loss
Network failure   → split brain, partition
Human error       → wrong delete, misconfiguration
Firmware failure  → disk silent corruption
```

**Silent data corruption (bit rot)**:
- Data physically changes on disk but no error reported
- Portworx uses checksums to detect this
- ZFS is designed specifically to detect bit rot

**Write hole** (RAID 5 weakness):
- Power fails during a write
- Parity becomes inconsistent with data
- Next read returns corrupted data
- Fix: NVRAM-backed write journal (why Portworx uses a journal disk)

Study:
- `dmesg | grep -i error` after a disk failure
- SMART data: `smartctl -a /dev/sda`
- What happens to a RAID 5 array during rebuild if a second disk fails

---

## Hands-On Labs

### Lab 1.1 — Observe Disk Performance

```bash
# Install fio (I/O benchmarking tool)
sudo apt-get install fio

# Sequential read test
fio --name=seq-read \
    --rw=read \
    --bs=1M \
    --size=1G \
    --numjobs=1 \
    --iodepth=1 \
    --runtime=30 \
    --filename=/tmp/fio-test

# Random read test (IOPS focused)
fio --name=rand-read \
    --rw=randread \
    --bs=4k \
    --size=1G \
    --numjobs=4 \
    --iodepth=32 \
    --runtime=30 \
    --filename=/tmp/fio-test

# Observe: IOPS, BW (throughput), latency in output
```

**Expected outcome**: Understand the gap between sequential and random IOPS on your disk type.

### Lab 1.2 — Monitor Disk I/O in Real Time

```bash
# Watch disk stats every 1 second
iostat -x 1

# Fields to observe:
# r/s      → reads per second (IOPS)
# w/s      → writes per second (IOPS)
# rkB/s    → read throughput
# wkB/s    → write throughput
# await    → average wait time (latency)
# %util    → disk utilization (100% = saturated)
```

**Exercise**: Run the fio test from Lab 1.1 in one terminal, watch iostat in another. Correlate fio's reported IOPS with iostat's `r/s`.

### Lab 1.3 — Simulate Disk Failure with Loop Devices

```bash
# Create two virtual block devices (simulate disks)
dd if=/dev/zero of=/tmp/disk1.img bs=1M count=100
dd if=/dev/zero of=/tmp/disk2.img bs=1M count=100
losetup /dev/loop0 /tmp/disk1.img
losetup /dev/loop1 /tmp/disk2.img

# Create a RAID 1 array
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/loop0 /dev/loop1

# Check status
cat /proc/mdstat

# Write data
sudo mkfs.ext4 /dev/md0
sudo mount /dev/md0 /mnt/test
echo "important data" > /mnt/test/data.txt

# Simulate disk failure
sudo mdadm --fail /dev/md0 /dev/loop0
cat /proc/mdstat     # observe degraded state

# Verify data still accessible
cat /mnt/test/data.txt

# Cleanup
sudo umount /mnt/test
sudo mdadm --stop /dev/md0
sudo losetup -d /dev/loop0
sudo losetup -d /dev/loop1
```

---

## Exercises

### Exercise 1 — Workload Classification

For each workload below, choose the right storage type (Block/File/Object) and disk type (HDD/SSD/NVMe):

| Workload | Storage Type | Disk Type | Reasoning |
|----------|-------------|-----------|-----------|
| PostgreSQL primary | | | |
| Application logs archive | | | |
| Shared config files between pods | | | |
| Portworx journal disk | | | |
| etcd cluster | | | |
| Video backup storage | | | |
| Redis in-memory cache with AOF | | | |

### Exercise 2 — IOPS Math

A disk has:
- 10,000 IOPS
- 4KB block size
- 0.5ms average latency

1. What is the maximum throughput?
2. If you increase block size to 64KB, what happens to IOPS?
3. A database needs 50,000 IOPS. How many of these disks do you need?

### Exercise 3 — RAID Failure Scenarios

For each RAID level, describe exactly what happens when:

| RAID Level | 1 disk fails | 2 disks fail |
|------------|-------------|-------------|
| RAID 0 | | |
| RAID 1 | | |
| RAID 5 | | |
| RAID 6 | | |
| RAID 10 | | |

---

## Interview Questions

1. A production database is showing high `await` in `iostat`. Walk me through how you investigate.
2. What is the difference between IOPS and throughput? Give a scenario where you need high IOPS but low throughput.
3. Why does Portworx recommend a separate NVMe disk for the journal?
4. A developer says "we don't need RAID because Portworx replicates data." Is this correct?
5. What is write amplification in SSDs and why does it matter for Portworx?
6. A RAID 5 array has one failed disk. The operations team says "it's fine, we can replace it next week." What is the risk?
7. Explain bit rot. How does Portworx protect against it?
8. A new application writes 1000 small 4KB files per second. A second application reads a 1GB video file sequentially. Both go to the same disk. What happens?

---

## Production Takeaways

1. **NVMe for etcd and Portworx journal** — these are the most latency-sensitive writes in your cluster. Skimp here and you get cascading failures.

2. **`await > 10ms` on SSD is a red flag** — investigate immediately. It means disk is saturated or failing.

3. **RAID rebuild is the most dangerous time** — a second failure during rebuild is total data loss. Never run degraded arrays for more than 24 hours.

4. **Object storage is not a filesystem** — do not try to mount S3 as a POSIX filesystem for databases. Use it for backups and blobs only.

5. **IOPS are additive across disks** — Portworx can pool multiple disks per node and present them as a single pool. Understanding this makes pool expansion operations obvious.

6. **Silent corruption is real** — always enable checksums. Portworx does this internally.

---

## Success Criteria

You have completed this module when you can:

- [ ] Explain RAM vs Disk tradeoffs without notes
- [ ] Calculate throughput given IOPS and block size
- [ ] Describe RAID 0, 1, 5, 10 failure behavior for any failure scenario
- [ ] Choose block/file/object storage for any given workload
- [ ] Read `iostat -x` output and identify a saturated disk
- [ ] Explain why Portworx uses a journal disk and what disk type it should be
- [ ] Complete Lab 1.3 (RAID 1 simulation) independently

---

## Next Module

[Module 02 — Linux Storage](./module-02-linux-storage.md)
