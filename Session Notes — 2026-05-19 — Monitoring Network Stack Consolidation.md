---
project_id: Homelab-2025
date: '2026-05-19T00:00:00.000Z'
tags:
  - session-note
  - homelab
status: Completed
phase: 'Phase 5: Docker Swarm'
---

# Session Notes — 2026-05-19 — Monitoring Network Stack Consolidation

## Session Goal

Decouple the monitoring overlay network from the monitoring stack, deploy NFS benchmarking and Prometheus VLAN 50 scrape targets, add a restic deferred-run sentinel to prevent silent backup skips, and consolidate all stack files under a single directory.

Update gluetan to persist the wireguard api key to a secret file

## Completed

- Decoupled the `monitoring` overlay network from `stack-monitoring.yml` — network is now an external standalone resource created by `create-monitoring-network.sh` (subnet `10.200.1.0/24`, `--internal`), preventing monitoring stack teardowns from disconnecting `stack-arr.yml` and `stack-controller.yml` (PR #41)
- Updated `stack-arr.yml` and `stack-controller.yml` to remove the `name: monitoring_monitoring` override and reference the external network directly (PR #41)
- Delivered `deploy-worker-scripts.sh` — local install script that copies `nfs-bench.sh` and cron file to `/tmp/` on `worker-media-01` and runs the benchmark to seed the `.prom` file (PR #41)
- Added `node_exporter_external` Prometheus scrape job with static targets for VLAN 50 media VMs, plus `exportarr-sonarr` and `exportarr-radarr` scrape jobs targeting `worker-mediamanagement-01` at `10.0.50.30` (PR #40)
- Audited and deployed `nfs-bench.sh` — writes `nfs_bench_last_exit_code` metric to textfile collector; added `proxmox-swarm/cron/nfs-bench` reference cron file and merged NFS performance row into `backup-status.json` Grafana dashboard (PR #40)
- Added `textfile_collector` volume and flag to `stack-monitoring.yml` node-exporter service so `.prom` files from `nfs-bench.sh` and `disk-space-check.sh` are ingested (PR #40)
- Fixed restic deferred-run gap: `restic-backup.sh` now writes `/var/run/restic-deferred` sentinel on deferral; `vm-backup.sh` releases its lock and disarms the EXIT trap before checking the sentinel, then `exec`s `restic-backup.sh` if the sentinel is present — eliminating silent overnight backup skips (PR #38)
- Added 08:00 safety-net cron entry to `deploy-backup-scripts.sh` heredoc to match the entry already documented in `cron-jobs.md` (PR #38)
- Consolidated all Docker Swarm stack files under `proxmox-swarm/stacks/` — moved `stacks/stack-restic.yml` into the canonical path and updated `CLAUDE.md` directory layout and `yamllint` commands (PR #42)
- Moved `stack-controller.yml` out of `controller/` subdirectory to flatten the stacks directory; updated path references in `create-monitoring-network.sh` and `copy-swarm-config.sh` (PR #43)
- Migrated `compose-vpn` Gluetun from OpenVPN to WireGuard (NordLynx) — eliminates single-threaded CPU pin that caused 3–8s Arr UI latency at download saturation; `WIREGUARD_PRIVATE_KEY` placeholder added with operator extraction instructions (PR #39)
- Added `sonarr-exporter` and `radarr-exporter` Exportarr sidecar services to `stack-arr.yml` (ports 9707/9708) with monitoring overlay network attachment (PR #39)
- Appended `exportarr-sonarr` and `exportarr-radarr` scrape jobs to `prometheus.yml` targeting `worker-mediamanagement-01` at `10.0.50.30` (PR #39)
- Fixed `automation/run-session-note.ps1` — removed `2>>` stderr redirect from `claude.exe` call; PowerShell 5.1 wraps native exe stderr in `ErrorRecord` objects and triggers Stop errors even on success (PR #44)
- Gluetan update done on on `compose-vpn.yml` (done in PR #45)

## Notes

- Post-deploy checklist for PR #40 is not yet complete: pfSense firewall rule for VLAN60→VLAN50 TCP/9100, monitoring stack redeploy to pick up the `textfile_collector` mount, and Grafana NFS dashboard row import are still outstanding.
- Post-deploy checklist for PR #39 is not yet complete: `WIREGUARD_PRIVATE_KEY` must be set in Portainer env vars, Sonarr/Radarr API keys needed for Exportarr, arr stack redeploy required.
- `nfs-bench.sh` has only been run on `worker-media-01` — a second data point from `worker-mediamanagement-01` is still outstanding.
- The monitoring network migration requires a one-time manual step: remove the old stack-owned `monitoring` network and recreate it with `create-monitoring-network.sh` before redeploying the monitoring stack.

## Related Notes

[[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
[[01 Homelab Rebuild - Phase 7 Backup Verification Hub]]
[[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]]
