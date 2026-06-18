# Module 02 — Linux Storage

```
Phase       : 1 — Storage Fundamentals
Module      : 02
Status      : draft
Prereqs     : Module 01 — Storage From First Principles
Estimated   : 5–7 hours
```

---

## Why This Module Exists

Portworx runs inside Linux. Every volume operation — provisioning, mounting, resizing, unmounting — is a sequence of Linux kernel operations. When something breaks, the errors surface in Linux: `dmesg`, `/proc`, `sysfs`, `udev`.

If you don't understand how Linux manages disks, you will read error messages and not know what they mean.

---

## Learning Objectives

After completing this module, you will be able to:

1. Trace the path from raw disk to mounted filesystem in Linux
2. Use `lsblk`, `blkid`, `df`, `du`, `mount`, `fdisk`, `parted` fluently
3. Explain inodes and why they matter when a filesystem shows "full" but `df` disagrees
4. Create and manage LVM volumes (what Portworx does internally for local volumes on some backends)
5. Explain Device Mapper and how it powers LVM, LUKS, and thin provisioning
6. Describe loop devices and how Portworx uses them in development environments
7. Compare ext4 vs XFS — pick the right one and explain why
8. Debug common Linux storage failures from first symptoms to root cause

---

## Prerequisites

- Module 01 completed
- Linux terminal access (local VM or cloud instance)
- Root/sudo access

---

## Concepts to Study

### 1. The Linux Storage Stack

```
Application (read/write syscall)
       ↓
VFS (Virtual Filesystem Switch)
       ↓
Filesystem (ext4 / xfs)
       ↓
Block Layer (request queue, scheduler)
       ↓
Device Mapper (optional — LVM, encryption, thin provisioning)
       ↓
Block Device Driver
       ↓
Physical Disk (/dev/sda, /dev/nvme0n1)
```

Every layer adds abstraction. Every layer can add latency. Every layer can fail.

**Portworx sits between the Filesystem and Physical Disk** — it intercepts block I/O and replicates it to other nodes before acknowledging the write.

---

### 2. Disk → Partition → Filesystem → Mount

The fundamental flow to memorize:

```
/dev/sda         ← raw block device (the disk)
/dev/sda1        ← partition (carved from the disk)
mkfs.ext4        ← filesystem written onto partition
mount            ← filesystem attached to directory tree
/data            ← application reads/writes here
```

**Commands**:

```bash
# List all block devices
lsblk

# Output example:
# NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
# sda      8:0    0  100G  0 disk
# ├─sda1   8:1    0   99G  0 part /
# └─sda2   8:2    0    1G  0 part [SWAP]

# Show filesystem type and UUID
blkid /dev/sda1

# Show disk usage (filesystem level)
df -h

# Show directory usage (file level)
du -sh /var/log

# Show what is mounted
mount | column -t

# Mount a device
mount /dev/sda1 /mnt/data

# Unmount
umount /mnt/data
```

**Key insight**: `df` shows free space at the filesystem level. `du` shows space consumed by files. If `df` says full but `du` shows files using much less, you likely have deleted-but-open files or inode exhaustion.

---

### 3. Partitioning

Two partition table formats:

**MBR (Master Boot Record)**
- Legacy, max 4 primary partitions, max 2TB disk
- Tool: `fdisk`

**GPT (GUID Partition Table)**
- Modern, 128 partitions, no size limit
- Tool: `parted` or `gdisk`

```bash
# View partition table
sudo fdisk -l /dev/sda

# Interactive partitioning with fdisk
sudo fdisk /dev/sda

# View and modify with parted
sudo parted /dev/sda print
sudo parted /dev/sda mkpart primary ext4 0% 100%

# After partitioning — create filesystem
sudo mkfs.ext4 /dev/sda1
sudo mkfs.xfs /dev/sda2
```

**Portworx connection**: When Portworx claims a disk for its storage pool, it does NOT create a partition — it uses the whole raw block device. This is intentional: fewer abstraction layers, lower overhead.

---

### 4. Filesystems — ext4 vs XFS

**ext4**

```
Default Linux filesystem
Journaling: writeback, ordered, journaled
Max file size: 16TB
Max filesystem: 1EB
Resize: online grow, offline shrink
```

- Mature, battle-tested
- Good for general purpose workloads
- Default for many Linux distros

**XFS**

```
High-performance journaling filesystem (Silicon Graphics origin)
Excellent for large files and parallel I/O
Max file size: 8EB
Max filesystem: 8EB
Resize: online grow only (cannot shrink)
```

- Better performance at scale
- Better concurrent I/O (multiple write streams)
- Kubernetes default for many storage systems including Portworx volumes
- Portworx creates XFS filesystems on its volumes by default

```bash
# Create ext4
sudo mkfs.ext4 /dev/sda1

# Create XFS
sudo mkfs.xfs /dev/sda2

# XFS info
xfs_info /dev/sda2

# ext4 info
tune2fs -l /dev/sda1

# Check filesystem health (unmounted)
fsck.ext4 /dev/sda1
xfs_repair /dev/sda2
```

**When Portworx formats a volume**, it uses XFS by default because XFS handles concurrent random I/O better — critical for database workloads.

---

### 5. Inodes

```
Inode = metadata structure for a file or directory
        (NOT the data itself)

Inode contains:
  - File size
  - Permissions
  - Owner/group
  - Timestamps
  - Pointer to data blocks

Does NOT contain:
  - Filename (that lives in the directory entry)
```

**Why inodes matter**:

A filesystem has a fixed number of inodes created at format time. If you run out of inodes, you cannot create new files — even if there is free disk space.

```bash
# Check inode usage
df -i

# Output:
# Filesystem      Inodes  IUsed   IFree IUse% Mounted on
# /dev/sda1      6553600 200000 6353600    3% /

# Find which directory is consuming inodes
find / -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head -20
```

**Production incident pattern**:
> A pod cannot write to a volume. `df -h` shows 60% free. The pod log says "No space left on device." Root cause: inode exhaustion from millions of small temporary files.

**XFS vs ext4 inode handling**:
- ext4: fixed inode count at format time
- XFS: dynamic inode allocation — essentially never runs out of inodes

This is another reason Portworx prefers XFS.

---

### 6. Journaling

A journal is a write-ahead log for filesystem metadata:

```
Without journal:
  Power failure mid-write → filesystem inconsistency → fsck on reboot (slow)

With journal:
  Write metadata to journal first → apply to filesystem → mark complete
  Power failure → replay journal → consistent state (fast recovery)
```

**Journal modes (ext4)**:

| Mode | What is journaled | Performance | Safety |
|------|------------------|-------------|--------|
| `journal` | data + metadata | Slowest | Highest |
| `ordered` | metadata only, data written first | Default | Good |
| `writeback` | metadata only, data order not guaranteed | Fastest | Lower |

**Why Portworx uses a journal disk**:
Portworx maintains its own internal journal (separate from the filesystem journal) to ensure write ordering across replicas. A dedicated NVMe journal disk means this journal write is extremely fast — reducing write latency for the entire Portworx cluster.

---

### 7. LVM (Logical Volume Manager)

LVM adds a virtualization layer between disks and filesystems:

```
Physical Volumes (PV)    → /dev/sda, /dev/sdb (raw disks or partitions)
       ↓
Volume Group (VG)        → pool combining multiple PVs
       ↓
Logical Volume (LV)      → virtual disk carved from VG
       ↓
Filesystem               → ext4 / xfs on top of LV
```

**Benefits**:
- Resize volumes without downtime
- Snapshot support
- Span multiple disks seamlessly
- Thin provisioning

```bash
# View LVM state
pvs       # physical volumes
vgs       # volume groups
lvs       # logical volumes

# Create PV from disk
sudo pvcreate /dev/sdb

# Create VG
sudo vgcreate vg_data /dev/sdb

# Create LV (50GB)
sudo lvcreate -L 50G -n lv_data vg_data

# Format and mount
sudo mkfs.xfs /dev/vg_data/lv_data
sudo mount /dev/vg_data/lv_data /mnt/data

# Extend LV online
sudo lvextend -L +20G /dev/vg_data/lv_data
sudo xfs_growfs /mnt/data    # for XFS
sudo resize2fs /dev/vg_data/lv_data    # for ext4
```

**Portworx connection**: Portworx uses LVM internally on some backends to manage its storage pools. Understanding LVM makes `pxctl` storage pool operations intuitive.

---

### 8. Device Mapper

Device Mapper is the Linux kernel framework that powers:
- LVM
- dm-crypt (LUKS encryption)
- dm-thin (thin provisioning)
- dm-multipath (multiple paths to same LUN)
- Docker overlay storage

```bash
# View device mapper devices
ls /dev/mapper/

# Show device mapper table
sudo dmsetup table
sudo dmsetup ls

# Show device mapper status
sudo dmsetup status
```

**Portworx uses Device Mapper** for:
- Thin provisioning of volumes (over-provisioning)
- Volume snapshots (copy-on-write via dm-thin)
- Encryption (when PX-Security is enabled)

---

### 9. Loop Devices

A loop device lets you use a file as if it were a block device:

```
/tmp/disk.img (file)
       ↓
losetup /dev/loop0
       ↓
/dev/loop0 (treated as block device)
       ↓
mkfs.ext4 /dev/loop0
       ↓
mount /dev/loop0 /mnt/test
```

```bash
# Create a 1GB disk image
dd if=/dev/zero of=/tmp/test.img bs=1M count=1024

# Attach to loop device
sudo losetup /dev/loop0 /tmp/test.img

# Or auto-find next free loop device
sudo losetup -f --show /tmp/test.img

# Format and use
sudo mkfs.ext4 /dev/loop0
sudo mount /dev/loop0 /mnt/test

# List loop devices
losetup -l

# Detach
sudo losetup -d /dev/loop0
```

**Portworx connection**: In Kind/Minikube/K3d environments, Portworx uses loop devices backed by image files to simulate disks. This is why lab instructions ask you to pre-create disk images before installing Portworx in test environments.

---

## Command Reference

### Essential Commands

```bash
# Block devices
lsblk                          # tree view of all block devices
lsblk -f                       # include filesystem type and UUID
blkid /dev/sda1                # filesystem UUID and type
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT

# Disk usage
df -h                          # filesystem usage (human readable)
df -i                          # inode usage
du -sh /path/to/dir            # directory size
du -sh /* 2>/dev/null          # top-level directory sizes

# Filesystem
mount                          # list mounted filesystems
mount -t ext4 /dev/sda1 /mnt  # mount with explicit type
umount /mnt                    # unmount
findmnt                        # tree view of mounts
findmnt /dev/sda1              # where is this device mounted?

# Partitioning
sudo fdisk -l                  # list all disks and partitions
sudo parted -l                 # list all disks (GPT-aware)
sudo fdisk /dev/sdb            # interactive partitioning

# Filesystem creation
sudo mkfs.ext4 /dev/sdb1
sudo mkfs.xfs /dev/sdb1
sudo mkfs.xfs -f /dev/sdb1    # force (overwrite existing)

# Filesystem info
xfs_info /dev/sdb1             # XFS filesystem details
sudo tune2fs -l /dev/sdb1      # ext4 filesystem details
sudo dumpe2fs /dev/sdb1 | head -50

# Filesystem check (unmounted only)
sudo fsck.ext4 /dev/sdb1
sudo xfs_repair /dev/sdb1

# LVM
pvs && vgs && lvs              # quick LVM overview
pvdisplay && vgdisplay && lvdisplay   # verbose
sudo pvcreate /dev/sdb
sudo vgcreate vg0 /dev/sdb
sudo lvcreate -L 10G -n lv0 vg0
sudo lvextend -L +5G /dev/vg0/lv0

# I/O monitoring
iostat -x 1                    # extended I/O stats every second
iotop                          # per-process I/O
dstat                          # combined system stats
```

---

## Hands-On Labs

### Lab 2.1 — Build the Disk → Mount Path

```bash
# Create a virtual disk
dd if=/dev/zero of=/tmp/lab2.img bs=1M count=500
sudo losetup -f --show /tmp/lab2.img
# Note the loop device (e.g., /dev/loop5)

# Partition it
sudo parted /dev/loop5 mklabel gpt
sudo parted /dev/loop5 mkpart primary xfs 0% 100%

# Format with XFS
sudo mkfs.xfs /dev/loop5p1

# Mount it
sudo mkdir -p /mnt/lab2
sudo mount /dev/loop5p1 /mnt/lab2

# Verify
lsblk /dev/loop5
df -h /mnt/lab2
xfs_info /dev/loop5p1

# Write data
echo "persistent data" | sudo tee /mnt/lab2/test.txt
cat /mnt/lab2/test.txt

# Unmount and remount to verify persistence
sudo umount /mnt/lab2
sudo mount /dev/loop5p1 /mnt/lab2
cat /mnt/lab2/test.txt   # data should still be there

# Cleanup
sudo umount /mnt/lab2
sudo losetup -d /dev/loop5
rm /tmp/lab2.img
```

### Lab 2.2 — LVM Thin Provisioning

```bash
# Create two virtual disks
dd if=/dev/zero of=/tmp/disk1.img bs=1M count=500
dd if=/dev/zero of=/tmp/disk2.img bs=1M count=500
LOOP1=$(sudo losetup -f --show /tmp/disk1.img)
LOOP2=$(sudo losetup -f --show /tmp/disk2.img)

# Create PVs
sudo pvcreate $LOOP1 $LOOP2

# Create VG spanning both
sudo vgcreate vg_lab $LOOP1 $LOOP2

# Create thin pool (reserve 900MB for pool)
sudo lvcreate -L 900M --thinpool tp0 vg_lab

# Create thin volumes (over-provisioned!)
sudo lvcreate -V 400M --thin -n lv_app1 vg_lab/tp0
sudo lvcreate -V 400M --thin -n lv_app2 vg_lab/tp0

# Format and mount
sudo mkfs.ext4 /dev/vg_lab/lv_app1
sudo mkfs.ext4 /dev/vg_lab/lv_app2
sudo mkdir -p /mnt/app1 /mnt/app2
sudo mount /dev/vg_lab/lv_app1 /mnt/app1
sudo mount /dev/vg_lab/lv_app2 /mnt/app2

# Verify — two 400MB volumes from 900MB pool (thin)
lvs vg_lab
df -h /mnt/app1 /mnt/app2

# Cleanup
sudo umount /mnt/app1 /mnt/app2
sudo lvremove -f vg_lab
sudo vgremove vg_lab
sudo pvremove $LOOP1 $LOOP2
sudo losetup -d $LOOP1 $LOOP2
rm /tmp/disk1.img /tmp/disk2.img
```

### Lab 2.3 — Trigger and Debug Inode Exhaustion

```bash
# Create a small filesystem
dd if=/dev/zero of=/tmp/inodetest.img bs=1M count=100
LOOP=$(sudo losetup -f --show /tmp/inodetest.img)
# Create ext4 with very few inodes
sudo mkfs.ext4 -N 1000 $LOOP
sudo mkdir -p /mnt/inodetest
sudo mount $LOOP /mnt/inodetest

# Create files until inode exhaustion
for i in $(seq 1 1100); do
  touch /mnt/inodetest/file_$i 2>&1
done

# Observe the error
df -h /mnt/inodetest      # space looks fine
df -i /mnt/inodetest      # inodes exhausted

# Cleanup
sudo umount /mnt/inodetest
sudo losetup -d $LOOP
rm /tmp/inodetest.img
```

---

## Debugging Scenarios

### Scenario 1: "No space left on device" but `df` shows free space

```bash
# Step 1: Check inode usage
df -i /mount/point

# Step 2: Find which directory has millions of files
find /mount/point -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head

# Step 3: Check for deleted-but-open files
lsof +L1 | grep /mount/point   # files deleted but held open by process
# If found, restart the process to release file handles

# Step 4: Check for reserved space (ext4 reserves 5% for root)
tune2fs -l /dev/sda1 | grep -i reserved
```

### Scenario 2: Mount fails — "wrong fs type"

```bash
# Check what filesystem is actually on the device
sudo blkid /dev/sdb1

# Check kernel has the filesystem module
lsmod | grep xfs
sudo modprobe xfs

# Try manual mount with explicit type
sudo mount -t xfs /dev/sdb1 /mnt/test
```

### Scenario 3: I/O hangs — disk not responding

```bash
# Check dmesg for disk errors
dmesg | tail -50 | grep -iE 'error|fail|i/o|reset'

# Check if device is in error state
cat /sys/block/sda/stat
cat /sys/block/sda/device/state

# Check I/O scheduler queue
cat /sys/block/sda/queue/scheduler

# Force reset (last resort)
echo 1 > /sys/block/sda/device/delete   # remove device
# Then rescan
echo "- - -" > /sys/class/scsi_host/host0/scan
```

---

## Interview Questions

1. Describe the full path from a file write syscall to bits on disk in Linux.
2. `df -h` shows 10% free on `/var`. The application still says "No space left on device." What do you investigate?
3. What is the difference between `df` and `du`? Why might they disagree?
4. A Kubernetes node shows a volume as mounted but the pod cannot write to it. What Linux commands do you run?
5. Why does Portworx prefer XFS over ext4 for its volumes?
6. What is a loop device? How is it used in Portworx test environments?
7. Explain Device Mapper in one sentence. What Portworx features depend on it?
8. What happens to an LVM volume group when one of its physical volumes fails?
9. A disk shows `%util = 100%` in iostat but `await = 0.3ms`. Is the disk saturated?
10. What is journaling and why does Portworx care about it?

---

## Production Takeaways

1. **Always check inodes first when "disk full" appears** — `df -i` takes 1 second and has saved countless incidents.

2. **XFS cannot shrink** — size your XFS volumes correctly upfront. Portworx volume resize (expand only) inherits this constraint.

3. **`dmesg` is your first stop for disk errors** — before checking anything else, run `dmesg | grep -iE 'error|fail|i/o'`.

4. **Device Mapper devices appear in `/dev/mapper/`** — if you see unexpected dm-X devices on a Portworx node, they are normal — Portworx creates dm-thin volumes for its pool.

5. **Loop devices have performance overhead** — never use loop-device-backed Portworx in production. They are for development only.

6. **LVM thin provisioning can over-commit** — if you provision more LVs than physical space, writes fail when the thin pool fills. Monitor pool usage.

7. **`findmnt` is better than `mount`** — `findmnt` shows the tree structure, making it easy to see bind mounts (which Kubernetes uses extensively for pod volumes).

---

## Success Criteria

You have completed this module when you can:

- [ ] Draw the Linux storage stack from syscall to disk from memory
- [ ] Build the full disk → partition → filesystem → mount path in a lab (Lab 2.1)
- [ ] Explain inodes and trigger inode exhaustion (Lab 2.3)
- [ ] Debug all three debugging scenarios without looking at the answers
- [ ] Explain LVM PV → VG → LV and resize a logical volume online
- [ ] Explain why Portworx uses XFS and loop devices in test environments
- [ ] Read `iostat -x` output and explain every column

---

## Next Module

[Module 03 — Why Containers Need Persistent Storage](../phase-2-kubernetes-storage/module-03-containers-and-persistent-storage.md)
