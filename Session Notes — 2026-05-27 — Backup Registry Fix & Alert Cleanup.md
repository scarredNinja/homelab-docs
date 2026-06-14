---
project_id: Homelab-2025
date: '2026-05-27T00:00:00.000Z'
session_type: Bugfix + Refactor
status: Completed
tags:
  - homelab
  - backup
  - session-note
  - docker-swarm
phase: 'Phase 5: Docker Swarm'
---

# Session Notes — 2026-05-27 — Backup Registry Fix & Alert Cleanup

## Summary

Diagnosed why only 6 of 9 registered VMs were being backed up. Root cause was two issues: the deployed `vm-backup.sh` was still the old hardcoded pre-PR #58 version, and all 9 registry files were missing the `first_manager` field causing managers to be silently skipped. Fixed both and deployed the updated script. Also removed success-only Discord pings from all backup scripts — alerts now fire on failure only.

## Completed

- **Homepage kuma widget fix** (`54db0ab`) — switched Uptime Kuma widget URL from Traefik hostname to internal Docker service URL (`http://uptime-kuma:3001`) to bypass `internal-only@file` middleware blocking API access. Resolves "Missingkuma" widget error.

- **VM backup registry fix** (hands-on Proxmox) — investigated why only 6/9 VMs were being backed up nightly despite 9 entries in `/root/vm-registry/`:
  - Root cause 1: `vm-backup.sh` deployed on Proxmox host was the old hardcoded version (pre-PR #58); `deploy-backup-scripts.sh` was never re-run after PR #58 merged
  - Root cause 2: all 9 registry JSON files were missing the `first_manager` field — `build_vm_list` uses `select(.role == "manager" and .first_manager == true/false)`, which silently skips managers when the field is `null`
  - Fix: added `first_manager: true` to `manager-01.json`, `first_manager: false` to `manager-02.json` and `manager-03.json`
  - Ran `deploy-backup-scripts.sh` on Proxmox host to install registry-driven `vm-backup.sh` from PR #58

- **PR #62 — Remove success Discord pings** (`7ff7a4e`) — removed `notify_success` function and calls from `vm-backup.sh`, `restic-backup.sh`, and `proxmox-config-backup.sh`. Discord alerts now fire on failure only. Grafana + log files are sufficient to confirm healthy runs.

## Open / Follow-up

- [ ] Verify 2026-05-28 03:00 backup log shows `VM backup order (9 VMs)` line and all 9 VMs succeed
- [ ] Run `deploy-backup-scripts.sh` again after PR #62 merges to push failure-only alerting to Proxmox host
- [ ] Watch backup timing — 9 VMs vs previous 6 in the 03:00–05:00 window; confirm restic isn't deferred

## Notes

- `first_manager` field was missing on all 9 registry files — this is a gap in how the VMs were originally provisioned. Current `02-provision-vm.sh` writes the field correctly for new VMs going forward.
- Workers don't need `first_manager` in the registry (the `select(.role != "manager")` filter ignores it), but managers need it explicitly set to be included in backups.

## Related Notes

- [[01 Homelab Rebuild - Phase 7 Backup Verification Hub]]
- [[Session Notes — 2026-05-25 — Swarm Manager HA & Backup Automation]]
- [[Docker Swarm Infrastructure Runbook]]
