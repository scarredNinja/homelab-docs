---
date: 2026-03-23
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
session_type: Debugging
status: Completed
tags:
  - SessionNotes
  - DockerSwarm
  - Proxmox
  - CloudInit
  - Docker
  - Debugging
  - VMTemplate
---

# Session Notes — DataSourceNoCloud Deep Dive & Template Fix

> [!abstract] Overview This session was dominated by one root cause: the Ubuntu 24.04 cloud image defaults to `DataSourceNoCloud` which prevented cloud-init vendor configs from executing on cloned VMs. Multiple approaches were attempted before landing on the correct fix — writing `99-pve.cfg` directly into the source image using NBD before importing into Proxmox.

---

## The Core Problem

Every cloned VM stopped at the same point during cloud-init:

- `nfs-common` installed (from `packages:` block)
- NFS client target reached
- Then stopped — no Docker install, no runcmd execution

**Root cause:** Ubuntu 24.04 cloud images default datasource order prefers `NoCloud`. On boot, cloud-init found a NoCloud seed on `/dev/sr0` (the Proxmox cloud-init CD-ROM) and used that. Because it had a valid NoCloud seed it never looked at the Proxmox `cicustom` vendor config.

```
detail: DataSourceNoCloud [seed=/dev/sr0]
```

The `packages:` block runs regardless of datasource. `runcmd:` does not — it requires a valid datasource to be identified. This is why packages installed but nothing in runcmd ever ran.

---

## Why It's a Chicken-and-Egg Problem

The intended fix was to write `/etc/cloud/cloud.cfg.d/99-pve.cfg` during runcmd:

```yaml
runcmd:
  - cat > /etc/cloud/cloud.cfg.d/99-pve.cfg << 'EOF'
    datasource_list:
      - ConfigDrive
      - NoCloud
      - None
    EOF
```

**This doesn't work** because:

1. Cloud-init reads the datasource before running runcmd
2. runcmd only runs if a valid datasource is found
3. Without `99-pve.cfg` already present, cloud-init picks NoCloud
4. With NoCloud selected, runcmd never runs
5. So `99-pve.cfg` never gets written

The fix must be present **before** cloud-init runs for the first time — it cannot be written by cloud-init itself.

---

## Approaches Attempted (and Why They Failed)

### Attempt 1 — Write 99-pve.cfg via runcmd in vendor yaml

**Result:** Failed — runcmd never executes because datasource is wrong before runcmd runs.

### Attempt 2 — Set datasource_list at top of vendor yaml

**Result:** Failed — vendor yaml itself isn't loaded because the datasource issue prevents it from being read in the first place.

### Attempt 3 — cloud-init clean + reboot with 99-pve.cfg written manually

**Result:** Partially worked on existing VMs but doesn't fix the template for future clones.

### Attempt 4 — kpartx to mount template disk and write file

**Result:** Corrupted the filesystem — EXT4 inode bitmap errors, disk remounted read-only, VM unbootable.

```
EXT4-fs error (device sda1): ext4_validate_inode_bitmap
Corrupt inode bitmap — block_group = 5
Remounting filesystem read-only
```

> [!danger] Do not use kpartx on ZFS-backed Proxmox VM disks kpartx corrupted the EXT4 filesystem on the ZFS zvol. The VM became unbootable. Always use NBD for this type of operation.

### Attempt 5 — guestfs-tools / libguestfs-tools

**Result:** Package not available on this Proxmox version.

```
E: Unable to locate package libguestfs-tools
E: Unable to locate package guestfs-tools
```

### Attempt 6 — Mount ZFS zvol directly

**Result:** Failed — `/dev/zvol/local-zfs/vm-9000-disk-0` did not exist. ZFS pool was actually `rpool/data` not `local-zfs`.

### Attempt 7 — NBD (Network Block Device) on source image ✅

**Result:** Success — clean mount, file written, no corruption.

---

## The Correct Fix — NBD on Source Image

Mount the original cloud image using NBD **before** importing into Proxmox. Write `99-pve.cfg` directly into the image. Then import and create the template. Every VM cloned from the template has the fix present from the very first boot.

```bash
# Load NBD kernel module (built into Proxmox/QEMU, no install needed)
modprobe nbd max_part=8

# Connect the cloud image
qemu-nbd --connect=/dev/nbd0 /root/noble-server-cloudimg-amd64.img

# Check partition layout
fdisk -l /dev/nbd0

# Mount root partition
mkdir -p /mnt/template-disk
mount /dev/nbd0p1 /mnt/template-disk

# Write the datasource fix
mkdir -p /mnt/template-disk/etc/cloud/cloud.cfg.d/
cat > /mnt/template-disk/etc/cloud/cloud.cfg.d/99-pve.cfg << 'EOF'
datasource_list:
  - ConfigDrive
  - NoCloud
  - None
EOF

# Verify
cat /mnt/template-disk/etc/cloud/cloud.cfg.d/99-pve.cfg

# Clean unmount
umount /mnt/template-disk
qemu-nbd --disconnect /dev/nbd0
```

Then run the template creation script as normal.

> [!tip] NBD is always available on Proxmox `qemu-nbd` is part of the QEMU package already installed on every Proxmox host. No additional packages needed. Always prefer NBD over kpartx for mounting QEMU disk images.

---

## ZFS Pool Path — Lesson Learned

The Proxmox web UI shows storage as `local-zfs` but the actual ZFS dataset path is different.

```bash
# Find actual ZFS paths
zfs list | grep vm-<vmid>
# Shows: rpool/data/vm-9000-disk-0   not   local-zfs/vm-9000-disk-0
```

|Web UI name|Actual ZFS path|
|---|---|
|`local-zfs`|`rpool/data`|
|Device path|`/dev/zvol/rpool/data/vm-<id>-disk-0`|

---

## datasource_list — Where It Must Live

|Location|Works?|Why|
|---|---|---|
|Vendor yaml `runcmd` (write file)|❌|runcmd doesn't run without valid datasource|
|Top of vendor yaml as `datasource_list:`|❌|Vendor yaml not loaded without valid datasource|
|`/etc/cloud/cloud.cfg.d/99-pve.cfg` on disk|✅|Read before any cloud-init phase runs|

The file must be physically present in the image on disk before first boot. There is no cloud-init-based way to write it — it must be injected into the image externally.

---

## Updated Template Creation Process

The correct sequence for building a clean Proxmox swarm template from an Ubuntu 24.04 cloud image:

```bash
# 1. Download cloud image
# scp noble-server-cloudimg-amd64.img root@<proxmox-ip>:/root/

# 2. Inject 99-pve.cfg via NBD BEFORE importing
modprobe nbd max_part=8
qemu-nbd --connect=/dev/nbd0 /root/noble-server-cloudimg-amd64.img
mount /dev/nbd0p1 /mnt/template-disk
mkdir -p /mnt/template-disk/etc/cloud/cloud.cfg.d/
cat > /mnt/template-disk/etc/cloud/cloud.cfg.d/99-pve.cfg << 'EOF'
datasource_list:
  - ConfigDrive
  - NoCloud
  - None
EOF
umount /mnt/template-disk
qemu-nbd --disconnect /dev/nbd0

# 3. Run template creation script
bash /root/create-swarm-template.sh
```

> [!note] This step should be added to create-swarm-template.sh The NBD injection should be automated as a step in the template script so it's never forgotten when rebuilding from a fresh image.

---

## Full Issues List This Session

|Issue|Cause|Fix|
|---|---|---|
|cloud-init stops after nfs-common|DataSourceNoCloud — runcmd never runs|Write 99-pve.cfg into image via NBD before import|
|kpartx corrupted VM disk|kpartx not safe on ZFS zvols|Use NBD instead|
|guestfs-tools not available|Not in Proxmox Debian repos|Use NBD — already available via QEMU|
|ZFS mount path wrong|Web UI name ≠ actual ZFS path|Use `zfs list` to find real path|
|datasource_list in vendor yaml ignored|Vendor yaml not loaded without valid datasource|Must be on disk before first boot|
|Template VM unbootable after kpartx|EXT4 filesystem corruption|Rebuild from source image using NBD|

---

## Key Lessons

**1 — DataSourceNoCloud is the Ubuntu 24.04 default** The Ubuntu cloud image ships with NoCloud as the preferred datasource. On Proxmox, this means cloud-init reads the CD-ROM seed and ignores cicustom vendor configs. The fix (`99-pve.cfg` with `datasource_list`) must be physically in the image before first boot — it cannot be applied by cloud-init itself.

**2 — NBD is the right tool for modifying cloud images on Proxmox** `qemu-nbd` is already installed on every Proxmox host. It cleanly mounts qcow2 images without risk of filesystem corruption. kpartx on ZFS zvols is unsafe and should be avoided.

**3 — packages: runs, runcmd: doesn't — without a valid datasource** This is the tell for the DataSourceNoCloud problem. If you see packages installing but nothing in runcmd executing, check `cloud-init status --long` for the datasource detail line immediately.

**4 — Proxmox web UI storage names don't match ZFS paths** `local-zfs` in the UI = `rpool/data` in ZFS. Always verify with `zfs list` before using device paths.

**5 — The fix must be baked in before import, not after** Any approach that tries to write `99-pve.cfg` after the template is created (via cloud-init, via mounting the converted zvol) is fighting against the problem rather than solving it. Inject into the source `.img` file before the `qm importdisk` step.

---

## Session Outcome

> [!success] Closed — 2026-03-23
> NBD fix identified and documented. Lessons promoted to runbook Step 0.0.
>
> All outstanding tasks promoted to [[Swarm Topology]] and [[Docker Swarm Infrastructure Runbook]].
> → Current status: [[Swarm Topology]]


## Related

- [[Session Notes — 2026-03-19 — VM Template Debugging]]
- [[Docker Swarm Infrastructure Runbook]]
- [[Docker Swarm — Expansion & Storage Planning Notes]]