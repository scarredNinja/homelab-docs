---
date: 2026-03-27
tags: [Homelab, Phase5, DockerSwarm, Provisioning, Scripting, virtiofs, Debugging]
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
branch: claude/suspicious-lamport
---

# Session Notes — Provisioning Scripts Live Debug & Fix

> [!abstract] Overview
> Live testing of the three provisioning scripts against a real VM uncovered several
> non-obvious bugs. All fixed in the branch. VM `traefik-dmz-01` is running but
> blocked on a Proxmox routing issue before `03-post-boot.sh` can SSH in.

---

## What Was Being Fixed

Three scripts for spinning up Docker Swarm VMs on Proxmox:
- `01-create-template.sh` — unchanged this session
- `02-provision-vm.sh` — major rework
- `03-post-boot.sh` — targeted additions

---

## 02-provision-vm.sh — Changes

| Fix | Detail |
|---|---|
| **virtiofs syntax** | Was passing `dirid=/mnt/docker-data,tag=docker-data` — Proxmox only accepts a mapping ID. Fixed to `qm set $VMID --virtiofs0 "docker-data"` |
| **Directory mappings** | Added `ensure_dir_mappings()` — auto-creates missing `/cluster/mapping/dir` entries via `pvesh` before attach |
| **Machine type guard** | virtiofs requires `q35`. Added auto-detection and upgrade from `i440fx` in `attach_virtiofs()` |
| **qm error surfacing** | virtiofs errors were silently swallowed. Now captures `2>&1` and includes qm output in die message |
| **`(( idx++ ))` bug** | With `set -e`, `(( 0++ ))` = exit code 1, killing the script. virtiofs0 was attaching fine but the counter crashed it. Fixed to `idx=$(( idx + 1 ))` |
| **jq `.result[]` bug** | `qm agent` outputs a plain array `[...]` not `{"result":[...]}`. IP never extracted, `wait_for_ip()` always timed out. Fixed with `if type == "array" then .[] else .result[] end` |
| **Guest agent channel** | Added `--agent enabled=1` to VM config — without this the virtio-serial channel doesn't exist and `qm agent` never responds |
| **SSH key CRLF** | Windows-generated `homelab_ed25519.pub` had `\r\n` endings. Added `sed -i 's/\r//'` at provision time |
| **NIC2 MAC bug** | `configure_network_2()` was using `MAC_NIC0` — fixed to `MAC_NIC1` |
| **Multi-NIC support** | Added `--nic1-vlan` alias, `--nic3-vlan`, `MAC_NIC2`, `configure_network_3()` |
| **Restricted VLAN guard** | Warns + prompts if `--nic1-vlan` is 80 (DMZ) or 20 (IoT) |
| **Interactive DHCP flow** | Replaced auto-start with `print_summary()` + `prompt_dhcp_confirm()` — shows NIC MACs, storage, virtiofs table, ZFS cron reminder, prompts pfSense confirmation before starting |
| **Registry fields** | Added `nic2_vlan`, `nic3_vlan` to saved JSON |
| **`scsi_idx++` bug** | Same `set -e` arithmetic issue, fixed consistently |

---

## 03-post-boot.sh — Changes

| Fix | Detail |
|---|---|
| **`configure_zvol_mounts()`** | New function: reads `extra_volumes` from registry, idempotent `ext4` format via `blkid`, UUID-based fstab, mounts at `/mnt/local/<name>`, sets ownership to `CI_USER` |
| **Datasource hard fail** | `verify_datasource()` changed from warn → `die` on non-ConfigDrive |
| **Writable mount tests** | `verify_provisioning()` now touch/rm tests all virtiofs and zvol mounts |
| **cloud-init status check** | Added `cloud-init status` verification to final check |
| **`(( idx++ ))` bug** | Same fix as `02` — `idx=$(( idx + 1 ))` |

---

## Bugs Found During Live Testing

> [!warning] These were only caught by running against a real VM — not visible in code review

1. **`(( idx++ ))` with `set -e`** — virtiofs attached successfully but script died on the counter increment. Looked like a virtiofs failure, was actually arithmetic.
2. **`base-vendor.yaml` had `0x97` bytes** — Windows em dash characters at positions 33 and 266. cloud-init YAML parser rejected the file silently, vendor config skipped entirely, `qemu-guest-agent` never installed.
3. **`qm agent` jq mismatch** — output is a plain array `[...]` not `{"result":[...]}`. IP was reported in Proxmox but `03` never parsed it, so `wait_for_ip()` always timed out.
4. **`--agent enabled=1` missing** — guest agent service was running inside the VM but there was no virtio-serial channel for Proxmox to talk through.

---

## Current State

| Item | Status |
|---|---|
| `traefik-dmz-01` (VMID 200) | ✅ Running at `10.0.60.102` |
| Scripts in branch | ✅ All fixes applied |
| `03-post-boot.sh` execution | ❌ Blocked — Proxmox has no route to `10.0.60.0/24` |

---

## Blocking Issue — Proxmox Routing

Proxmox host cannot reach `10.0.60.0/24` so `03-post-boot.sh` cannot SSH into the VM.

**Three options to fix:**
```bash
# Option 1 — Temporary route (lost on reboot)
ip route add 10.0.60.0/24 via <pfsense-mgmt-ip>

# Option 2 — Add VLAN 60 interface to Proxmox (vmbr0.60)
# Proxmox UI → System → Network → Add → Linux VLAN
# VLAN raw device: vmbr0, VLAN tag: 60, IP: 10.0.60.50/25

# Option 3 — pfSense inter-VLAN rule
# Allow Proxmox host IP → 10.0.60.0/25 on VLAN 60
```

> [!tip] Option 2 is the right long-term fix. Proxmox should have a VLAN 60 interface so it can always reach its own VMs for management without routing through pfSense.

---

## Session Outcome

> [!success] Closed — 2026-03-27
> All script bugs fixed and merged. VLAN 60 routing resolved. Branch merged.
>
> All outstanding tasks promoted to [[Swarm Topology]] and [[Docker Swarm Infrastructure Runbook]].
> → Current status: [[Swarm Topology]]


## Key Lessons

> [!note] `(( expr ))` with `set -e` is a trap
> Any arithmetic expression that evaluates to 0 returns exit code 1 under `set -e`.
> `(( idx++ ))` when `idx=0` kills the script. Always use `idx=$(( idx + 1 ))`.

> [!note] Windows line endings break SSH keys and YAML
> Both `homelab_ed25519.pub` (CRLF) and `base-vendor.yaml` (em dashes from Word-style editors)
> caused silent failures. Validate encoding on any file generated on Windows before use on Linux.

> [!note] `qm agent` requires `--agent enabled=1` on the VM config
> The guest agent service running inside the VM is not enough. Proxmox needs the
> virtio-serial channel enabled at the VM config level to communicate with it.

---

## Related Notes
- [[Session Notes — DataSourceNoCloud Deep Dive & Template Fix]]
- [[Session Notes - VM Template Debugging & Fixes]]
- [[Docker Swarm Infrastructure Runbook]]
- [[Docker Swarm — Expansion & Storage Planning Notes]]