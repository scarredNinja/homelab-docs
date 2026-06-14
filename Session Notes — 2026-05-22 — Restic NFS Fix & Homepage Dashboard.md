---
project_id: Homelab-2025
date: '2026-05-22T00:00:00.000Z'
tags:
  - session-note
  - homelab
status: Completed
phase: 'Phase 5: Docker Swarm'
---

# Session Notes — 2026-05-22 — Restic NFS Fix & Homepage Dashboard

## Session Goal

Stabilise the restic backup stack by replacing the hard NFS mount with a soft mount, and complete the Homepage dashboard deployment that was deferred from the previous session.

## Completed

- Changed `stack-restic.yml` NFS mount options from `hard,timeo=600,retrans=5` to `soft,timeo=50,retrans=3`, resolving Gotcha #49 (restic-rest-server hang/SIGKILL under Synology load) — PR #50 merged (backed by metrics in [[NFS Mount Performance Analysis]])
- Cleared stale restic lock on Proxmox host (`restic unlock`); confirmed overnight backup succeeded — no manual re-run required
- Closed PR #31 as superseded — all its cron and script changes were already present in main via PRs #38 and #50
- Merged PR #23 (`feat: add Homepage dashboard stack`) — `stack-homepage.yml` deployed, container confirmed healthy at localhost:3000
- Fixed monitoring network name bug in Homepage stack (`monitoring_monitoring` → `monitoring`) and renamed router hostname from `homepage.home.purvishome.com` to `dashboard.home.purvishome.com` (commit 03cc1be pushed directly to main)
- Conducted session triage: confirmed PRs #45, #46, #47, #48, #49 all merged to main; no outstanding merge debt

## Notes

- Portainer redeploy still pending for both `restic` stack (picks up PR #50 soft-mount) and `homepage` stack (picks up hostname rename to `dashboard.home.purvishome.com`)
- Exportarr Docker secrets (`sonarr_apikey_v2`, `radarr_apikey_v2`) still need to be created on the Swarm manager, then exporter services force-updated
- Uptime Kuma: stack deploy via Portainer, service monitors, and `widgets.yaml` kuma slug update all still pending

## Related Notes

[[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
[[01 Homelab Rebuild - Phase 7 Backup Verification Hub]]
[[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]]
