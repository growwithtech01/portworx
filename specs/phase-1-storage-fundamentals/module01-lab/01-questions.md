# Module 01 Lab — 20 Use-Case Questions

```
Prereq  : Complete 00-lab-setup.md first
Env     : Ubuntu 22.04 container via Podman or Docker Desktop (privileged)
Progress: Questions 1–5 → Observe | 6–10 → Measure | 11–15 → Build | 16–20 → Break & Fix
```

---

## How to Use This File

- Attempt each question yourself before reading the solution
- Solutions are in [02-solutions.md](./02-solutions.md)
- Each question builds on the previous — do them in order
- All commands run **inside the container**, not on your Mac

---

## Will Any of This Affect My MacBook?

Short answer: **No — except disk space from the volume mount.**

Here is exactly what each section does and does not touch on your Mac:

```
Your MacBook (macOS)
      ↓  only bridge
-v ~/px-lab:/tmp/px-lab   ← shared folder (files here = on your Mac SSD at ~/px-lab/)
      ↓  blast radius stops here
Podman Machine VM (or Docker Desktop VM)
      ↓  everything below is inside the VM
Ubuntu Container
  ├── /dev/loop*        loop devices      → inside VM kernel only
  ├── mdadm RAID        software RAID     → inside VM kernel only
  ├── mkfs / fdisk      filesystem ops    → on loop image files only
  ├── /proc /sys        kernel params     → VM's /proc, not macOS
  └── fio / dd writes   benchmark writes  → to files inside VM
```

### Section-by-Section Mac Impact

| Section | Commands Used | Mac Disk Space | Mac Kernel | macOS Affected? |
|---------|--------------|---------------|-----------|----------------|
| A — Observe (Q1–Q5) | `lsblk`, `df`, `mount`, `dd` | None | No | No |
| B — Measure (Q6–Q10) | `dd`, `fio`, `iostat` | None | No | No |
| C — Build (Q11–Q15) | `losetup`, `mkfs`, `mdadm`, `mount` | ~2–3 GB in `/tmp/px-lab/` | No | No |
| D — Break & Fix (Q16–Q20) | `mdadm --fail`, `fio`, `iostat` | ~2–3 GB in `/tmp/px-lab/` | No | No |

### The Only Thing on Your Mac

The `-v /tmp/px-lab:/tmp/px-lab` flag creates a shared folder between your Mac and the container.

**Before you start any labs** — the folder is empty (or does not exist yet):

```bash
# On your Mac terminal, check right now:
ls /tmp/px-lab/
# Output: No such file or directory
# OR: disks/   mounts/   (both empty)
```

Docker creates `/tmp/px-lab` on your Mac automatically the first time you run the container. Until you run lab questions that create disk images, nothing is inside it.

**After running Q11 onwards** — disk image files appear here:

```
/tmp/px-lab/              ← exists on your Mac (auto-created by Docker)
├── disks/
│   ├── disk01.img        ← appears after Q11 (500MB image file)
│   ├── raid0-disk1.img   ← appears after Q16 (300MB image file)
│   └── raid0-disk2.img   ← appears after Q16 (300MB image file)
└── mounts/               ← always empty on Mac side
                             (actual mounts happen inside the VM only)
```

**Q1–Q10 create zero files on your Mac.** They only read, measure, and write temporarily inside the VM.

**Q11–Q20 create `.img` files** in `/tmp/px-lab/disks/` on your Mac — these are just plain files, not real disks. They cannot harm macOS.

To clean up after all labs are done:

```bash
# On your Mac terminal (not inside the container)
rm -rf ~/px-lab
```

### What "Break and Fix" Means in Section D

Section D simulates disk failures and RAID degradation. These sound scary but:

- `mdadm --fail /dev/md0` marks a **virtual disk** (loop device inside the VM) as failed
- Nothing on your Mac fails
- The RAID array exists only in the VM's kernel memory
- Closing Docker Desktop wipes it completely

---

## Section A — Observe (Questions 1–5)

> Goal: See what storage looks like from inside a container. Understand what the container sees vs the host.

---

### Q1 — What Disks Exist Inside This Container?

**Scenario**: You are a new SRE joining a team. Your first task is to audit what storage is available on a Linux host. You SSH in and need to understand the storage landscape.

**Task**:
1. List all block devices visible inside the container
2. Identify: what type of device is it (disk, partition, loop)?
3. Find the total size of each device
4. Identify which device the container's root filesystem is mounted on

**Question to answer**: Where is the container's storage actually coming from? Is it a real disk?

---

### Q2 — How Much Space Does the Container Have, and Where Is It?

**Scenario**: A developer reports their container is running out of space. You need to find out: how much space exists, what is using it, and where the storage physically lives.

**Task**:
1. Check filesystem-level usage (how full is each mounted filesystem?)
2. Check which directory is using the most space in `/`
3. Find the type of filesystem the container root is using
4. Understand the word "overlay" — what does it mean here?

**Question to answer**: If you write a 1GB file inside the container, where does it actually go — on the Mac's disk or somewhere else?

---

### Q3 — What Is the Difference Between Storage the Container Sees vs The Host Has?

**Scenario**: You are debugging a disk space issue. The container says it has 50GB free but you know the Mac only has 20GB free on disk. You need to understand how this is possible.

**Task**:
1. Inside the container: check available space with `df -h /`
2. On your Mac (open a second terminal): check available space with `df -h`
3. Check what Docker is using: on Mac, open Docker Desktop → Settings → Resources
4. Understand the Docker VM disk that sits between Mac and container

**Question to answer**: Draw a diagram (in your notes) of: Mac SSD → Docker VM Virtual Disk → Container Overlay Filesystem → Container Root `/`

---

### Q4 — What Happens to Data When You Write Inside the Container?

**Scenario**: A junior developer writes a 500MB file inside a container and then deletes the container. They are shocked when the file is gone. You need to explain why this happens using what you can observe.

**Task**:
1. Create a 100MB file inside the container at `/root/test-ephemeral.bin`
2. Verify it exists and check where it lives in the filesystem
3. Check the overlay filesystem layers
4. Record the container ID
5. **Do not delete it yet** — just understand that this data lives in the container's write layer

**Question to answer**: What Linux filesystem concept explains why this data disappears when the container is removed?

---

### Q5 — Read-Only vs Read-Write Storage Layers

**Scenario**: You are explaining container storage to a developer. They ask: "Why can I modify `/etc/nginx.conf` inside the container if it came from the Docker image (which is read-only)?"

**Task**:
1. Create a test file in `/etc/test-layer.txt` inside the container
2. Check if `/etc/` is writable: `ls -la /etc/test-layer.txt`
3. Find where this file actually lives using `mount | grep overlay`
4. Look at the `upperdir` of the overlay mount to find your file

**Question to answer**: Explain copy-on-write in one paragraph using what you observed. What is the `upperdir` and `lowerdir` in the overlay mount output?

---

## Section B — Measure (Questions 6–10)

> Goal: Measure IOPS, throughput, and latency. Build the habit of measuring before assuming.

---

### Q6 — How Fast Can This Storage Write Sequentially?

**Scenario**: A team wants to store large video files (1GB+ each) in a container. You need to tell them the sequential write speed of the underlying storage before they commit to this approach.

**Task**:
1. Use `dd` to write a 500MB file sequentially and measure the speed
2. Run it 3 times and record each result
3. Calculate the average write throughput in MB/s
4. Now read the same file back and measure read speed

**Record in a table**:

| Run | Write MB/s | Read MB/s |
|-----|-----------|----------|
| 1   | | |
| 2   | | |
| 3   | | |
| Avg | | |

**Question to answer**: Is this fast enough for writing 10 video files simultaneously, each at 100MB/s write speed?

---

### Q7 — How Fast Is Random 4KB I/O? (The Database Question)

**Scenario**: A database team wants to run PostgreSQL in this container. Databases do random 4KB reads and writes — not sequential. You need to measure the actual random IOPS this storage can handle.

**Task**:
1. Use `fio` to measure random write IOPS at 4KB block size
2. Use `fio` to measure random read IOPS at 4KB block size
3. Compare these numbers to the sequential throughput from Q6

**Question to answer**: If PostgreSQL needs 5,000 random write IOPS minimum, can this storage handle it? Would you run a production database here?

---

### Q8 — What Is Write Latency on This Storage?

**Scenario**: An etcd cluster is showing election timeouts. etcd needs write latency under 10ms (ideally under 1ms). You need to measure the actual write latency on the storage being used.

**Task**:
1. Use `fio` with `--sync=1` to force synchronous writes (simulates fsync behavior)
2. Measure: mean latency, 99th percentile latency, 99.9th percentile latency
3. Compare to: etcd requirement (<10ms), NVMe typical (0.02–0.1ms), SSD typical (0.1–1ms), HDD typical (5–10ms)

**Question to answer**: Based on your measurement, is this storage appropriate for etcd? For a database? For log archival?

**Classification to fill**:

| Use Case | Latency Required | Your Measured Latency | Suitable? |
|----------|-----------------|----------------------|-----------|
| etcd | < 10ms | | |
| PostgreSQL primary | < 5ms | | |
| Log archive writes | < 500ms | | |

---

### Q9 — The Block Size Tradeoff: IOPS vs Throughput

**Scenario**: You have a storage system that can do 10,000 IOPS. A developer asks: "Can I get 10GB/s throughput from this?" You need to prove with math and measurement why the answer depends on block size.

**Task**:
1. Run fio with block size 4KB and record: IOPS, throughput (MB/s)
2. Run fio with block size 64KB and record: IOPS, throughput (MB/s)
3. Run fio with block size 1MB and record: IOPS, throughput (MB/s)
4. Fill the table below

| Block Size | IOPS | Throughput MB/s | Calculation |
|-----------|------|----------------|-------------|
| 4KB | | | IOPS × 4KB = ? |
| 64KB | | | IOPS × 64KB = ? |
| 1MB | | | IOPS × 1MB = ? |

**Question to answer**: What stays roughly constant as block size changes? What changes? Write the formula that connects IOPS, block size, and throughput.

---

### Q10 — Watch I/O Happen in Real Time With iostat

**Scenario**: A production incident is in progress. Someone says "the disk is saturated." You need to verify this claim in real time using standard Linux tools — not vendor dashboards.

**Task**:
1. Open **two terminal sessions** into the container (use `podman exec -it px-storage-lab bash` or `docker exec -it px-storage-lab bash` for the second)
2. Terminal 1: run `iostat -x 1` and let it run
3. Terminal 2: run a fio workload in the background
4. In Terminal 1, observe the iostat output change in real time

**Observe and record**:
- Before fio runs: what is `%util`? What is `await`?
- While fio runs: what is `%util`? What is `r/s` or `w/s`?
- After fio ends: what happens?

**Question to answer**: Explain what each of these iostat columns means: `r/s`, `w/s`, `rkB/s`, `wkB/s`, `await`, `%util`. Which column tells you a disk is saturated?

---

## Section C — Build (Questions 11–15)

> Goal: Create virtual storage from scratch. Every concept you read in Module 01 becomes something you built with your hands.
>
> **Your setup reminder**: MacBook (macOS) → Docker/Podman VM (Linux) → Ubuntu container. All commands below run inside the container. Loop devices live in the VM kernel — not on your Mac.

---

### Q11 — Create a Virtual Block Device (Loop Device)

**Scenario**: In your team's development environment, there are no spare physical disks to test Portworx installation. You need to simulate a disk using only the filesystem. This is exactly how Portworx dev environments and CI pipelines work — no real hardware needed.

**Task**:
1. Create a 500MB disk image file at `/tmp/px-lab/disks/disk01.img`
2. Attach it to a loop device using `losetup -f --show`
3. Verify it appears as a block device using `lsblk`
4. Check its size: does `lsblk` show 500MB?
5. Confirm it behaves like a real block device: `ls -la $DISK01` — what does the first character tell you?

**Container reality**: The loop device (`/dev/loop0`) is created inside the Docker/Podman VM's Linux kernel, not on macOS. Your Mac never sees `/dev/loop0`. The image file (`disk01.img`) lives at `/tmp/px-lab/disks/` inside the container, which maps to `~/px-lab/disks/` on your Mac — so the file persists even if the container restarts.

**Question to answer**: What is a loop device? Why does the OS treat a regular file as a block device? Why does `lsmod | grep loop` show nothing even though loop devices work fine in this container?

---

### Q12 — Partition the Virtual Disk

**Scenario**: A new SSD arrived in the data center. You need to partition it before the storage team can use it. More importantly, you need to understand partitioning before you can understand why Portworx deliberately skips it.

**Task**:
1. Using the loop device from Q11, create a GPT partition table with `parted`
2. Create two partitions: first 50%, second 50%
3. After `parted` completes, re-attach the loop device with partition scanning enabled (the critical container step — see note below)
4. Verify `lsblk` shows `loop0p1` and `loop0p2` as child devices
5. Check the partition table with `parted $DISK01 print`

> **Container Note**: In a normal Linux server, `partprobe` or `kpartx` notify the kernel about new partitions. Inside a container, these tools are unreliable because the container shares the VM kernel but lacks full udev support. The correct approach is to detach the loop device and re-attach it with `--partscan`, which forces the kernel to read the partition table at attach time. The solution explains exactly how.

**After completing**: Portworx uses entire raw block devices (`useAll: true` in StorageCluster) without any partitioning. Once you see how partition scanning works — and how it can fail in containers — the Portworx design decision will make sense.

**Question to answer**: What is the difference between GPT and MBR partition tables? Why does Portworx prefer raw unpartitioned disks over partitioned ones?

---

### Q13 — Format with ext4 and Explore Inodes

**Scenario**: You are setting up a log aggregation server for a microservices platform. A developer says "we will write one log file per request — that could be millions of tiny 1KB files per day." You need to understand inode limits before this goes to production and pages you at 3AM.

**Task**:
1. Format the first partition (`${DISK01}p1`) from Q12 with ext4
2. Mount it at `/tmp/px-lab/mounts/ext4test`
3. Run `df -i` — how many inodes does a 250MB ext4 partition get by default?
4. Create 100 small files and watch inode usage climb
5. Now create a new 100MB image, format with `mkfs.ext4 -N 500` (only 500 inodes), mount it, then write 600 files
6. Observe the exact error message when inodes run out

> **Container Note**: `${DISK01}p1` refers to the first partition of the re-attached loop device from Q12. This only works after the `--partscan` re-attach step. If you skipped that step, the device node `loop0p1` will not exist.

**Question to answer**: What is an inode? What error do you get when inodes are exhausted but disk space is not? Why is this error message misleading — and what two commands do you run to diagnose it correctly?

---

### Q14 — Format with XFS and Compare to ext4

**Scenario**: Portworx creates XFS filesystems on every volume it manages. You are asked in a job interview: "Why does Portworx use XFS instead of ext4?" You need to be able to answer from first-hand observation, not just documentation.

**Task**:
1. Create a new 500MB disk image at `/tmp/px-lab/disks/disk02.img` and attach as a second loop device (`$DISK02`)
2. Format it with XFS: `mkfs.xfs $DISK02` (format the full device, not a partition)
3. Mount it at `/tmp/px-lab/mounts/xfstest`
4. Run `df -i` on both mounts — what does XFS report for inode count?
5. Get detailed filesystem info: `tune2fs -l ${DISK01}p1` (ext4) and `xfs_info /tmp/px-lab/mounts/xfstest` (XFS)
6. Attempt to shrink XFS: `xfs_growfs -d /tmp/px-lab/mounts/xfstest` — what is the result?

> **Container Note**: For ext4 info, `tune2fs -l ${DISK01}p1` uses the partition device from Q12. For XFS, `xfs_info` takes the mount point, not the device path. Both work correctly inside the container.

**Fill this comparison table** (complete it from your observations):

| Property | ext4 | XFS |
|----------|------|-----|
| Inode allocation | Fixed at format time | ? |
| Max filesystem size | 1 EB | ? |
| Can shrink online? | No | ? |
| Can grow online? | Yes | ? |
| Performance at large file count | Good | ? |
| Default journal mode | ordered | ? |
| Inode exhaustion risk | Yes | ? |

**Question to answer**: Give two specific reasons — from what you observed in this lab — why Portworx defaults to XFS. Connect each reason to a real-world database or production workload.

---

### Q15 — Simulate Inode Exhaustion (Production Incident)

**Scenario**: It is 3AM. PagerDuty fires: "Elasticsearch pod cannot write to its data volume. Error: No space left on device." You SSH into the node and run `df -h` — the disk is only 18% full. Your heart sinks. Something is wrong at a level below disk space. Walk through the diagnosis.

**Task**:
1. Create a 200MB ext4 filesystem limited to 1000 inodes: `mkfs.ext4 -N 1000`
2. Mount it at `/tmp/px-lab/mounts/inodecrash`
3. Run the file-creation loop below until it fails — capture the exact error
4. Run `df -h` (shows space available) then `df -i` (shows inodes at 100%) — compare the two outputs
5. Find the directory with the most files using `find`
6. Write the 3-step production diagnostic procedure

**The script to run inside the container**:
```bash
for i in $(seq 1 1100); do
  result=$(touch /tmp/px-lab/mounts/inodecrash/file_$i 2>&1)
  if [ -n "$result" ]; then
    echo "FAILED at file $i: $result"
    break
  fi
done
```

> **Container Note**: This entire exercise runs inside the Ubuntu container on the Podman/Docker VM. The inode exhaustion happens on a loop-device-backed filesystem inside the container — zero impact on your Mac or on the VM's own filesystem.

**Real-world connection**: Elasticsearch writes one segment file per indexed document batch. Under high ingest load, this can create thousands of small files per hour on the data volume. An ELK stack (Elasticsearch + Logstash + Kibana) running on a Portworx XFS volume never hits this problem — XFS allocates inodes dynamically. The same stack on an ext4 volume with default inode density can exhaust inodes within days under heavy log ingestion.

**Question to answer**: Write the 3-step diagnostic for "No space left on device" errors. What is the first command? What does each step rule out? What is the fix for inode exhaustion (hint: you cannot just delete data)?

---

## Section D — Break and Fix (Questions 16–20)

> Goal: Simulate RAID, failure, and recovery. These are the scenarios that happen in production.

---

### Q16 — Create RAID 0 and Measure Striped Performance

**Scenario**: A data analytics team says their current single disk cannot give them enough throughput for their ETL jobs. They want to know if RAID 0 across two disks would double their throughput. You need to test this.

**Task**:
1. Create two 300MB disk images and attach them as loop devices
2. Create a RAID 0 array (`mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/loopX /dev/loopY`)
3. Format RAID 0 with XFS and mount it
4. Measure sequential write throughput on RAID 0
5. Compare to single disk throughput from Q6

**Question to answer**: Did RAID 0 double your throughput? If not, why not? What is the storage capacity of RAID 0 vs using one disk? What happens to your data if one disk fails?

---

### Q17 — Create RAID 1 and Survive a Disk Failure

**Scenario**: A critical service is running on a single disk with no redundancy. The SRE team has been asked to add redundancy without downtime. You need to test RAID 1 behavior, including how data survives after a disk failure.

**Task**:
1. Create two 300MB disk images → attach as loop devices
2. Create RAID 1: `mdadm --create /dev/md1 --level=1 --raid-devices=2 /dev/loopX /dev/loopY`
3. Format and mount
4. Write a file: `echo "critical production data" > /mnt/raid1/important.txt`
5. **Simulate disk failure**: `mdadm --fail /dev/md1 /dev/loopX`
6. Check RAID status: `cat /proc/mdstat`
7. Read the file: is the data still there?
8. Check health: `mdadm --detail /dev/md1`
9. **Simulate replacement**: remove failed disk, add new disk, watch rebuild

**Question to answer**: What does the RAID status show as "degraded"? What is the risk during rebuild? Why should you never ignore a degraded RAID array?

---

### Q18 — Storage Capacity Math (RAID vs Raw)

**Scenario**: Your team is buying 6 disks × 10TB each for a Portworx cluster. The finance team asks: "How much usable storage will we actually have?" The answer depends entirely on the RAID or replication strategy chosen.

**Task** (calculation exercise — no disk creation needed):

Given: 6 × 10TB disks

Fill this table:

| Strategy | Total Raw | Usable Storage | Fault Tolerance | Formula |
|----------|-----------|---------------|-----------------|---------|
| No RAID (JBOD) | 60TB | | None | |
| RAID 0 (2 disks) | 60TB | | | |
| RAID 1 (pairs) | 60TB | | | |
| RAID 5 (6 disks) | 60TB | | | (N-1)/N × Total |
| RAID 6 (6 disks) | 60TB | | | (N-2)/N × Total |
| RAID 10 (6 disks) | 60TB | | | |
| Portworx repl=3 | 60TB | | | |

Then answer:
1. Which option gives the most usable space?
2. Which option gives the best fault tolerance?
3. Portworx replication factor 3 behaves most like which RAID level?
4. If Portworx repl=3 and you have 60TB raw — how much usable storage do you have?

---

### Q19 — Simulate Disk Full and Understand the Impact

**Scenario**: A production database hits a "disk full" error at 2AM. The alert says the Portworx pool is 100% utilized. You need to understand what happens when storage is completely full — and what your emergency options are.

**Task**:
1. Create a 100MB disk image and format it with ext4
2. Mount it at `/tmp/px-lab/mounts/fulltest`
3. Write files until the disk is completely full
4. Observe the exact error
5. Check with `df -h` — what does 100% look like?
6. Try to fix without adding a disk:
   - Delete the largest file and verify writes work again
   - Check for hidden space (ext4 reserves 5% for root)
   - Calculate how much the reserved space is

**Script to fill the disk**:
```bash
while true; do
  dd if=/dev/urandom of=/tmp/px-lab/mounts/fulltest/fill_$(date +%s%N).bin \
     bs=1M count=10 status=none 2>&1
  if [ $? -ne 0 ]; then
    echo "Disk is full"
    df -h /tmp/px-lab/mounts/fulltest
    break
  fi
done
```

**Question to answer**: What does ext4 "reserved blocks" mean? Why does a `root` user see more free space than a regular user? What is the `tune2fs` command to adjust reserved space in an emergency?

---

### Q20 — End-to-End: The Slow Disk Investigation

**Scenario**: Production incident. A Portworx volume is showing high latency. Application is slowing down. You are on-call and have 15 minutes to diagnose before the SLA breach. You must follow a systematic investigation — no guessing.

**Task**: Simulate a "slow disk" scenario and perform the full investigation.

**Setup (run this to simulate the problem)**:
```bash
# Create a "disk" with limited I/O using cgroups throttling
# (simulated via a slow fio background job filling I/O)
dd if=/dev/zero of=/tmp/px-lab/disks/slowdisk.img bs=1M count=200 status=none
SLOWDISK=$(losetup -f --show /tmp/px-lab/disks/slowdisk.img)
mkfs.ext4 $SLOWDISK -q
mkdir -p /tmp/px-lab/mounts/slowdisk
mount $SLOWDISK /tmp/px-lab/mounts/slowdisk

# Run a background process saturating the disk
fio --name=saturate --rw=randwrite --bs=4k --size=100M \
    --numjobs=4 --iodepth=32 --filename=/tmp/px-lab/mounts/slowdisk/saturate \
    --runtime=120 --time_based --output=/dev/null &
SATPID=$!
echo "Disk saturated. PID=$SATPID — now investigate!"
```

**Your investigation procedure** (do not look at solutions first):

Step 1: Identify which device is the problem
Step 2: Measure current I/O wait and throughput
Step 3: Find which process is causing the I/O
Step 4: Measure the latency
Step 5: Document your findings in the format below and propose a fix

**Findings template**:
```
Device: ____________
%util: ____________
await (ms): ____________
Culprit process: ____________
Write IOPS: ____________
Throughput: ____________
Root cause: ____________
Proposed fix: ____________
```

**Question to answer**: What is your 5-step "slow disk" runbook? Write it from memory after completing this investigation.

---

## Progress Checklist

```
Section A — Observe
[ ] Q1  — List block devices and identify types
[ ] Q2  — Measure filesystem usage and understand overlay
[ ] Q3  — Container vs host storage comparison
[ ] Q4  — Understand ephemeral writes and overlay write layer
[ ] Q5  — Copy-on-write and overlay layers

Section B — Measure
[ ] Q6  — Sequential throughput benchmark
[ ] Q7  — Random 4KB IOPS benchmark
[ ] Q8  — Write latency measurement
[ ] Q9  — Block size vs IOPS vs throughput tradeoff
[ ] Q10 — Real-time I/O observation with iostat

Section C — Build
[ ] Q11 — Create loop device (virtual disk)
[ ] Q12 — Partition the virtual disk
[ ] Q13 — Format with ext4, explore inodes
[ ] Q14 — Format with XFS, compare to ext4
[ ] Q15 — Trigger inode exhaustion, diagnose it

Section D — Break and Fix
[ ] Q16 — RAID 0 striping and performance measurement
[ ] Q17 — RAID 1 mirroring and disk failure survival
[ ] Q18 — Storage capacity math (all RAID levels)
[ ] Q19 — Disk full simulation and emergency recovery
[ ] Q20 — End-to-end slow disk investigation
```

---

When you have attempted all 20 questions, go to [02-solutions.md](./02-solutions.md) for the full solutions, explanations, and the thinking behind each.
