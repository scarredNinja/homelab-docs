---
project_id: Homelab-2025
date: '2026-05-25T00:00:00.000Z'
tags:
  - session-note
  - homelab
status: Completed
phase: 'Phase 5: Docker Swarm'
---

# Session Notes — 2026-05-25 — Swarm Manager HA & Backup Automation

## Summary

Provisioned two new Swarm managers to achieve 3-manager quorum, fixing the single-manager SPOF. Fixed a latent arithmetic bug in the provisioning script. Refactored `vm-backup.sh` to discover VMs from the registry dynamically so future provisions are automatically included in nightly backups.

## Completed

- **Provisioned manager-02** (VMID 210, IP 10.0.60.31, VLAN 60) — joined swarm as manager via `02-provision-vm.sh` + `03-post-boot.sh`
- **Provisioned manager-03** (VMID 211, IP 10.0.60.32, VLAN 60) — joined swarm as manager
- **Swarm now has 3-manager quorum** — manager-01 (Leader), manager-02 (Reachable), manager-03 (Reachable); survives one manager failure
- **Fixed `(( ++ ))` arithmetic bug** in `02-provision-vm.sh` — `(( candidate++ ))` and `(( idx++ ))` replaced with `var=$(( var + 1 ))` safe form; [PR #57](https://github.com/scarredNinja/docker-swarm-home/pull/57)
- **Updated `vm-backup.sh`** to add manager-02/03 VMIDs, then refactored to full registry-driven VM discovery — reads `/root/vm-registry/*.json`, orders non-managers → additional managers → first manager (leader) last; [PR #58](https://github.com/scarredNinja/docker-swarm-home/pull/58)
- **Marked Easy Win #5 complete** in Homelab Next Phase — Review & Roadmap (the `(( idx++ ))` fix)

## Open / Follow-up

- [x] Deploy updated `vm-backup.sh` to Proxmox host — done 2026-05-27 via `deploy-backup-scripts.sh` ✅
- [ ] Verify overnight backup run includes all 9 VMs — check `/var/log/vm-backup.log` 2026-05-28 morning (note: all 9 registry files had missing `first_manager` field, fixed 2026-05-27 before deploy)
- [ ] Verify `docker node ls` shows 3 managers: Leader + 2 Reachable
- [ ] Roadmap: move one manager to a second physical host when hardware arrives for true physical HA

## Notes

- `zfs-snapshot.sh` required no changes — it snapshots shared virtiofs datasets used by all VMs
- `restic-backup.sh` required no changes — backs up data volumes, not per-VM state
- Backup window is now 8 VMs in 2h (03:00–05:00); watch logs to confirm restic isn't deferred
- 3 managers on one Proxmox host = Swarm HA (quorum), NOT physical HA — host failure still takes all three down

## Related Notes

- [[Docker Swarm Infrastructure Runbook]]
- [[Homelab Next Phase — Review & Roadmap]]
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
- [[01 Homelab Rebuild - Phase 7 Backup Verification Hub]]
