# Module 01 Lab — Environment Setup

```
Environment : Ubuntu container on macOS (Podman or Docker Desktop)
Module      : 01 — Storage From First Principles
All labs run inside the container — not on your Mac
```

---

## Is Any of This Dangerous to My MacBook?

**No. Here is exactly why.**

---

### Containers on Mac Always Run Inside a VM

On a Linux server, containers share the host kernel directly:

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

**Your Mac is different.** macOS runs the Darwin (XNU) kernel — not Linux. Neither Podman nor Docker Desktop can share your Mac kernel with a Linux container because they are completely different kernels.

Both Podman and Docker Desktop solve this the same way — they install a **hidden Linux VM** first, then run containers inside that VM:

```
┌──────────────────────────────────────────────────────────┐
│  Your MacBook (macOS / Darwin XNU kernel)                │
│  ← never touched by any container command                │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │  Podman Machine VM  (or Docker Desktop VM)       │    │
│  │  Lightweight Linux — Apple Virtualization        │    │
│  │                                                  │    │
│  │  ┌──────────────┐  ┌──────────────┐             │    │
│  │  │ Ubuntu       │  │ Any other    │             │    │
│  │  │ Container    │  │ Container    │             │    │
│  │  └──────┬───────┘  └──────┬───────┘             │    │
│  │         │                 │                     │    │
│  │  ───────┴─────────────────┴──────────────────   │    │
│  │      Linux Kernel (belongs to the VM only)      │    │
│  └──────────────────────────────────────────────┘  │    │
│                                                          │
│  Apple Silicon / Intel hardware                          │
└──────────────────────────────────────────────────────────┘
```

**Podman Machine = same VM safety as Docker Desktop.** The blast radius of every lab command stops at the VM boundary.

---

### What This Means for Every Command in This Lab

| Command / Operation | Affects Mac? | Why |
|--------------------|-------------|-----|
| `--privileged` flag | No | Root access to the VM's Linux kernel, not macOS |
| Loop devices (`/dev/loop*`) | No | Created inside the VM kernel, invisible to macOS |
| `mdadm` RAID arrays | No | Software RAID lives inside the VM |
| `mkfs`, `fdisk`, `parted` | No | Operate on loop device image files inside the VM |
| `/proc/sys/vm/drop_caches` | No | The VM's `/proc`, not macOS |
| `fio`, `dd` writes | No | Write to files inside the VM filesystem |
| Crashing the VM | No | Podman/Docker restarts the VM automatically |

**Worst case**: you crash the VM. Podman shows a restart prompt. Your Mac is unaffected.

---

### The ONE Thing That Touches Your Mac

The volume mount flag:

```bash
-v ~/px-lab:/tmp/px-lab
```

This is the **only bridge** between macOS and the container:

```
macOS filesystem            VM filesystem        Container
────────────────            ─────────────        ──────────
~/px-lab/       ←──────────────────────────────→ /tmp/px-lab/
(on your Mac SSD)  shared via Apple Hypervisor   (inside container)
```

Disk image files you create inside `/tmp/px-lab/` are stored as plain files in `~/px-lab/` on your Mac. They consume disk space but cannot harm macOS.

**Total disk space across all labs**: 1–3 GB maximum.

**To clean up after finishing all labs**:
```bash
# On your Mac terminal (NOT inside the container)
rm -rf ~/px-lab
```

---

## What You Are Building

```
Your MacBook (macOS — untouched by lab commands)
└── Podman Machine VM (Linux VM, Apple Virtualization Framework)
    └── Ubuntu 22.04 Container   ← you work here
        ├── /dev/loop*           ← virtual disks you create (inside VM)
        ├── /tmp/px-lab/disks/   ← image files (shared back to Mac at ~/px-lab/)
        ├── fio                  ← I/O benchmarking tool
        ├── sysstat (iostat)     ← real-time I/O monitoring
        ├── mdadm                ← software RAID management
        └── util-linux           ← lsblk, blkid, losetup, fdisk
```

You will create virtual disks using loop devices. These behave identically to real block devices — you can partition them, format them, create RAID arrays on them, and observe all the same kernel behavior as real hardware.

---

## Choose Your Runtime

### Option A — Podman (Recommended — Free, No License)

**Install Podman**:

```bash
# Install via Homebrew
brew install podman

# Verify installation
podman --version
```

**Initialize and start the Podman VM** (one-time setup):

```bash
# Create the VM with enough resources for the labs
podman machine init \
  --cpus 2 \
  --memory 2048 \
  --disk-size 20 \
  --volume ~/px-lab:/tmp/px-lab

# Start the VM
podman machine start

# Verify the VM is running
podman machine list
```

Expected output:
```
NAME                     VM TYPE     CREATED     LAST UP     CPUS  MEMORY  DISK SIZE
podman-machine-default*  applehv     1 min ago   1 min ago   2     2GiB    20GiB
```

The `*` means it is the active machine.

---

### Option B — Docker Desktop (If Already Installed)

If Docker Desktop is available and allowed, it works identically. Skip the Podman setup and use `docker` wherever this guide says `podman`.

| Podman command | Docker equivalent |
|---------------|------------------|
| `podman run` | `docker run` |
| `podman exec` | `docker exec` |
| `podman ps` | `docker ps` |
| `podman start` | `docker start` |
| `podman machine start` | (Docker Desktop auto-starts) |

---

## Step 1 — Create the Lab Directory and Start the Container

**On your Mac terminal**, create the shared directory first:

```bash
mkdir -p ~/px-lab/disks
mkdir -p ~/px-lab/mounts
echo "Lab directory ready:"
ls -la ~/px-lab/
```

**Start the Ubuntu container**:

```bash
# Using Podman:
podman run -it \
  --name px-storage-lab \
  --privileged \
  --cap-add=SYS_ADMIN \
  --cap-add=MKNOD \
  -v ~/px-lab:/tmp/px-lab \
  ubuntu:22.04 \
  bash

# Using Docker Desktop (identical flags):
docker run -it \
  --name px-storage-lab \
  --privileged \
  --cap-add=SYS_ADMIN \
  --cap-add=MKNOD \
  -v ~/px-lab:/tmp/px-lab \
  ubuntu:22.04 \
  bash
```

**Why `--privileged`?**
Loop devices, RAID (mdadm), and filesystem operations require elevated kernel privileges. `--privileged` gives the container root-level access to the **VM's** Linux kernel — not macOS. Safe on Mac because of the VM layer.

**Why `-v ~/px-lab:/tmp/px-lab`?**
Persists disk image files on your Mac so they survive container restarts. This is the only path shared between macOS and the container.

**Why `~/px-lab` instead of `/tmp/px-lab`?**
Podman Machine automatically shares your home directory (`~`). `/tmp` on macOS is a symlink to `/private/tmp` and may not be shared depending on Podman version. Using `~/px-lab` is reliable across all versions.

You should now see a prompt like:

```
root@abc123def456:/#
```

---

## Step 2 — Install Required Tools

Run this entire block **inside the container**:

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

## Step 3 — Verify Your Environment

Run this verification script **inside the container**:

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

echo "[ SHARED VOLUME MOUNT ]"
if ls /tmp/px-lab/ &>/dev/null; then
  echo "  ✓ /tmp/px-lab/ is mounted (shared with Mac ~/px-lab/)"
  ls /tmp/px-lab/
else
  echo "  ✗ /tmp/px-lab/ NOT mounted — check -v flag in run command"
fi

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

**Expected output**:

```
=== Environment Verification ===

[ TOOLS ]
  ✓ fio found at /usr/bin/fio
  ✓ iostat found at /usr/bin/iostat
  ✓ mdadm found at /sbin/mdadm
  ✓ lsblk found at /bin/lsblk
  ...

[ BLOCK DEVICES ]
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda    252:0    0   20G  0 disk
└─vda1 252:1    0   20G  0 part /

[ LOOP DEVICES ]
  No loop devices active

[ SHARED VOLUME MOUNT ]
  ✓ /tmp/px-lab/ is mounted (shared with Mac ~/px-lab/)
  disks  mounts

[ DISK SPACE IN CONTAINER ]
Filesystem      Size  Used Avail Use% Mounted on
overlay          20G  1.5G   18G   8% /

[ PRIVILEGED CHECK ]
  ✓ Container has device access (privileged)
```

---

## Step 4 — Returning to the Lab After Closing

If you close your terminal, the container keeps running. To return:

```bash
# Using Podman:
podman ps                                             # list running containers
podman exec -it px-storage-lab bash                   # re-attach

# If container was stopped:
podman start px-storage-lab && podman exec -it px-storage-lab bash

# Using Docker Desktop:
docker ps
docker exec -it px-storage-lab bash
docker start px-storage-lab && docker exec -it px-storage-lab bash
```

**If Podman Machine was stopped** (e.g., after Mac reboot):

```bash
podman machine start           # restart the VM first
podman start px-storage-lab    # then restart the container
podman exec -it px-storage-lab bash
```

**Important**: Loop devices do NOT persist across container restarts. Disk image files in `~/px-lab/disks/` persist (because of the volume mount), but you must recreate `losetup` bindings after each restart.

---

## Step 5 — Helper Functions

Add these to your shell **inside the container** for convenience during labs:

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

# Show all active loop devices and disk images
disks() {
  echo "=== Loop Devices ==="
  losetup -l
  echo ""
  echo "=== Disk Images on Mac (~/px-lab/disks/) ==="
  ls -lh /tmp/px-lab/disks/ 2>/dev/null || echo "  (empty)"
}

HELPERS

source ~/.bashrc
echo "Helper functions loaded: mkdisk, rmdisk, disks"
```

---

## Troubleshooting Podman

**Problem: `podman machine start` fails**

```bash
# Stop and recreate the machine
podman machine stop
podman machine rm
podman machine init --cpus 2 --memory 2048 --disk-size 20 --volume ~/px-lab:/tmp/px-lab
podman machine start
```

**Problem: `/tmp/px-lab` not found inside container**

```bash
# On Mac: verify the directory exists
ls ~/px-lab

# Check if Podman machine was initialized with the volume
podman machine inspect | grep -A 5 Mounts

# If volume not set, recreate machine with --volume flag (see above)
```

**Problem: `--privileged` operations fail (loop devices, mdadm)**

```bash
# Verify the container is truly privileged
podman inspect px-storage-lab | grep Privileged
# Should show: "Privileged": true

# If false: remove and recreate container with --privileged flag
podman rm -f px-storage-lab
# Then re-run the podman run command from Step 1
```

**Problem: `podman` command not found after install**

```bash
# Reload your shell PATH
source ~/.zshrc   # or ~/.bashrc depending on your shell

# Or use full path
/opt/homebrew/bin/podman --version
```

---

## You Are Ready

Move to [01-questions.md](./01-questions.md) — 20 use-case questions.

Come back to this file if any tool is missing or if Podman needs troubleshooting.
