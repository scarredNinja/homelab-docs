---
date: 2026-03-27
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
session_type: Code
status: Complete
tags:
  - SessionNotes
  - DockerSwarm
  - Provisioning
  - Scripting
  - virtiofs
---

# Session Notes — Provisioning Scripts v2 Complete

## Session Summary
Completed fixes to 02-provision-vm.sh and 03-post-boot.sh following virtiofs
attach failure identified in previous session.

---

## 02-provision-vm.sh — Changes

| # | Change | Detail |
|---|---|---|
| 1 | `ensure_dir_mappings()` | Auto-creates missing Proxmox directory mappings via `pvesh create /cluster/mapping/dir` |
| 2 | virtiofs fix | `attach_virtiofs()` now passes mapping ID only: `qm set $VMID --virtiofs0 "docker-data"` |
| 3 | Multi-NIC | `--nic1-vlan` / `--vlan` (primary), `--nic2-vlan`, `--nic3-vlan`; separate MAC per NIC |
| 4 | NIC2 MAC bug | `configure_network_2()` was using MAC_NIC0 — now uses MAC_NIC1 |
| 5 | NIC conditionals | configure_network_2/3 return early if VLAN unset |
| 6 | Restricted VLAN guard | Warns + prompts if `--nic1-vlan` is 80 or 20 |
| 7 | Registry | Adds nic2_vlan, nic3_vlan to JSON |
| 8 | `print_summary()` | NICs + MACs, storage slots, virtiofs table, ZFS snapshot cron reminder |
| 9 | `prompt_dhcp_confirm()` | Interactive DHCP → start → 03-post-boot.sh handoff |
| 10 | VMID bug | `qm start $VMID` → `qm start $NEW_VMID` |
| 11 | `--no-autostart` | Removed — start now controlled interactively |

---

## 03-post-boot.sh — Changes

| # | Change | Detail |
|---|---|---|
| 1 | `configure_zvol_mounts()` | Formats ZFS zvols idempotently via blkid, UUID-based fstab, mounts at `/mnt/local/<name>` |
| 2 | `verify_datasource()` | Changed warn → die on non-ConfigDrive datasource |
| 3 | `verify_provisioning()` | Writable touch/rm test on all virtiofs + zvol mounts; cloud-init status check |
| 4 | `main()` | Calls `configure_zvol_mounts` between `configure_virtiofs` and `install_docker` |

---

## Architecture Decisions Made

### Plex Transcode Storage
- virtiofs (FUSE) has overhead unsuitable for 4K transcode scratch writes
- ZFS zvol as raw block device → ext4 → `/mnt/local/transcode`
- Deploy: `./02-provision-vm.sh --name plex-01 --volume transcode:50G`
- PBS does NOT capture zvols — must add to ZFS snapshot cron (printed in summary)

### Multi-NIC Design
- `--nic1-vlan` = primary management VLAN (60) — cloud-init, apt, swarm traffic
- `--nic2-vlan` = secondary (80 DMZ for Traefik, 20 IoT for HA)
- `--nic3-vlan` = storage (100) for VMs needing direct NAS NFS access
- Guard: warns if nic1 VLAN is 80 or 20

---

## Session Outcome

> [!success] Closed — 2026-03-27
> 02-provision-vm.sh and 03-post-boot.sh v2 complete. Full provision cycle ready for end-to-end test.
>
> All outstanding tasks promoted to [[Swarm Topology]] and [[Docker Swarm Infrastructure Runbook]].
> → Current status: [[Swarm Topology]]


## ZFS Snapshot Cron — Template
Add to `/etc/cron.d/zfs-snapshots` on Proxmox host after each new zvol:
```bash
0 2 * * * root \
  SNAP="daily-$(date +%Y%m%d)"; \
  zfs snapshot rpool/data/plex-01_transcode@$SNAP; \
  zfs list -t snapshot -o name | grep "plex-01_transcode@daily-" \
    | head -n -14 | xargs -r zfs destroy
```

---
