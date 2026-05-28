---
project_id: Homelab-2025
date: 2026-05-20
tags:
  - homelab
  - session-note
  - monitoring
  - alertmanager
status: Complete
---
# Session Notes — 2026-05-20 — Alertmanager + Discord Alerting

## Summary

Deployed Alertmanager with Discord webhook notifications. Completed post-deploy work from PRs #39–41. Added Phase 2 inter-VLAN SSH automation to deploy scripts.

## Completed

- Alertmanager added to monitoring stack (`prom/alertmanager:v0.27.0`), pinned to `zone=monitoring`, UI at `alertmanager.home.purvishome.com`
- `alerting` block added to `prometheus.yml` pointing to `tasks.monitoring_alertmanager:9093`
- Alert rules file created at `config/monitoring/alert-rules.yml` — 7 rules deployed (node down, CPU high, memory high, disk warning/critical, ZFS degraded, service restart loop)
- Discord webhook configured in Alertmanager — URL stored at `/etc/alertmanager/discord-webhook` on Proxmox host (chmod 600), injected by `copy-swarm-config.sh`
- Test alert fired and received in Discord
- PR #48 (feat(monitoring): Alertmanager + Discord webhook alerting) merged to main
- pfSense firewall rule added: VLAN60 (10.0.60.0/25) → VLAN50 (10.0.50.0/24) TCP/22 for Phase 2 inter-VLAN SSH

## Working Sessions

| Start | End | Focus |
|---|---|---|
| 2026-05-20 | 2026-05-20 | Alertmanager + Discord alerting, Phase 2 deploy automation |

## Notes

- Discord webhook URL stored at `/etc/alertmanager/discord-webhook` on Proxmox host (chmod 600, not in git)
- `webhook_url_file` not supported for `discord_configs` in Alertmanager v0.27.0 — use `webhook_url` with `__DISCORD_WEBHOOK_URL__` placeholder, injected by `copy-swarm-config.sh`
- Alertmanager data dir `/mnt/docker-tsdb/alertmanager` must be owned by `nobody` (uid 65534)
- Phase 2: pfSense VLAN60→VLAN50 TCP/22 rule added; `deploy-worker-scripts.sh --remote <host>` ready to test next session
- PR #48 merged to main

## Open / Next Session

- Investigate high CPU usage across VMs [priority::1] — manager-01 at 102.7% (over capacity), traefik-dmz-01 at 100%, worker-controller-01 at 100%, pve host at 87.9%. Observed in Proxmox summary view during nfs-bench run on worker-media-01. Check top/htop on each VM, identify runaway processes, determine if load is from nfs-bench or a sustained issue.
- Remove and redeploy monitoring stack from repo [priority::2] — stack-monitoring.yml has been updated (Alertmanager, new prometheus targets, monitoring overlay network changes). Steps: `docker stack rm monitoring`, then `docker stack deploy -c stacks/stack-monitoring.yml monitoring`. Also re-import `backup-status.json` v5 dashboard after redeploy.
- Add DNS A record: `alertmanager.home.purvishome.com` → Traefik/DMZ IP
- Test SSH from manager-01: `ssh ubuntu@worker-media-01`
- Test remote deploy: `bash /mnt/docker-swarm/scripts/deploy-worker-scripts.sh --remote worker-media-01 --no-run`
- Add hardware alert rules: `proxmox-swarm/stacks/monitoring/alerts/hardware.yml`
- Add ZFS pool health alert rules: `proxmox-swarm/stacks/monitoring/alerts/zfs.yml`
- Add Swarm service alert rules: `proxmox-swarm/stacks/monitoring/alerts/swarm.yml`

---

## Session 2 — NFS Benchmark Multi-Mount + Grafana Split by Host/Mount

### Completed

- Extended `nfs-bench.sh` to benchmark multiple NFS mounts per run — `TVShows`, `Animation`, and `Movies` all benchmarked in a single cron invocation
- Added `worker-mediamanagement-01` as a second benchmarked VM alongside `worker-media-01`
- Fixed `nfs-bench.sh` bug: auto-install `fio` and `bc` via `apt-get` when missing on first run (commit `b911e1c`)
- Fixed `nfs-bench.sh` bug: auto-trigger autofs mounts before preflight check so unmounted paths are not skipped (commit `73dfc14`)
- Fixed `nfs-bench.sh` bug: skip `fio` entirely on read-only mounts rather than attempting writes (commit `a313033`)
- Fixed `nfs-bench.sh` bug: strip whitespace from metric values before writing to `.prom` file to prevent malformed Prometheus output (commit `7854902`)
- Fixed `nfs-bench.sh` bug: set `644` permissions on `.prom` file after atomic `mv` so `node_exporter` textfile collector can read it (commit `35bdbe3`)
- Fixed `deploy-worker-scripts.sh`: set correct permissions on `textfile_collector` directory on `worker-mediamanagement-01` (commit `1d8134c`)
- Fixed cron configuration: removed incorrect `--mount` flag from `nfs-bench-mediamanagement` cron entry (commit `d57222a`)
- Updated Grafana `backup-status.json` dashboard to v5: split NFS panels by `host` + `mount`, added Last Benchmark Run panel, added `$host` template variable dropdown (commit `c206f22`)
- Deployed `nfs-bench.sh` to both `worker-media-01` and `worker-mediamanagement-01`, fixed `textfile_collector` permissions, confirmed metrics flowing to Prometheus
- PR #47 (`feat(nfs-bench): multi-mount support + Grafana split by host/mount/time`) opened and rebased against main after PR #48 merged

### Notes

- `fio` cannot write to read-only NFS mounts — bench now performs read-only `fio` pass on those mounts and skips write test
- Autofs mounts under `/srv/nfs/` are not present until accessed; bench now `ls`es the path to trigger the mount before running preflight
- `textfile_collector` dir must be `755` and `.prom` files must be `644` for `node_exporter` to pick them up — `deploy-worker-scripts.sh` now sets this explicitly
- PR #47 branch: `claude/clever-elbakyan-7f7255`; not yet merged as of end of session — rebase conflict with main resolved

## Related Notes

- [[01 Homelab Rebuild - Phase 7 Backup Verification Hub]]
- [[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]]
- [[Docker Swarm Infrastructure Runbook]]
