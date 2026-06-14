---
date: '2026-04-02T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
session_type: Diagnose + Fix
status: Completed
tags:
  - SessionNotes
  - DockerSwarm
  - Proxmox
  - virtiofs
  - Storage
  - Debugging
  - Provisioning
---

# Session Notes — virtiofs Fix, Permissions & Provisioning Script Bugs

> [!abstract] Overview
> Full debugging session covering virtiofs mount failures on manager-01 and
> traefik-dmz-01. Root cause was a two-layer problem: VM fstab entries missing,
> and daemon.json had hostname baked into data-root. Scripts fixed in Claude Code
> session (commit f61b9f4). ZFS dataset permissions partially fixed. Blocked on
> DHCP reservation for manager-01 before re-provisioning can complete.

---

## Bugs Found & Fixed (commit f61b9f4)

| # | File | Bug | Fix |
|---|---|---|---|
| 1 | `03-post-boot.sh` | `daemon.json` data-root had `$(hostname)` baked in — Docker internals landed in service config dir | Hard-coded `"data-root": "/mnt/docker-data"` |
| 2 | `03-post-boot.sh` | No fstab entries written for virtiofs mounts — Docker started against unmounted path | Added idempotent fstab block as first step, with `mountpoint -q` gate before continuing |
| 3 | `03-post-boot.sh` | `fuse-overlayfs` install order — daemon.json written before package confirmed installed | Moved package install before daemon.json write |
| 4 | `02-provision-vm.sh` | virtiofs devices not attached to VMs | Added `qm set $VMID --virtiofs0..3` with idempotency guard |

## Improvements Added

| # | Improvement |
|---|---|
| 1 | Idempotency guards on all steps (fstab grep, docker install check, swarm join check) |
| 2 | Step logging with timestamps via `log()` function |
| 3 | Sanity check summary at end — PASS/FAIL per check, exits non-zero on failure |
| 4 | `02-provision-vm.sh` now attaches all 4 virtiofs devices at provision time |

## Key Discovery — DataSourceNoCloud on Proxmox

> [!important]
> Proxmox itself uses NoCloud datasource natively for its cloud-init CD-ROM.
> The `99-pve.cfg` fix (ConfigDrive first) was causing issues. cicustom vendor
> works correctly with NoCloud. The NBD injection approach in `01-create-template.sh`
> needs to be reviewed — may need to revert `99-pve.cfg` change.

## ZFS Permissions Fixed
```bash
# Run on Proxmox host (no sudo — already root)
chown root:root /mnt/docker-data /mnt/docker-tsdb /mnt/docker-db /mnt/docker-swarm
chmod 755 /mnt/docker-data /mnt/docker-tsdb /mnt/docker-db /mnt/docker-swarm
rm -rf /mnt/docker-data/manager-01   # misplaced Docker internals removed
```

> [!warning] docker-db, docker-swarm, docker-tsdb still showing root:admin
> `chown` not sticking on 3 of 4 datasets — ZFS storing ownership in root inode.
> Fix with unmount/remount cycle:
> ```bash
> for ds in docker-tsdb docker-db docker-swarm; do
>   zfs unmount rpool/$ds && zfs mount rpool/$ds
>   chown 0:0 /mnt/$ds && chmod 755 /mnt/$ds
> done
> ls -la /mnt/ | grep docker
> ```

## Correct Permissions Model

| Location | Owner | Permissions | Reason |
|---|---|---|---|
| `/mnt/docker-*` (Proxmox host) | `root:root` | `755` | virtiofs passes through unchanged; Docker runs as root |
| Inside VM via virtiofs | `root:root` | `755` | Same as host — not independently settable |
| Docker socket | `root:docker` | `660` | Set by Docker install — controls CLI access |

> [!note] virtiofs is not NFS — no uid/gid remapping. Host permissions are VM permissions.

## Blocked — DHCP Reservation Not Applying

manager-01 (VMID 200) got `10.0.60.101` (dynamic) instead of `10.0.60.30` (reserved).
MAC address `52:54:fa:68:e3:8a` is configured in pfSense but VM is not using it.

**Next session task:** Diagnose why pfSense DHCP reservation is not being honoured.

---

## Current State

| VM | VMID | Status | IP | Blocker |
|---|---|---|---|---|
| manager-01 | 200 | Running, wrong IP | 10.0.60.101 | DHCP reservation not applying |
| traefik-dmz-01 | 201 | Provisioned | TBC | Waiting on manager-01 |

## Session Outcome

> [!success] Closed — 2026-04-02
> daemon.json path bug fixed, fstab step added, ZFS permissions partially fixed. DHCP blocker identified.
>
> All outstanding tasks promoted to [[Swarm Topology]] and [[Docker Swarm Infrastructure Runbook]].
> → Current status: [[Swarm Topology]]
