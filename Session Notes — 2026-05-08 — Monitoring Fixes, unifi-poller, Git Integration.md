---
date: 2026-05-08
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
session_type: Infrastructure, Monitoring, Maintenance
status: Complete
tags:
  - SessionNotes
  - DockerSwarm
  - Monitoring
  - unifi-poller
  - Portainer
  - Backup
  - Plex
---

# Session Notes — 2026-05-08 — Monitoring Fixes, unifi-poller, Git Integration

## Session Goal

Resolve all outstanding Phase 5 blockers, fix unifi-poller authentication, clean up open tasks across all phases, migrate Portainer stacks to Git integration.

---

## Completed This Session

### 1. unifi-poller Authentication Fixed

**Root cause:** Two compounding issues — `UP_UNIFI_DEFAULT_PASS_FILE` is not a supported env var in unpoller v2, and Portainer's secret UI adds a trailing newline to secret values.

**Fix:** Switched to volume-mounted TOML config at `/mnt/docker-data/unpoller/up.conf` with `pass = "..."` inline. No Docker secrets or configs needed.

**Verified:** UniFi metrics confirmed flowing into Grafana dashboards 11311 (UAP), 11315 (Clients), 11312 (USW).

See Gotcha #57 in Runbook Appendix D.

### 2. Uptime Kuma Monitors Completed

| Monitor | URL | Status |
|---------|-----|--------|
| Prometheus | `http://monitoring_prometheus:9090` | ✅ |
| Grafana | `http://monitoring_grafana:3000` | ✅ |
| Transmission | `http://gluetun:9091` (via arr_arr_default) | ✅ |
| Sonarr | `http://arr_sonarr:8989` | ✅ |
| Radarr | `http://arr_radarr:7878` | ✅ |
| Prowlarr | `http://arr_prowlarr:9696` | ✅ |
| pfSense | `https://10.0.60.1` | ✅ |
| Home Assistant | `http://10.0.60.42:8123` | ✅ |

### 3. Task Sweep — All Open Items Resolved or Triaged

- **Traefik asymmetric routing fix** — already done 2026-04-29, stale task closed
- **pfSense VLAN 50 VPN alias** — superseded by Gluetun; pfSense policy routing not required
- **UniFi Prometheus metrics** — confirmed in Grafana ✅
- **Home VLAN → Home Assistant** — pfSense rule added; `internal-only` middleware already had `10.0.10.0/24`
- **Redback HACS** — v0.3 (`juicejuice/homeassistant_redback`) working; `ActiveExportedPowerInstantaneouskW` is non-breaking log warning
- **HA analytics** — disabled via Settings → System → Analytics
- **Grafana data sources** — updated to Swarm DNS names (`monitoring_prometheus`, `monitoring_influxdb`, `monitoring_loki`)
- **pfSense rule VLAN 60 → 10.0.90.50:9100** — confirmed already in place

### 4. PR #19 — Backup Script Fixes (merged)

| File | Change |
|------|--------|
| `proxmox-config-backup.sh` | Replaced `du -sh` with `stat + numfmt` for accurate archive sizes on ZFS |
| `zfs-snapshot.sh` | Replaced `&&/||` with `if/else` in `cmd_send` to prevent duplicate log entries |
| `restic-backup.sh` | New `restic_run()` helper captures stderr via `log()` with `RESTIC:` prefix; ERR trap includes exit code |
| `restic-backup.sh` + `deploy-backup-scripts.sh` | Preflight `curl` check against REST server; distinct Discord alert if unreachable |
| `stack-controller.yml` | unpoller switched from Docker Configs/Secrets to virtiofs bind mount |

### 5. PR #24 — Traefik Metrics (open)

| File | Change |
|------|--------|
| `config/traefik/traefik.yaml` | Added `metrics` entrypoint on `:8080` and `prometheus` block |

Prometheus was already scraping `tasks.traefik_traefik:8080` — the metrics entrypoint and prometheus block were the missing half. Pending merge.

### 6. PR #22 — Plex Stack Fixes (merged)

| File | Change |
|------|--------|
| `stack-plex.yml` | Corrected transcode bind mount `/tmp/transcode` → `/mnt/transcode` |
| `stack-plex.yml` | cloudflared tunnel token passed via `--token` flag (env var ignored in Swarm mode) |

### 7. Portainer Git Integration — All Stacks Migrated

All Swarm stacks now pull directly from `scarredNinja/docker-swarm-home` (main branch) via Portainer Git integration. Manual file copy step eliminated.

| Stack | Compose path |
|-------|-------------|
| traefik | `proxmox-swarm/stacks/stack-traefik.yml` |
| monitoring | `proxmox-swarm/stacks/stack-monitoring.yml` |
| plex | `proxmox-swarm/stacks/stack-plex.yml` |
| arr | `proxmox-swarm/stacks/stack-arr.yml` |
| controller | `proxmox-swarm/stacks/controller/stack-controller.yml` |
| uptime-kuma | `proxmox-swarm/stacks/stack-uptime-kuma.yml` |
| restic | `stacks/stack-restic.yml` |

Not migrated: `portainer` (self-managed), `compose-vpn` (Docker Compose, not Swarm).

---

### 8. Homepage Dashboard — In Progress

Branch `claude/elegant-lehmann-00bda3` pushed, 4 commits:
- `e158ea3` — stack + scripts
- `48c7355` — tag fix
- `e2e0e4c` — healthcheck/resources
- `1f9a3a6` — Traefik network label fix

Container healthy on `worker-monitoring-01`. Traefik routing untested — no PR opened yet (test-first plan).

---

## Pending Verification (Overnight)

| Item | Check |
|------|-------|
| 4 backup jobs in Grafana | Check dashboard after overnight cron run |
| `zfs-snapshot.sh` duplicate log fix | Check `/var/log/zfs-snapshot.log` after 02:15 cron run — should show single entries |
| `restic-backup.sh` stderr capture | Check log for `RESTIC:` prefixed lines after current run completes |

---

## Next Session Priorities

1. Verify overnight backup logs (all 3 items above)
2. Homepage dashboard build — code session planned
3. pfSense firewall rule ordering review
4. Loki + Promtail dashboard (ID 15141) — import when ready

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]] — Gotcha #57 (unpoller auth)
- [[Swarm Topology]]
- [[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]]
