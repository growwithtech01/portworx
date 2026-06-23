# Module 01 Lab — Environment Setup

```
Environment : Docker Ubuntu container on macOS
Module      : 01 — Storage From First Principles
All labs run inside the container — not on your Mac
```

---

## Is Any of This Dangerous to My MacBook?

**No. Here is exactly why.**

---

### Docker on Mac Is Not Docker on Linux

On a Linux server, containers share the host's kernel directly:

```
┌─────────────────────────────────────┐
│  Linux Server                       │
│                                     │
│  ┌──────────┐  ┌──────────┐        │
│  │Container1│  │Container2│        │
│  └────┬─────┘  └────┬─────┘        │
│       │              │              │
│  ─────┴──────────────┴──────────   │
│        Linux Kernel (shared)        │
│  ─────────────────────────────     │
│        Physical Hardware            │
└─────────────────────────────────────┘
```

This is why `--privileged` on a Linux server is risky — the container touches the real host kernel.

**Your Mac is different.** macOS runs the Darwin (XNU) kernel — not Linux. Docker Desktop cannot share your Mac kernel with a Linux container because they are different kernels entirely.

So Docker Desktop installs a **hidden Linux VM** first, then runs containers inside that VM:

```
┌──────────────────────────────────────────────────────┐
│  Your MacBook (macOS / Darwin XNU kernel)            │
│  ← never touched by any container command            │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │  Docker Desktop VM (Apple Hypervisor)        │    │
│  │  Lightweight Linux distro (LinuxKit)         │    │
│  │                                              │    │
│  │  ┌──────────────┐  ┌──────────────┐         │    │
│  │  │ Ubuntu       │  │ Any other    │         │    │
│  │  │ Container    │  │ Container    │         │    │
│  │  └──────┬───────┘  └──────┬───────┘         │    │
│  │         │                 │                 │    │
│  │  ───────┴─────────────────┴──────────────   │    │
│  │      Linux Kernel (belongs to the VM)       │    │
│  └──────────────────────────────────────────┘  │    │
│                                                      │
│  Apple Silicon / Intel hardware                      │
└──────────────────────────────────────────────────────┘
```

---

### What This Means for Every Command in This Lab

| Command / Operation | Affects Mac? | Why |
|--------------------|-------------|-----|
| `--privileged` flag | No | Gives root access to the VM's Linux kernel, not macOS |
| Loop devices (`/dev/loop*`) | No | Created inside the VM kernel, invisible to macOS |
| `mdadm` RAID arrays | No | Software RAID runs inside the VM |
| `mkfs`, `fdisk`, `parted` | No | Operating on loop device image files inside the VM |
| `/proc/sys/vm/drop_caches` | No | The VM's `/proc`, not macOS |
| `fio`, `dd` writes | No | Write to files inside the VM filesystem |
| Killing processes, crashing VM | No | Docker Desktop restarts the VM automatically |

**Worst case**: you crash the VM. Docker Desktop shows a restart prompt. Your Mac is unaffected.

---

### The ONE Thing That Touches Your Mac

The volume mount in the `docker run` command:

```bash
-v /tmp/px-lab:/tmp/px-lab
```

This is the **only bridge** between macOS and the container:

```
macOS filesystem               VM filesystem        Container
────────────────               ─────────────        ──────────
/tmp/px-lab/       ←────────────────────────────→  /tmp/px-lab/
(on your Mac SSD)    shared via Apple Hypervisor    (inside container)
```

Files you create in `/tmp/px-lab` inside the container are stored as regular files on your Mac's SSD. They consume disk space but cannot harm macOS.

**Total disk space used across all labs**: 1–3 GB maximum.

**To clean up everything after finishing**:
```bash
# Run this on your Mac terminal (NOT inside the container)
rm -rf /tmp/px-lab
```

`/tmp` on macOS is also automatically cleared on every reboot.

---

## What You Are Building

```
Your MacBook (macOS — untouched by lab commands)
└── Docker Desktop VM (Linux VM, managed by Apple Hypervisor)
    └── Ubuntu 22.04 Container   ← you work here
        ├── /dev/loop*           ← virtual disks you create (inside VM)
        ├── /tmp/px-lab/disks/   ← image files (shared back to Mac via volume)
        ├── fio                  ← I/O benchmarking tool
        ├── sysstat (iostat)     ← real-time I/O monitoring
        ├── mdadm                ← software RAID management
        └── util-linux           ← lsblk, blkid, losetup, fdisk
```

You will create virtual disks using loop devices. These behave identically to real block devices — you can partition them, format them, create RAID arrays on them, and observe all the same kernel behavior as real hardware.

---

## Step 1 — Start the Ubuntu Container

Open your terminal on macOS and run:

```bash
docker run -it \
  --name px-storage-lab \
  --privileged \
  --cap-add=SYS_ADMIN \
  --cap-add=MKNOD \
  -v /tmp/px-lab:/tmp/px-lab \
  ubuntu:22.04 \
  bash
```

**Why `--privileged`?**
Loop devices, RAID (mdadm), and filesystem operations require elevated kernel privileges. `--privileged` gives the container root-level access to the Docker VM's Linux kernel — not to macOS. Safe on Mac because of the VM layer.

**Why `-v /tmp/px-lab:/tmp/px-lab`?**
This mounts a directory from your Mac into the container. Disk image files you create in `/tmp/px-lab/` persist even if the container restarts. This is the only directory shared between macOS and the container.

You should now see a prompt like:
```
root@abc123def456:/#
```

---

## Step 2 — Install Required Tools

Run this entire block inside the container:

```bash
apt-get update -qq && apt-get install -y \
  fio \
  sysstat \
  mdadm \
  util-linux \
  parted \
  e2fsprogs \
  xfsprogs \
  smartmontools \
  procps \
  bc \
  tree \
  lvm2 \
  --no-install-recommends \
  && echo "All tools installed successfully"
```

**Tools and what they do:**

| Tool | Package | Purpose |
|------|---------|---------|
| `fio` | fio | I/O benchmarking (IOPS, throughput, latency) |
| `iostat` | sysstat | Real-time disk I/O monitoring |
| `mdadm` | mdadm | Software RAID management |
| `lsblk`, `losetup`, `blkid` | util-linux | Block device inspection |
| `parted`, `fdisk` | parted/util-linux | Disk partitioning |
| `mkfs.ext4` | e2fsprogs | Create ext4 filesystems |
| `mkfs.xfs`, `xfs_info` | xfsprogs | Create XFS filesystems |
| `bc` | bc | Command-line math calculator |

---

## Step 3 — Prepare the Disk Image Directory

```bash
mkdir -p /tmp/px-lab/disks
mkdir -p /tmp/px-lab/mounts
echo "Lab directory ready at /tmp/px-lab"
ls -la /tmp/px-lab/
```

---

## Step 4 — Verify Your Environment

Run this verification script:

```bash
cat << 'VERIFY' > /tmp/verify-env.sh
#!/bin/bash
echo "=== Environment Verification ==="
echo ""

echo "[ TOOLS ]"
for tool in fio iostat mdadm lsblk losetup fdisk parted mkfs.ext4 mkfs.xfs; do
  if command -v $tool &>/dev/null; then
    echo "  ✓ $tool found at $(which $tool)"
  else
    echo "  ✗ $tool NOT FOUND — re-run apt-get install"
  fi
done

echo ""
echo "[ BLOCK DEVICES ]"
lsblk
echo ""

echo "[ LOOP DEVICES ]"
losetup -l 2>/dev/null || echo "  No loop devices active"
echo ""

echo "[ DISK SPACE IN CONTAINER ]"
df -h /
echo ""

echo "[ KERNEL VERSION ]"
uname -r
echo ""

echo "[ PRIVILEGED CHECK ]"
if touch /dev/testfile 2>/dev/null; then
  rm /dev/testfile
  echo "  ✓ Container has device access (privileged)"
else
  echo "  ✗ Container may not be privileged — restart with --privileged"
fi

echo ""
echo "=== Verification Complete ==="
VERIFY

chmod +x /tmp/verify-env.sh
bash /tmp/verify-env.sh
```

**Expected output** (yours will vary slightly):
```
=== Environment Verification ===

[ TOOLS ]
  ✓ fio found at /usr/bin/fio
  ✓ iostat found at /usr/bin/iostat
  ✓ mdadm found at /sbin/mdadm
  ✓ lsblk found at /bin/lsblk
  ...

[ BLOCK DEVICES ]
NAME  MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda   252:0    0  59G  0 disk
└─vda1 252:1   0  59G  0 part /

[ LOOP DEVICES ]
  No loop devices active

[ DISK SPACE IN CONTAINER ]
Filesystem      Size  Used Avail Use% Mounted on
overlay         59G   4.5G   52G  8% /

[ PRIVILEGED CHECK ]
  ✓ Container has device access (privileged)
```

---

## Step 5 — Returning to the Lab After Closing

If you close your terminal, your container keeps running. To return:

```bash
# List running containers
docker ps

# Re-attach to the container
docker exec -it px-storage-lab bash

# If container was stopped:
docker start px-storage-lab && docker exec -it px-storage-lab bash
```

**Important**: Loop devices do NOT persist across container restarts. Disk image files in `/tmp/px-lab/disks/` persist (because of the volume mount), but you must recreate `losetup` bindings after each restart.

---

## Helper Functions

Add these to your shell for convenience during labs:

```bash
cat << 'HELPERS' >> ~/.bashrc

# Create a disk image and attach as loop device
mkdisk() {
  # Usage: mkdisk <name> <size_in_MB>
  local name=$1 size=${2:-500}
  local img="/tmp/px-lab/disks/${name}.img"
  dd if=/dev/zero of=$img bs=1M count=$size status=none
  local dev=$(losetup -f --show $img)
  echo "Created: $img → $dev"
  echo $dev
}

# Remove a loop device and its image
rmdisk() {
  local dev=$1
  local img=$(losetup -l | grep $dev | awk '{print $6}')
  losetup -d $dev 2>/dev/null
  rm -f $img
  echo "Removed: $dev and $img"
}

# Show all active loop devices
disks() {
  echo "=== Loop Devices ==="
  losetup -l
  echo ""
  echo "=== Disk Images ==="
  ls -lh /tmp/px-lab/disks/ 2>/dev/null
}

HELPERS

source ~/.bashrc
echo "Helper functions loaded: mkdisk, rmdisk, disks"
```

---

## You Are Ready

Move to [01-questions.md](./01-questions.md) — 20 use-case questions.

Come back to this file if any tool is missing or if you need to rebuild the environment.
