# Module 01 Lab — Solutions, Explanations & Thinking

```
Read this AFTER attempting each question yourself.
For each solution: Command → Output → Why it works → Portworx connection
```

---

## Section A — Observe

---

### Solution Q1 — What Disks Exist Inside This Container?

**Commands**:

```bash
# Primary command: block device tree
lsblk

# With filesystem info
lsblk -f

# With all columns
lsblk -o NAME,MAJ:MIN,RM,SIZE,RO,TYPE,MOUNTPOINTS
```

**Expected output inside Docker Ubuntu**:

```
NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
vda     252:0    0    59G  0 disk
└─vda1  252:1    0    59G  0 part /
```

OR on some Docker setups:

```
NAME  MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda     8:0   0   59G  0 disk
└─sda1  8:1   0   59G  0 part /
```

**How to read this output**:

```
Column: TYPE
  disk       → the actual block device (the "hard drive")
  part       → a partition carved from the disk
  loop       → a loop device (file-backed virtual disk)
  lvm        → logical volume (LVM)
  md         → software RAID device

Column: MAJ:MIN
  8:0  → SCSI disk (sd*)
  252:0 → virtio disk (vd*) — common in VMs and Docker
  7:0   → loop device

Column: RM
  0 → not removable
  1 → removable (USB, CD)
```

**Answer to the question**:

The container does NOT have a real disk. It sees a virtual disk (`vda` or `sda`) that is actually a disk image file managed by Docker Desktop's internal Linux VM. The structure is:

```
Your Mac SSD (physical)
     ↓
Docker Desktop VM (runs on Mac using hypervisor)
     ↓  creates a virtual disk image file
/dev/vda inside the VM (59GB virtual disk)
     ↓  Docker overlay filesystem
Container root / → overlay on top of vda1
```

**Portworx connection**: When Portworx starts on a Kubernetes node, `pxctl status` shows node storage using the same `lsblk` data. Understanding this output is how you verify which physical disks Portworx claimed for its storage pool.

---

### Solution Q2 — How Much Space Does the Container Have?

**Commands**:

```bash
# Filesystem usage (human readable)
df -h

# Find largest directories
du -sh /* 2>/dev/null | sort -rh | head -20

# What filesystem type is the root?
df -T /
# OR
mount | grep "on / "

# Find overlay mount details
mount | grep overlay
```

**Expected output**:

```
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          59G  4.5G   52G   8% /
tmpfs            64M     0   64M   0% /dev
...

$ df -T /
Filesystem     Type  1K-blocks    Used Available Use% Mounted on
overlay        overlay  61896700 4718476  54015452   8% /

$ mount | grep overlay
overlay on / type overlay (rw,relatime,lowerdir=...,upperdir=...,workdir=...)
```

**Key insight — the word "overlay"**:

```
Overlay filesystem is a union filesystem.
It stacks multiple directories and presents them as one.

Container sees: /  (looks like a normal root filesystem)
Reality:
  lowerdir = Docker image layers (read-only, shared between containers)
  upperdir = This container's write layer (read-write, unique to this container)
  workdir  = Internal working directory for overlay

When you write to / inside the container:
  → Write goes to upperdir (container-specific)
  → lowerdir layers are never modified
  → On container deletion: upperdir is deleted, lowerdir remains for other containers
```

**Answer**: When you write a 1GB file inside the container, it goes to the `upperdir` on the Docker VM's virtual disk (`/dev/vda`), which is a file on your Mac's SSD.

**Portworx connection**: This is EXACTLY the problem PVCs solve. A Portworx volume bypasses the overlay filesystem and gives you a real block device with a real filesystem — so data survives container restarts.

---

### Solution Q3 — Container Storage vs Host Storage

**Commands**:

```bash
# Inside container
df -h /

# On Mac (second terminal — NOT inside container)
df -h /

# Check Docker VM settings
# Docker Desktop → Settings → Resources → Disk image size
```

**Why they differ**:

```
Mac:       df -h → shows your Mac's actual SSD (e.g., 500GB Apple SSD)
Container: df -h → shows the Docker VM's virtual disk (e.g., 59GB)

The chain:
  Mac SSD (500GB physical) → hosts →
  Docker Desktop VM virtual disk file (e.g., ~/Library/Containers/Docker/Data/vms/0/data/Docker.raw, 59GB cap)
  This virtual disk file contains the Docker VM filesystem
  Inside that VM: overlay filesystems for each container
```

**The diagram**:

```
Mac Physical SSD (500GB NVMe)
         ↓  (file on Mac SSD)
Docker VM Virtual Disk ~/...Docker.raw (59GB image)
         ↓  (virtual block device inside VM)
/dev/vda (59GB virtual disk in the VM)
         ↓  (partition)
/dev/vda1 (59GB partition)
         ↓  (ext4 filesystem on partition)
Docker overlay root (container reads this)
         ↓  (union of layers)
Container /  (what you see in df -h)
```

**Portworx connection**: This layering is why Portworx asks for "raw unformatted block devices" — it wants `/dev/sdb` directly, NOT a path inside a filesystem. Portworx manages its own pool on top of raw blocks, bypassing all existing filesystem layers.

---

### Solution Q4 — What Happens When You Write Inside the Container?

**Commands**:

```bash
# Create a test file
dd if=/dev/urandom of=/root/test-ephemeral.bin bs=1M count=100 status=progress

# Verify
ls -lh /root/test-ephemeral.bin

# Find the overlay upperdir (where this file ACTUALLY lives)
mount | grep "on / type overlay"
# Find the upperdir= value

# Example: if upperdir=/var/lib/docker/overlay2/abc123/diff
# Your file lives at: /var/lib/docker/overlay2/abc123/diff/root/test-ephemeral.bin
# (on the Docker VM, which is inside the Docker.raw image on your Mac)

# Inside the container, you can verify:
stat /root/test-ephemeral.bin
# Device: the device number matches the overlay filesystem
```

**The Linux concept**: **Ephemeral storage in overlay (Copy-on-Write upper layer)**

```
When container writes /root/test-ephemeral.bin:
  1. Overlay checks: does this path exist in lowerdir (image layers)?
     NO → write directly to upperdir
  2. File created at: upperdir/root/test-ephemeral.bin

When container is deleted:
  docker rm px-storage-lab
  → Docker deletes the upperdir directory tree
  → All files that only existed in upperdir are gone
  → lowerdir (image) is untouched (used by other containers)
```

**Portworx connection**: The motivation for PVCs is precisely this. A Portworx PVC is NOT in the overlay `upperdir`. It is a completely separate block device mounted INTO the container at a specific path. Data in the PVC mount path survives container restarts because it's not in the write layer at all.

---

### Solution Q5 — Read-Only vs Read-Write Storage Layers

**Commands**:

```bash
# Create a file in /etc
echo "test layer file" > /etc/test-layer.txt
ls -la /etc/test-layer.txt
cat /etc/test-layer.txt

# Find the overlay mount
mount | grep "on / type overlay"
# Note the upperdir= value

# The overlay mount output looks like:
# overlay on / type overlay (rw,relatime,
#   lowerdir=/var/lib/docker/overlay2/l/ABC:l/DEF:l/GHI,
#   upperdir=/var/lib/docker/overlay2/XYZ123/diff,
#   workdir=/var/lib/docker/overlay2/XYZ123/work)

# Check if your file is in upperdir
UPPERDIR=$(mount | grep "on / type overlay" | grep -oP "upperdir=\K[^,)]+")
echo "upperdir: $UPPERDIR"
ls $UPPERDIR/etc/test-layer.txt 2>/dev/null && echo "File found in upperdir!" || echo "Not found"
```

**Understanding overlay layers**:

```
lowerdir (read-only, left to right = bottom to top of stack):
  Layer 1: ubuntu:22.04 base OS (FROM ubuntu:22.04 in Dockerfile)
  Layer 2: app layer (RUN apt-get install nginx)
  Layer 3: config layer (COPY nginx.conf /etc/nginx/nginx.conf)
  ...these are SHARED between all containers using this image

upperdir (read-write, unique to THIS container):
  Everything you write goes here
  Including: /etc/test-layer.txt

merged (what you see at /):
  Union of all lowerdir layers + upperdir
  If same path exists in upperdir AND lowerdir → upperdir wins
```

**Copy-on-Write explained**:

```
You run: vim /etc/nginx/nginx.conf (file exists in lowerdir — image layer)
overlay:
  1. File is read-only in lowerdir (in the image)
  2. Overlay COPIES the file to upperdir/etc/nginx/nginx.conf
  3. Your edit goes to the upperdir copy
  4. Original in lowerdir is untouched
  5. Other containers using the same image see the original

This is Copy-on-Write: only copy on first write, then write in place
```

**Why this matters**: Writing large files with CoW means the first write copies the entire original file. For a 1GB database file, first modification = 1GB copy. This is why databases must NOT live in the overlay filesystem.

---

## Section B — Measure

---

### Solution Q6 — Sequential Write and Read Speed

**Commands**:

```bash
# Sequential WRITE test (dd writes zeros from /dev/zero to a file)
dd if=/dev/zero of=/tmp/seq-write-test.bin bs=1M count=500 conv=fdatasync status=progress
# conv=fdatasync forces actual disk flush (not just OS buffer cache)

# Run 3 times:
for i in 1 2 3; do
  echo "Run $i:"
  dd if=/dev/zero of=/tmp/seq-write-test.bin bs=1M count=500 conv=fdatasync status=progress 2>&1 | tail -1
  sleep 2
done

# Sequential READ test (reading back from file)
# First: drop page cache so we read from disk, not RAM
echo 3 > /proc/sys/vm/drop_caches 2>/dev/null || echo "(cache drop skipped — run as test)"

dd if=/tmp/seq-write-test.bin of=/dev/null bs=1M count=500 status=progress
```

**Example output**:

```
500+0 records in
500+0 records out
524288000 bytes (524 MB, 500 MiB) copied, 1.23456 s, 425 MB/s
```

**Typical results in Docker on Mac**:

| Run | Write MB/s | Read MB/s |
|-----|-----------|----------|
| 1 | ~300–500 | ~800–1500 |
| 2 | ~300–500 | ~800–1500 |
| 3 | ~300–500 | ~800–1500 |

**Why write is slower than read**:
- Write must flush data to disk (with `fdatasync`)
- Read can use page cache (already in RAM from previous write)
- Mac NVMe → Docker VM → overlay → container: each layer adds overhead

**Answer**: 10 video files × 100MB/s = 1,000 MB/s needed. This storage provides ~300–500MB/s. It would NOT handle 10 simultaneous 100MB/s streams. This is a real production consideration: Docker container storage is not suitable for high-throughput video ingestion.

**Why `conv=fdatasync` matters**:

```
Without fdatasync:
  dd writes to OS page cache (RAM) → VERY fast (~3–5 GB/s)
  Data not on disk yet
  Speed reported is RAM speed, not disk speed → misleading!

With fdatasync:
  dd forces each block to disk before reporting done
  Speed is actual disk speed → honest measurement

Always use fdatasync when benchmarking storage for databases.
```

---

### Solution Q7 — Random 4KB IOPS

**Commands**:

```bash
# Create a test file first (fio will use it)
mkdir -p /tmp/px-lab/fio-test

# Random WRITE IOPS (4KB blocks, simulates database writes)
fio --name=rand-write-iops \
    --rw=randwrite \
    --bs=4k \
    --size=500M \
    --numjobs=4 \
    --iodepth=32 \
    --runtime=30 \
    --time_based \
    --filename=/tmp/px-lab/fio-test/randwrite.bin \
    --output-format=normal

# Random READ IOPS (4KB blocks)
fio --name=rand-read-iops \
    --rw=randread \
    --bs=4k \
    --size=500M \
    --numjobs=4 \
    --iodepth=32 \
    --runtime=30 \
    --time_based \
    --filename=/tmp/px-lab/fio-test/randwrite.bin \
    --output-format=normal
```

**Reading fio output**:

```
rand-write-iops: (groupid=0, jobs=4): err= 0:
  write: IOPS=8234, BW=32.2MiB/s (33.8MB/s)
    clat (usec): min=512, max=89432, avg=3890.12, stdev=5123.45
    lat (usec): min=512, max=89432, avg=3901.23, stdev=5125.67
    clat percentiles (usec):
      | 1.00th=[ 1237], 5.00th=[ 1729], 10.00th=[ 2040],
      | 50.00th=[ 3097], 90.00th=[ 7177], 95.00th=[10552],
      | 99.00th=[25297], 99.50th=[33817], 99.90th=[65536]
```

**How to read this**:

```
IOPS=8234     → 8,234 random 4KB writes per second
BW=32.2MiB/s  → throughput = IOPS × blocksize = 8234 × 4KB = 32.9 MB/s ✓

clat avg=3890µs → average write latency = 3.89ms per write
99th pct=25297µs → 1% of writes take > 25ms → important for tail latency
```

**Answer**: 

In Docker on Mac, you'll typically see 5,000–15,000 random write IOPS at 4KB. This is because:
- You're going through Mac's hypervisor virtualization
- The "disk" is a virtual disk inside a VM image file on Mac NVMe

For production PostgreSQL needing 5,000 IOPS: borderline — would work for development but not production load. A real NVMe SSD does 500,000–1,000,000 random IOPS.

**Why 4KB specifically?**

```
4KB = typical database page size for PostgreSQL, MySQL (InnoDB page = 16KB), etcd
When the database reads or writes one "record", it reads/writes one page
Database IOPS = database page operations per second
This is the most important benchmark for database storage
```

---

### Solution Q8 — Write Latency Measurement

**Commands**:

```bash
# Synchronous write latency test (simulates fsync behavior)
fio --name=sync-write-latency \
    --rw=randwrite \
    --bs=4k \
    --size=100M \
    --numjobs=1 \
    --iodepth=1 \
    --sync=1 \
    --runtime=30 \
    --time_based \
    --filename=/tmp/px-lab/fio-test/syncwrite.bin \
    --output-format=normal

# Key fio flags for latency measurement:
# --numjobs=1   → single job (serial, not parallel)
# --iodepth=1   → no queue depth (one I/O at a time)
# --sync=1      → force O_SYNC (each write is fsynced)
```

**Reading the latency output**:

```
  write: IOPS=312, BW=1249KiB/s
    clat (usec): min=1234, max=23456, avg=3205.45, stdev=1234.56
    lat (usec): min=1234, max=23456, avg=3212.34, stdev=1235.67
    clat percentiles (usec):
      |  1.00th=[ 1369],  5.00th=[ 1614], 10.00th=[ 1909],
      | 50.00th=[ 2900], 90.00th=[ 4883], 95.00th=[ 6259],
      | 99.00th=[12124], 99.50th=[14877], 99.90th=[20579]
```

**Classification table** (your numbers will vary):

| Use Case | Required | Typical Result in Docker | Suitable? |
|----------|---------|--------------------------|-----------|
| etcd | < 10ms | avg ~3ms, 99th ~12ms | Marginal (99th >10ms) |
| PostgreSQL primary | < 5ms | avg ~3ms | OK for dev, not prod |
| Log archive | < 500ms | avg ~3ms | Yes, very suitable |

**Key insight — why `iodepth=1` matters for latency**:

```
iodepth=32 (high queue depth):
  OS queues 32 I/Os → disk reorders for efficiency → high throughput
  But each individual I/O waits in queue → higher latency
  Good for: batch operations, backups

iodepth=1 (no queue):
  Each I/O completes before next starts → latency = actual disk response time
  Lower throughput but shows true latency
  Good for: simulating database fsync behavior
```

**Portworx connection**: etcd is Kubernetes' control plane database. If etcd write latency exceeds 10ms, Kubernetes starts failing health checks, leader elections happen, and eventually the cluster destabilizes. This is why Portworx recommends NVMe specifically for the KVDB (embedded etcd) disk.

---

### Solution Q9 — Block Size vs IOPS vs Throughput

**Commands**:

```bash
# Helper function to run fio with different block sizes
run_blocksize_test() {
  local bs=$1
  echo "=== Block Size: $bs ==="
  fio --name=bs-test \
      --rw=randwrite \
      --bs=$bs \
      --size=500M \
      --numjobs=1 \
      --iodepth=32 \
      --runtime=15 \
      --time_based \
      --filename=/tmp/px-lab/fio-test/bs-test.bin \
      --output-format=terse | cut -d';' -f8,49,50
  # Fields: IOPS, read-bw, write-bw
  echo ""
}

run_blocksize_test 4k
run_blocksize_test 64k
run_blocksize_test 1M
```

**Understanding the math** (example numbers):

| Block Size | IOPS | Throughput MB/s | Calculation |
|-----------|------|----------------|-------------|
| 4KB | 8,000 | ~32 MB/s | 8000 × 4096 = 32.7 MB/s |
| 64KB | 1,000 | ~64 MB/s | 1000 × 65536 = 65.5 MB/s |
| 1MB | 100 | ~100 MB/s | 100 × 1048576 = 100 MB/s |

**The formula**:

```
Throughput = IOPS × Block Size

Rearranged:
IOPS = Throughput / Block Size
Block Size = Throughput / IOPS
```

**What stays constant**: **Throughput** roughly plateaus at the disk's sequential bandwidth ceiling. As block size increases, IOPS drops but throughput stays about the same (up to the disk's max sequential speed).

**What changes**: IOPS drops as block size increases. Each I/O does more work, so fewer are needed per second to hit the same throughput.

**The key insight**:

```
A disk has TWO limits:
1. IOPS limit      → maximum operations per second (constrained by seek time, latency)
2. Bandwidth limit → maximum bytes per second (constrained by interface speed)

At small block sizes: IOPS limit is hit first
At large block sizes: Bandwidth limit is hit first

Example:
  Disk: 10,000 IOPS max, 200 MB/s bandwidth max
  At 4KB:  10,000 × 4KB = 40 MB/s → IOPS-limited (can't do more ops)
  At 64KB: 10,000 × 64KB = 640 MB/s → bandwidth-limited (capped at 200 MB/s)
  Actual at 64KB: 200 MB/s ÷ 64KB = 3,125 IOPS (bandwidth limit kicks in)
```

**Portworx connection**: Portworx uses 4KB blocks internally for its replication engine. This is why the journal disk's random 4KB IOPS directly determines Portworx write performance — every write through Portworx is a 4KB block write to the journal first.

---

### Solution Q10 — Real-Time I/O Observation with iostat

**Commands**:

```bash
# Terminal 1: Watch I/O continuously
iostat -x 1

# Terminal 2: Generate I/O
fio --name=generate-io \
    --rw=randwrite \
    --bs=4k \
    --size=500M \
    --numjobs=4 \
    --iodepth=32 \
    --runtime=60 \
    --time_based \
    --filename=/tmp/px-lab/fio-test/iostat-test.bin \
    --output=/dev/null &
```

**Reading iostat output**:

```
Device     r/s    w/s  rkB/s  wkB/s  await  %util
sda       0.00  0.00   0.00   0.00   0.00   0.00   ← before fio
sda       0.00 8234.00  0.00 32936.00  3.89  98.45  ← while fio runs
sda       0.00  0.00   0.00   0.00   0.00   0.00   ← after fio
```

**Column explanations**:

| Column | Meaning | Alert Threshold |
|--------|---------|----------------|
| `r/s` | Read operations per second (IOPS) | Depends on workload |
| `w/s` | Write operations per second (IOPS) | Depends on workload |
| `rkB/s` | Read throughput in KB/s | Depends on workload |
| `wkB/s` | Write throughput in KB/s | Depends on workload |
| `await` | Average time (ms) for I/O to complete | > 10ms for SSD = investigate |
| `%util` | Percentage of time disk was busy | > 90% = saturated |
| `r_await` | Average read latency (ms) | > 5ms for SSD = investigate |
| `w_await` | Average write latency (ms) | > 10ms for SSD = investigate |

**Which column says "disk is saturated"**: `%util` at or near 100%. When `%util = 100%`, the disk has no idle time — every request waits. Combined with high `await`, this confirms saturation.

**Important nuance**:

```
%util can hit 100% even at LOW IOPS if requests are slow (HDD)
%util = 100% at HIGH IOPS on SSD means true saturation

The real saturation signal:
  %util = 100% AND await is higher than usual for this disk type
  = disk cannot keep up with I/O demand
```

**Portworx connection**: `pxctl service drive iostat` shows the same metrics for Portworx storage pool disks. During a "Portworx volume is slow" incident, checking iostat on the underlying disk is step one of the investigation.

---

## Section C — Build

> All commands in this section run inside your Ubuntu container. The container runs inside the Docker/Podman VM on your Mac. Loop devices and filesystems are created in the VM kernel — your macOS is never touched.

---

### Solution Q11 — Create a Virtual Block Device

**Commands** (run inside container):

```bash
# Step 1: Create the disk image file (500MB of zeros)
dd if=/dev/zero of=/tmp/px-lab/disks/disk01.img bs=1M count=500 status=progress

# Step 2: Attach to the next available loop device, capture device path
DISK01=$(losetup -f --show /tmp/px-lab/disks/disk01.img)
echo "Disk attached as: $DISK01"
# Typical output: /dev/loop0

# Step 3: Verify it appears as a block device
lsblk $DISK01
# NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
# loop0    7:0    0  500M  0 loop

# Step 4: Check the device type
ls -la $DISK01
# brw-rw---- 1 root disk 7, 0 ... /dev/loop0
# ↑ 'b' = block device (c = character, d = directory, - = regular file)
```

**Where things actually live in your setup**:

```
MacBook (macOS / Darwin kernel)
└── ~/px-lab/disks/disk01.img   ← the image file on your Mac SSD
    ↕ (shared via Apple Hypervisor volume mount)
Docker/Podman VM (Linux kernel)
└── /tmp/px-lab/disks/disk01.img  ← same file, VM's view of it
    ↓ (losetup binds it)
    /dev/loop0                    ← virtual block device (lives in VM kernel)
Ubuntu container
└── sees /dev/loop0 via --privileged + shared VM kernel
    sees /tmp/px-lab/disks/ via -v volume mount
```

**How loop devices work**:

```
Regular file on disk:
  /tmp/px-lab/disks/disk01.img — 500MB of raw bytes, no structure

After losetup:
  /dev/loop0 is a kernel device node
  The loop driver intercepts ALL I/O addressed to /dev/loop0
  Translates: read/write at byte offset N → read/write at offset N in disk01.img
  Result: rest of the kernel (parted, mkfs, mount) sees a real block device

This is byte-for-byte identical to talking to a physical SSD.
The loop driver adds < 1μs overhead — negligible for learning.
```

**Why `lsmod | grep loop` returns nothing** (even though loop devices work):

```
Kernel features can exist in two forms:

  Built-in (CONFIG_BLK_DEV_LOOP=y):
    Compiled directly into the kernel binary
    Always available — no loading required
    NEVER appears in lsmod (lsmod only lists runtime-loadable modules)
    This is what Docker/Podman VM kernels use

  Loadable module (CONFIG_BLK_DEV_LOOP=m):
    Shipped as loop.ko file
    Loaded on first use via modprobe
    Shows up in lsmod after loading

The Docker Desktop and Podman Machine VMs compile loop as built-in.
lsmod returning nothing does NOT mean loop is absent.
```

**How to verify loop support without lsmod**:

```bash
# Fastest: ask for next free device — if it answers, loop works
losetup -f
# /dev/loop0

# Check kernel config (if available)
zcat /proc/config.gz 2>/dev/null | grep CONFIG_BLK_DEV_LOOP
# CONFIG_BLK_DEV_LOOP=y  ← 'y' = built-in, not a module

# Also works: list active loop attachments
losetup -l
```

**Real-world connection**:

```
AWS EBS volumes are virtual block devices — the same concept.
When an EC2 instance gets a 500GB EBS volume attached, it appears as /dev/xvdf.
Behind the scenes, AWS maps that device to a file or volume on their distributed storage.
Your loop device is the same abstraction at a smaller scale.

Portworx dev environments use loop devices because:
  - CI/CD pipelines have no spare disks
  - Loop devices are created/destroyed in seconds
  - Behave identically to real NVMe drives for API and filesystem purposes
  - Portworx's integration tests run entirely on loop devices in containers
```

---

### Solution Q12 — Partition the Virtual Disk

**The Container Problem First**

On a real Linux server, after creating partitions with `parted`, you run `partprobe` or `kpartx` to tell the kernel about new partitions. In a container on Docker/Podman VM, these tools fail silently or require udev support that containers do not have:

```
Real Linux server:
  parted creates partition → partprobe → udev sees event → /dev/sda1 appears
  Works because container has full access to udev daemon

Your setup (container inside VM):
  parted creates partition → partprobe → udev not running in container → nothing happens
  kpartx -a → device mapper may be unavailable → nothing happens
```

**The correct container approach: detach and re-attach with `--partscan`**

```bash
# Step 1: Create GPT partition table
parted $DISK01 mklabel gpt
# Output: Information: You may need to update /etc/fstab.
# ← This is NOT an error. Parted always prints this after partition operations. Ignore it.

# Step 2: Create first partition (50% = 250MB, for ext4)
parted $DISK01 mkpart primary ext4 0% 50%
# Output: Information: You may need to update /etc/fstab.

# Step 3: Create second partition (remaining 50% = 250MB, for XFS)
parted $DISK01 mkpart primary xfs 50% 100%
# Output: Information: You may need to update /etc/fstab.

# Step 4: Verify parted sees both partitions (parted stores this in the image file)
parted $DISK01 print
# Number  Start   End    Size    File system  Name     Flags
#  1      1049kB  263MB  262MB   ext4         primary
#  2      263MB   525MB  262MB   xfs          primary

# Step 5 (CRITICAL — container-specific): Detach and re-attach with --partscan
# This forces the kernel to read the partition table at attach time,
# creating partition device nodes without needing udev/partprobe.
losetup -d $DISK01
DISK01=$(losetup --partscan -f --show /tmp/px-lab/disks/disk01.img)
echo "Re-attached as: $DISK01"
# /dev/loop0  (same device, now with partition scanning enabled)

# Step 6: Verify partition nodes exist
lsblk $DISK01
# NAME       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
# loop0        7:0    0  500M  0 loop
# ├─loop0p1    7:1    0  250M  0 part
# └─loop0p2    7:2    0  250M  0 part
```

**Why `--partscan` works when `partprobe` does not**:

```
losetup --partscan:
  Instructs the loop driver to read the partition table during device attachment
  The loop driver is in the VM kernel, which has full privileges
  Creates /dev/loop0p1, /dev/loop0p2 directly in the kernel
  Does not require udev to be running
  Works correctly inside containers

partprobe:
  Sends BLKRRPART ioctl to the block device
  Requires udev to receive the event and create device nodes
  In a container, udev is not running — device nodes never appear
  Appears to succeed (exit code 0) but nothing happens

kpartx:
  Creates device-mapper nodes (/dev/mapper/loop0p1)
  Requires device mapper kernel module (dm-mod) to be available
  May work in some privileged containers but unreliable in this setup
```

**GPT vs MBR**:

```
MBR (Master Boot Record):
  1983 IBM PC standard
  Stored in first 512 bytes of disk
  Max 4 primary partitions
  Max disk size: 2TB (32-bit LBA addressing × 512-byte sectors)
  Common on older systems

GPT (GUID Partition Table):
  UEFI standard, replaces MBR
  Stores partition table in first 34 sectors AND last 34 sectors (backup copy)
  Max 128 partitions by default
  Max disk size: 9.4 ZB (64-bit LBA addressing)
  Every partition has a UUID (no conflicts across disks)
  All modern storage uses GPT

Portworx context:
  Portworx nodes use enterprise NVMe/SAS disks that are often > 2TB
  Portworx requires GPT on any disk > 2TB
  But Portworx uses the entire raw disk without any partition table at all (useAll: true)
  Why? Portworx manages its own internal layout across the raw device
  Skipping the partition table removes one layer of indirection and eliminates misalignment risk
```

**Real-world connection**:

```
AWS EBS: When you attach a 1TB EBS volume to EC2, you get /dev/xvdf with no partition table.
You run mkfs.ext4 /dev/xvdf or mount it directly.
Most cloud-native applications do the same — no partitions, just a raw device.

Portworx does this at scale:
  StorageCluster with useAll: true scans nodes for unpartitioned, unformatted disks
  Takes ownership of /dev/sdb, /dev/sdc directly
  Writes its own metadata in the first blocks (Portworx journal format)
  No partition table in Portworx-managed disks is intentional, not an oversight.

In this lab you created partitions to learn how they work.
In production Portworx, you deliberately avoid them.
```

---

### Solution Q13 — Format with ext4 and Explore Inodes

> **Prerequisite**: `$DISK01` must be the re-attached loop device from Q12 (the one with `--partscan`). If you opened a new shell, re-run: `DISK01=$(losetup -l | grep disk01.img | awk '{print $1}')` to recover the variable.

**Commands** (run inside container):

```bash
# Step 1: Format the first partition with ext4
mkfs.ext4 ${DISK01}p1
# ${DISK01}p1 = /dev/loop0p1 — exists because of --partscan in Q12
# You will see: mke2fs creating filesystem with 63488 4k blocks and 65536 inodes

# Step 2: Mount it
mkdir -p /tmp/px-lab/mounts/ext4test
mount ${DISK01}p1 /tmp/px-lab/mounts/ext4test

# Step 3: Check inode count
df -i /tmp/px-lab/mounts/ext4test
# Filesystem     Inodes IUsed  IFree IUse% Mounted on
# /dev/loop0p1    65536    11  65525    1% /tmp/px-lab/mounts/ext4test
# ↑ 65,536 inodes for a 250MB filesystem

# Why 65,536?
# ext4 default: 1 inode per 16KB of space
# 250MB = 256,000 KB → 256,000 / 16 = 16,000... but ext4 rounds up to powers of 2
# 65536 = 2^16 (next power of 2 above 16,000)

# Step 4: Get detailed inode info from ext4 superblock
tune2fs -l ${DISK01}p1 | grep -i inode
# Inode count:              65536
# Free inodes:              65525
# Inodes per group:         8192
# Inode size:               256     ← 256 bytes per inode

# Total space consumed by inodes: 65536 × 256 bytes = 16MB
# This space is reserved at format time and cannot be used for file data

# Step 5: Create 100 files, watch inode usage
for i in $(seq 1 100); do
  touch /tmp/px-lab/mounts/ext4test/file_$i
done
df -i /tmp/px-lab/mounts/ext4test
# IUsed: 111  (11 system + 100 your files)

# Step 6: Create a filesystem with artificially low inode count
dd if=/dev/zero of=/tmp/px-lab/disks/smallinodes.img bs=1M count=100 status=none
SMALL=$(losetup -f --show /tmp/px-lab/disks/smallinodes.img)
mkfs.ext4 -N 500 $SMALL
# -N 500 forces exactly 500 inodes regardless of filesystem size
mkdir -p /tmp/px-lab/mounts/inodetest
mount $SMALL /tmp/px-lab/mounts/inodetest

# Exhaust the 500 inodes
for i in $(seq 1 600); do
  result=$(touch /tmp/px-lab/mounts/inodetest/file_$i 2>&1)
  if [ -n "$result" ]; then
    echo "FAILED at file $i: $result"
    break
  fi
done
# Expected: FAILED at file ~491: touch: cannot touch '...': No space left on device
# (some inodes used by system — actual failure is around 491, not 500)

# The deceptive output:
df -h /tmp/px-lab/mounts/inodetest
# /dev/loop5   95M   3.0M   92M   4%  /tmp/px-lab/mounts/inodetest
# ← MISLEADING: 92MB free, looks healthy

df -i /tmp/px-lab/mounts/inodetest
# /dev/loop5   500   500   0  100%  /tmp/px-lab/mounts/inodetest
# ← TRUTH: 0 inodes free, exhausted
```

**What is an inode** (the kernel data structure):

```
Inode = Index Node (one per file/directory on the filesystem)

Stored on disk, in the inode table.
Every file and directory has exactly ONE inode.

What an inode stores:
  File size (in bytes)
  File type (regular, directory, symlink, block device...)
  Permissions (mode bits: rwxr-xr-x → 0755)
  Owner UID and group GID
  Timestamps: atime (accessed), mtime (modified), ctime (metadata changed)
  Hard link count
  Pointers to data blocks (where the actual content lives)

What an inode does NOT store:
  The filename — filenames live in directory entries
  The file content — content lives in data blocks

The directory is a file that maps: "disk01.img" → inode 12345
When you run ls -li, the first column is the inode number.

Run now: ls -li /tmp/px-lab/mounts/ext4test/ | head
You'll see a number like 12 13 14 ... in the first column.
```

**3-step diagnostic for "No space left on device"**:

```
Step 1: Is the disk full?
  df -h /affected/path
  → IUse% = 100%: disk full → delete or move large files
  → IUse% < 100%: not a space problem → go to Step 2

Step 2: Are inodes exhausted?
  df -i /affected/path
  → IUse% = 100%: inode exhaustion
  → Find the directory with millions of files:
    find /affected/path -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head
  → Fix: delete the directory of small files, or remount after freeing them
  → Permanent fix: use XFS (dynamic inodes) instead of ext4
  → IUse% < 100%: go to Step 3

Step 3: Are deleted files still held open?
  lsof +L1 | grep /affected/path
  → Shows files that are deleted from the directory but still open by a process
  → The kernel cannot free the disk blocks until the process closes the file handle
  → Fix: restart the process holding the file open
```

**Real-world connection**:

```
This exact scenario — "No space left on device" with df -h showing 10% used —
is one of the most common 3AM Elasticsearch incidents.

Production setup:
  Elasticsearch node → Portworx volume → data node pod
  Elasticsearch writes 1 Lucene segment file per index shard per merge cycle
  A busy cluster with 100 shards can create thousands of segment files per hour

On ext4 (default inode count):
  250GB ext4 volume → ~15 million inodes
  Sounds like a lot, but busy ES with 100 shards × heavy indexing exhausts this in days

On XFS (Portworx default):
  XFS allocates inodes dynamically — no fixed limit
  XFS never generates "No space left on device" due to inodes
  This is why Portworx chose XFS as default for all volumes

SRE playbook when you see this at 3AM:
  1. df -h → not full
  2. df -i → 100% inodes (now you know)
  3. find → Elasticsearch data directory has millions of segment files
  4. Short-term: delete old indices via Elasticsearch API
  5. Long-term: migrate to Portworx XFS volume with StorageClass xfs parameter
```

---

### Solution Q14 — Format with XFS and Compare

**Commands** (run inside container):

```bash
# Step 1: Create a second 500MB disk for XFS
dd if=/dev/zero of=/tmp/px-lab/disks/disk02.img bs=1M count=500 status=none
DISK02=$(losetup -f --show /tmp/px-lab/disks/disk02.img)
echo "XFS disk attached as: $DISK02"
# Note: DISK02 is a whole device (no partitions) — we format it directly
# This mirrors how Portworx uses raw block devices

# Step 2: Format with XFS
mkfs.xfs $DISK02
# You will see: meta-data=/dev/loop1 isize=512 ...

# Step 3: Mount it
mkdir -p /tmp/px-lab/mounts/xfstest
mount $DISK02 /tmp/px-lab/mounts/xfstest

# Step 4: Compare inode allocation between ext4 and XFS
echo "=== ext4 inodes (250MB partition, fixed at format) ==="
df -i /tmp/px-lab/mounts/ext4test
# Filesystem     Inodes  IUsed  IFree IUse% Mounted on
# /dev/loop0p1    65536    111  65425    1% /tmp/px-lab/mounts/ext4test
# ↑ 65,536 inodes — decided at mkfs time, never changes

echo "=== XFS inodes (500MB device, dynamic) ==="
df -i /tmp/px-lab/mounts/xfstest
# Filesystem     Inodes IUsed  IFree IUse% Mounted on
# /dev/loop1          0     3      0    0% /tmp/px-lab/mounts/xfstest
# ↑ Inodes: 0 — XFS doesn't pre-allocate; df -i can't show a fixed count
#   IFree: 0 — not zero because it's full, but because the number is dynamic
#   XFS grows its inode table as files are created; the limit is disk space itself

# Step 5: Get detailed filesystem info
echo "=== ext4 superblock info ==="
tune2fs -l ${DISK01}p1 | grep -iE "inode|block|journal"
# Inode count: 65536
# Block count: 63488
# Journal size: 4096 blocks
# Journal mode: ordered

echo "=== XFS filesystem geometry ==="
xfs_info /tmp/px-lab/mounts/xfstest
# meta-data=/dev/loop1 isize=512 agcount=4, agsize=32000 blks
#          =              sectsz=512  attr=2, projid32bit=1
# data     =              bsize=4096  blocks=128000, imaxpct=25
#          =              sunit=0 blks, swidth=0 blks
# naming   =version 2     bsize=4096  ascii-ci=0, ftype=1
# log      =internal log   bsize=4096  blocks=1368, version=2
# realtime =none           extsz=4096  blocks=0, rtextents=0

# imaxpct=25 means XFS will allocate up to 25% of disk space for inodes
# On a 500MB disk: up to 125MB available for inodes (dynamically allocated)

# Step 6: Attempt to shrink XFS (this will fail)
xfs_growfs -d /tmp/px-lab/mounts/xfstest 2>&1
# data blocks changed from 128000 to 128000
# or: xfs_growfs: XFS_IOC_FSGROWFSDATA xfsctl failed: Invalid argument
# XFS can only GROW, never shrink.
```

**Completed comparison table**:

| Property | ext4 | XFS |
|----------|------|-----|
| Inode allocation | Fixed at format time | Dynamic (grows as needed) |
| Max filesystem size | 1 EB | 8 EB |
| Can shrink online? | No | No |
| Can grow online? | Yes | Yes |
| Performance at large file count | Good | Better (parallel B-tree allocation) |
| Default journal mode | ordered | always-on (write-ahead log) |
| Inode exhaustion risk | Yes (fixed count, can hit 100%) | No (limited only by disk space) |
| Default in Portworx | No | Yes |

**Why XFS wins for Portworx — two specific reasons from what you observed**:

```
Reason 1: No inode exhaustion (you proved this in Q13)
  ext4 pre-allocates inodes at format time — you watched a 100MB filesystem run out
  at 491 files despite 92MB of free space.
  XFS does not pre-allocate — its inode table grows as files are created.
  Portworx volumes serve databases (MySQL, Postgres, Elasticsearch, Cassandra).
  These applications can create thousands of small files per hour.
  XFS eliminates the inode exhaustion failure mode entirely.

Reason 2: Better concurrent write throughput (visible in xfs_info output)
  XFS uses B-tree structures for both inode management and extent allocation.
  agcount=4 in xfs_info means XFS creates 4 allocation groups on a 500MB device.
  Each allocation group has its own lock.
  Multiple concurrent writers can operate on different allocation groups in parallel.
  ext4's single block group lock becomes a bottleneck under concurrent writes.
  Database engines do concurrent random writes (InnoDB, RocksDB, WAL writers).
  XFS handles this better → lower write latency under load → faster Portworx volumes.
```

**Real-world connection**:

```
Portworx StorageClass for a production MySQL cluster:

  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: portworx-mysql
  provisioner: pxd.portworx.com
  parameters:
    repl: "3"
    io_profile: "db"        ← tuned for database I/O patterns
    fs: "xfs"               ← explicitly XFS (default but made explicit here)

When Portworx provisions this volume, it runs mkfs.xfs on the raw Portworx device.
The MySQL pod's /var/lib/mysql is an XFS volume.
If a developer asks "why XFS?", you now have two data-driven answers from this lab.

Cloud provider comparison:
  AWS EFS (managed NFS): uses ext4 internally — can hit inode limits
  AWS EBS (block storage): you choose the filesystem when mounting
  Google Persistent Disk: XFS or ext4 depending on your mkfs call
  Portworx: always XFS by default — consistent behavior regardless of cloud
```

---

### Solution Q15 — Simulate Inode Exhaustion (Production Incident)

**Commands** (run inside container):

```bash
# Step 1: Create a limited-inode filesystem (simulates a poorly-configured production volume)
dd if=/dev/zero of=/tmp/px-lab/disks/inodecrash.img bs=1M count=200 status=none
INODEDISK=$(losetup -f --show /tmp/px-lab/disks/inodecrash.img)

mkfs.ext4 -N 1000 $INODEDISK -q
# -N 1000: create filesystem with only 1000 inodes
# -q: quiet mode
# The filesystem has ~196MB of data space but only 1000 slots for files

mkdir -p /tmp/px-lab/mounts/inodecrash
mount $INODEDISK /tmp/px-lab/mounts/inodecrash

# Verify initial state: space looks fine, inodes limited
df -h /tmp/px-lab/mounts/inodecrash
# Filesystem      Size  Used Avail Use%
# /dev/loopX      191M  1.6M  176M   1%   ← looks healthy

df -i /tmp/px-lab/mounts/inodecrash
# Filesystem     Inodes IUsed IFree IUse%
# /dev/loopX       1000    11   989    1%  ← 1000 total, 11 used by system

# Step 2: Run the file-creation loop (simulates Elasticsearch writing segment files)
for i in $(seq 1 1100); do
  result=$(touch /tmp/px-lab/mounts/inodecrash/file_$i 2>&1)
  if [ -n "$result" ]; then
    echo "FAILED at file $i: $result"
    break
  fi
done
# Expected output:
# FAILED at file ~990: touch: cannot touch '.../file_990': No space left on device
# (fails before 1000 because the filesystem itself uses ~11 inodes for metadata)

# Step 3: Run the deceptive diagnostics a confused SRE would run first
echo "=== What df -h shows (misleading) ==="
df -h /tmp/px-lab/mounts/inodecrash
# Filesystem      Size  Used Avail Use%
# /dev/loopX      191M  9.8M  168M   6%   ← 6% used, 168MB free — NOT the problem

echo "=== What df -i shows (reveals the truth) ==="
df -i /tmp/px-lab/mounts/inodecrash
# Filesystem     Inodes IUsed IFree IUse%
# /dev/loopX       1000  1000     0  100%  ← ZERO inodes left

# Step 4: Find which directory has the most files
find /tmp/px-lab/mounts/inodecrash -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head
# 989 /tmp/px-lab/mounts/inodecrash
# ← the root of the mount has 989 files (the ones you created)
```

**The 3-step production diagnostic procedure**:

```bash
# ─────────────────────────────────────────────────────
# 3AM RUNBOOK: "No space left on device" on a Portworx volume
# ─────────────────────────────────────────────────────

# STEP 1: Is the disk full? (rule out the obvious)
df -h /var/lib/elasticsearch    # or whatever the mount path is
# → If Use% is 100%: disk full
#   Short-term: delete old indices or log files
#   Long-term: expand the PVC via kubectl patch / Portworx Autopilot
#
# → If Use% < 100% (e.g., 6%): disk is NOT full → go to Step 2

# STEP 2: Are inodes exhausted? (the hidden culprit)
df -i /var/lib/elasticsearch
# → If IUse% is 100%: INODE EXHAUSTION confirmed
#   Find the offending directory:
find /var/lib/elasticsearch -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head
#   Short-term: delete old data (old ES indices, old log files, old segment files)
#   Long-term: migrate volume to XFS — XFS never exhausts inodes
#
# → If IUse% < 100%: go to Step 3

# STEP 3: Are deleted files still held open? (the ghost files)
lsof +L1 | grep /var/lib/elasticsearch
# → Shows files deleted from the directory but still open by Elasticsearch process
#   The kernel cannot reclaim those blocks until the process closes the file handle
#   Fix: restart Elasticsearch — it will close all file handles
#   The disk space is reclaimed immediately after process restart
```

**What the error looks like from a Kubernetes pod**:

```
$ kubectl logs elasticsearch-data-0 -n monitoring | tail -20
...
[2026-07-02T03:14:22,431][ERROR][o.e.i.e.Engine] [node-1] [logs-2026.07][0]
  failed engine [index]
org.apache.lucene.store.LockObtainFailedException: Failed to obtain lock on ...
Caused by: java.io.IOException: No space left on device
    at java.io.UnixFileSystem.createFileExclusively(Native Method)

$ kubectl exec -it elasticsearch-data-0 -- df -h /usr/share/elasticsearch/data
Filesystem      Size  Used Avail Use%
/dev/pxd/pxd123  200G   18G  182G  10%    ← 10% used, looks fine

$ kubectl exec -it elasticsearch-data-0 -- df -i /usr/share/elasticsearch/data
Filesystem       Inodes   IUsed   IFree IUse% Mounted on
/dev/pxd/pxd123 13107200 13107200       0  100%    ← 100% inodes
```

**Why this happens on ext4 Portworx volumes but not XFS**:

```
A Portworx PVC backed by ext4:
  200GB ext4 volume → ~13 million inodes (1 per 16KB)
  Elasticsearch with 50 shards × heavy indexing:
    → Lucene creates 5-10 segment files per shard per merge cycle
    → 50 shards × 8 files × 100 merges/day = 40,000 files/day
    → 13,000,000 ÷ 40,000 = ~325 days until inode exhaustion
    → Production incident in year 2 — looks random, hard to correlate

A Portworx PVC backed by XFS (the default):
  XFS allocates inodes dynamically — they grow as needed
  imaxpct=25 means up to 25% of disk space can be used for inodes
  200GB × 25% = 50GB available for inode table
  50GB / 512 bytes per inode = ~100 million inodes possible
  Elasticsearch would run out of disk space long before XFS runs out of inodes

This is why Portworx defaults to XFS. Not marketing — engineering.
```

**Real-world remediation**:

```
Emergency: pod is stuck, writes failing
  1. kubectl exec into pod
  2. df -h and df -i to confirm inode exhaustion
  3. Delete old data inside the pod to free inodes:
     curl -X DELETE http://localhost:9200/logs-2026.06.*  (Elasticsearch)
  4. Pod resumes writing immediately

Permanent fix option A: resize volume to ext4 with more inodes
  - Not possible — you cannot change inode count after format
  - You would need to create new volume, migrate data, swap PVC

Permanent fix option B: migrate to XFS (the correct fix)
  - Create new StorageClass with fs=xfs parameter
  - Create new PVC, snapshot-restore data, update pod spec
  - Portworx's volume clone/migration handles this without downtime

Prevention: Portworx Autopilot + XFS default
  - All new volumes created with Portworx's default StorageClass use XFS
  - Autopilot expands volumes before they fill up
  - Inode exhaustion on Portworx volumes is essentially eliminated
```

---

## Section D — Break and Fix

---

### Solution Q16 — RAID 0 Striping

**Commands**:

```bash
# Create two disk images
dd if=/dev/zero of=/tmp/px-lab/disks/raid0-disk1.img bs=1M count=300 status=none
dd if=/dev/zero of=/tmp/px-lab/disks/raid0-disk2.img bs=1M count=300 status=none
R0D1=$(losetup -f --show /tmp/px-lab/disks/raid0-disk1.img)
R0D2=$(losetup -f --show /tmp/px-lab/disks/raid0-disk2.img)

# Create RAID 0 array
mdadm --create /dev/md0 \
      --level=0 \
      --raid-devices=2 \
      $R0D1 $R0D2 \
      --force

# Verify RAID is online
cat /proc/mdstat
# md0 : active raid0 loop5[1] loop4[0]
#       614400 blocks super 1.2 512k chunks

# Check RAID details
mdadm --detail /dev/md0

# Format and mount
mkfs.xfs /dev/md0
mkdir -p /tmp/px-lab/mounts/raid0
mount /dev/md0 /tmp/px-lab/mounts/raid0

# Benchmark RAID 0 sequential write
dd if=/dev/zero of=/tmp/px-lab/mounts/raid0/bench.bin bs=1M count=200 conv=fdatasync status=progress
```

**Typical results**:

```
Single disk:  ~200–400 MB/s sequential write
RAID 0:       ~200–500 MB/s (limited gain due to Docker VM overhead)
```

**Why RAID 0 may not double speed in this environment**:

```
Theory: 2 disks striped → 2× throughput
Reality in Docker:
  Both "disks" are files on the same underlying physical Mac SSD
  Throughput is limited by: Mac NVMe speed → Docker VM I/O → overlay → container
  The bottleneck is the Mac NVMe, which both "disks" share
  Adding more "disks" doesn't help when they share the same physical path

In production with REAL hardware:
  Two actual NVMe SSDs → each has its own I/O path
  RAID 0 across them → actual ~2× throughput (approaching physical limit)
```

**Capacity math**:

```
RAID 0:
  2 × 300MB = 600MB total capacity (all space usable)
  Usable = N × disk_size (N = number of disks)

One disk: 300MB usable
RAID 0: 600MB usable (but zero fault tolerance)

"RAID 0 means 0 redundancy, 0 fault tolerance"
One disk fails → all data gone immediately (no partial data recovery)
```

**Portworx parallel**: Portworx pools multiple disks per node and presents them as a single storage pool. This is similar in concept to RAID 0 (combining capacity), but Portworx adds replication ACROSS nodes for fault tolerance — addressing RAID 0's fatal weakness.

---

### Solution Q17 — RAID 1 Disk Failure Survival

**Commands**:

```bash
# Create two disk images
dd if=/dev/zero of=/tmp/px-lab/disks/raid1-disk1.img bs=1M count=300 status=none
dd if=/dev/zero of=/tmp/px-lab/disks/raid1-disk2.img bs=1M count=300 status=none
R1D1=$(losetup -f --show /tmp/px-lab/disks/raid1-disk1.img)
R1D2=$(losetup -f --show /tmp/px-lab/disks/raid1-disk2.img)

# Create RAID 1
mdadm --create /dev/md1 \
      --level=1 \
      --raid-devices=2 \
      $R1D1 $R1D2 \
      --force

# Wait for initial sync
watch cat /proc/mdstat
# md1 : active raid1 loop7[1] loop6[0]
#       299008 blocks [2/2] [UU]    ← [UU] = both Up
# resync = INPROGRESS: 100% ... done

# Format and mount
mkfs.ext4 /dev/md1 -q
mkdir -p /tmp/px-lab/mounts/raid1
mount /dev/md1 /tmp/px-lab/mounts/raid1

# Write critical data
echo "critical production data - order #9876" > /tmp/px-lab/mounts/raid1/important.txt
cat /tmp/px-lab/mounts/raid1/important.txt

# Simulate disk failure
mdadm --fail /dev/md1 $R1D1
echo "=== Disk failed. Checking RAID status ==="
cat /proc/mdstat
# md1 : active raid1 loop7[1] loop6[0](F)    ← (F) = Failed
#       299008 blocks [2/1] [_U]              ← [_U] = 1 down, 1 up

mdadm --detail /dev/md1 | grep State
# State : clean, degraded

# DATA STILL ACCESSIBLE!
cat /tmp/px-lab/mounts/raid1/important.txt
# critical production data - order #9876 ← STILL THERE

# Write more data after failure
echo "data written after failure" >> /tmp/px-lab/mounts/raid1/important.txt
cat /tmp/px-lab/mounts/raid1/important.txt
# Both lines are visible!

# Simulate disk replacement
mdadm --remove /dev/md1 $R1D1     # remove failed disk
dd if=/dev/zero of=/tmp/px-lab/disks/raid1-replacement.img bs=1M count=300 status=none
NEWDISK=$(losetup -f --show /tmp/px-lab/disks/raid1-replacement.img)
mdadm --add /dev/md1 $NEWDISK     # add replacement

# Watch rebuild
watch cat /proc/mdstat
# md1 : active raid1 loop8[2] loop7[1]
#       [=>...................]  recovery = 8.5% (25600/299008) finish=0.1min speed=25600K/sec
```

**Why RAID 1 degraded is dangerous**:

```
Normal RAID 1 (2 disks): survives 1 disk failure
Degraded RAID 1 (1 disk): now at RAID 0 risk level
  → If remaining disk fails during rebuild: TOTAL data loss
  → Rebuild duration: hours to days depending on disk size
  → During rebuild: disk works harder (higher failure risk!)

This is called "the RAID rebuild problem":
  1 disk fails → RAID degraded → rebuild starts
  During rebuild: surviving disks run at 100% I/O load
  Higher I/O load → higher heat → higher failure probability
  If second disk fails during rebuild → all data gone

Production rule: never ignore a degraded RAID. Replace immediately.
```

**Portworx parallel**: Portworx replication = distributed RAID 1 across nodes. When a Portworx node fails, `pxctl alerts show` shows "Volume degraded." This is identical to RAID 1 degraded — you have N-1 replicas and must restore to full replication as soon as possible.

---

### Solution Q18 — Storage Capacity Math

**Completed table** (6 × 10TB = 60TB raw):

| Strategy | Total Raw | Usable Storage | Fault Tolerance | Formula |
|----------|-----------|---------------|-----------------|---------|
| No RAID (JBOD) | 60TB | 60TB | None | raw total |
| RAID 0 (all 6) | 60TB | 60TB | None (any fail = loss) | N × disk |
| RAID 1 (3 pairs) | 60TB | 30TB | 1 per pair | total / 2 |
| RAID 5 (6 disks) | 60TB | 50TB | 1 disk | (N-1)/N × total = 5/6 × 60 |
| RAID 6 (6 disks) | 60TB | 40TB | 2 disks | (N-2)/N × total = 4/6 × 60 |
| RAID 10 (6 disks) | 60TB | 30TB | 1 per mirror pair | total / 2 |
| Portworx repl=3 | 60TB | 20TB | 2 nodes | total / repl = 60/3 |

**Answers**:

```
1. Most usable space: RAID 0 or JBOD (60TB) — but zero fault tolerance
   Among fault-tolerant options: RAID 5 (50TB usable)

2. Best fault tolerance: RAID 6 (survives any 2 disk failures)
   Or RAID 10 if you care about rebuild performance too

3. Portworx repl=3 is most like: RAID 1 (pure mirroring)
   Both keep N copies of every byte
   Both survive N-1 failures (repl=3 → survive 2 node failures for reads)
   Key difference: Portworx replicates across NODES (failure domain), not disks on same node

4. Portworx repl=3 with 60TB raw = 20TB usable
   Because every byte is stored 3 times
   60TB / 3 = 20TB usable
```

**Why Portworx replication is better than disk-level RAID**:

```
Disk RAID (e.g., RAID 1 on node1):
  node1: disk1 ←→ disk2 (mirror)
  node1 host fails → both disks fail simultaneously (same power, same chassis)
  Data is lost even with RAID 1

Portworx replication:
  node1: replica1
  node2: replica2
  node3: replica3
  node1 fails → replicas on node2 and node3 survive
  Different hosts = different power supplies, different chassis, different rack
  Higher failure domain isolation = better actual fault tolerance
```

---

### Solution Q19 — Disk Full Emergency

**Commands**:

```bash
# Create 100MB filesystem
dd if=/dev/zero of=/tmp/px-lab/disks/fulldisk.img bs=1M count=100 status=none
FULLDISK=$(losetup -f --show /tmp/px-lab/disks/fulldisk.img)
mkfs.ext4 $FULLDISK -q
mkdir -p /tmp/px-lab/mounts/fulltest
mount $FULLDISK /tmp/px-lab/mounts/fulltest

# Fill the disk
while true; do
  dd if=/dev/urandom of=/tmp/px-lab/mounts/fulltest/fill_$(date +%s%N).bin \
     bs=1M count=5 status=none 2>&1
  if [ $? -ne 0 ]; then
    echo "Disk is full!"
    df -h /tmp/px-lab/mounts/fulltest
    break
  fi
done
```

**Expected output**:

```
dd: error writing '/tmp/px-lab/mounts/fulltest/fill_1234567890.bin': No space left on device
Disk is full!
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop9       93M   89M     0 100% /tmp/px-lab/mounts/fulltest
```

**Understanding ext4 reserved blocks**:

```bash
# Check reserved blocks
tune2fs -l $FULLDISK | grep -i reserved
# Reserved block count:  4793
# Reserved blocks uid:   0 (user root)
# Reserved GID:          0 (group root)

# How much is reserved?
# 4793 blocks × 4096 bytes = ~19MB reserved = ~5% of 100MB (default)

# Regular user hits "disk full" when filesystem is 95% full
# Root user can use the remaining 5%

# Why reserved space exists:
# 1. Prevents filesystem fragmentation (filesystem algorithms need free space to work)
# 2. Allows system daemons (running as root) to still write logs when disk is "full"
# 3. Prevents kernel from panicking when / filesystem fills up

# Emergency: reduce reserved space (when you need more space immediately)
tune2fs -m 1 $FULLDISK    # reduce to 1% reserved (from 5%)
# This frees up ~4% of disk immediately
# DO NOT set to 0% — fragmentation will hurt performance

df -h /tmp/px-lab/mounts/fulltest  # should show ~4% more free
```

**Emergency resolution priority order**:

```
1. Find and delete largest files:
   du -sh /path/* | sort -rh | head

2. Find deleted-but-open files (space not yet reclaimed):
   lsof +L1 | grep /path
   → restart the process to reclaim space

3. Reduce ext4 reserved blocks (emergency only):
   tune2fs -m 1 /dev/sdX
   → buys ~4% more space immediately

4. Delete old snapshots (in Portworx context):
   pxctl volume snapshot list
   pxctl volume snapshot delete <old-snap>

5. Add storage (permanent fix):
   Add disk to storage pool
```

---

### Solution Q20 — End-to-End Slow Disk Investigation

**Setup** (already done from question):
```bash
# The background fio job is saturating the disk
# Your job: find it and diagnose it
```

**Investigation procedure**:

```bash
# STEP 1: Identify which device is the problem
iostat -x 1

# Look at the output — find the device with high %util or await:
# NAME    r/s    w/s   rkB/s   wkB/s  await  %util
# loop10  0.0  8234.0   0.0  32936.0   3.89  98.45   ← this one!

# STEP 2: Measure current I/O throughput and utilization
# (already visible in iostat output)
# Record:
# %util: 98.45% → disk is SATURATED
# await: 3.89ms → writes are queuing
# w/s: 8234    → write IOPS
# wkB/s: 32936 → write throughput = ~32 MB/s

# STEP 3: Find which process is causing the I/O
# Option A: iotop (shows per-process I/O)
iotop -o -b -n 3
# Output shows: fio process writing at 32MB/s

# Option B: find processes with open files on the device
mount | grep loop10   # find where it's mounted
lsof /tmp/px-lab/mounts/slowdisk/ | grep -v "^COMMAND"
# Shows: fio  PID  root  ...  REG  ...  /tmp/px-lab/mounts/slowdisk/saturate

# Option C: check /proc for I/O stats per process
cat /proc/$(pgrep fio)/io
# rchar: ...
# wchar: ...
# write_bytes: 123456789   ← bytes written to disk

# STEP 4: Measure the latency
# Already visible in iostat: await = 3.89ms
# For more detail:
fio --name=latency-probe \
    --rw=randwrite \
    --bs=4k \
    --size=10M \
    --numjobs=1 \
    --iodepth=1 \
    --sync=1 \
    --runtime=10 \
    --time_based \
    --filename=/tmp/px-lab/mounts/slowdisk/probe \
    --output-format=normal 2>&1 | grep -E "IOPS|lat"

# STEP 5: Findings
```

**Completed findings template**:

```
Device:          /dev/loop10
%util:           98.45% (SATURATED)
await (ms):      3.89ms (normal for SSD under load, but high for this workload)
Culprit process: fio (PID: <pid>) writing to /tmp/px-lab/mounts/slowdisk/saturate
Write IOPS:      8,234
Throughput:      ~32 MB/s
Root cause:      A fio benchmark process is consuming 100% of disk I/O capacity,
                 leaving no headroom for other processes
Proposed fix:    Kill the fio process (kill $SATPID)
                 OR throttle it with ionice: ionice -c 3 -p <pid> (idle class)
                 OR add more storage disks to increase aggregate IOPS
```

**Stop the simulation**:

```bash
kill $SATPID 2>/dev/null
# Wait 5 seconds
sleep 5
# Verify disk I/O returns to normal
iostat -x 1 3
```

**The 5-step slow disk runbook** (memorize this):

```
1. OBSERVE:   iostat -x 1
              Look for: %util > 80%, await > expected baseline

2. IDENTIFY:  Which device has the problem?
              Map device name to mount point: mount | grep <device>

3. ISOLATE:   Which process is the culprit?
              iotop -o    → per-process I/O
              lsof /mount/point → open files on that filesystem

4. MEASURE:   Run fio latency probe on the affected device
              Establish: current latency vs expected latency for this disk type

5. RESOLVE:   Based on cause:
              a. Runaway process → kill or ionice
              b. Legitimate high load → scale out (add disks/nodes)
              c. Disk failing → check SMART, initiate disk replacement
              d. Wrong workload for disk type → migrate to faster storage
```

---

## Lab Complete

You have now built, measured, broken, and fixed storage concepts that underpin every Portworx operation:

```
Q1–Q5:   Container storage = overlay = ephemeral → WHY PVCs exist
Q6–Q10:  IOPS, throughput, latency are measurable → benchmark before you assume
Q11–Q15: Block devices, partitions, filesystems, inodes → the physical layer beneath Portworx
Q16–Q17: RAID 0 (performance) and RAID 1 (redundancy) → why Portworx replication factor exists
Q18:     Capacity math → how to answer "how much storage do I need?"
Q19–Q20: Disk full and slow disk → the two most common production incidents

Next: Module 02 → Linux Storage (lsblk, LVM, Device Mapper, XFS internals)
```
