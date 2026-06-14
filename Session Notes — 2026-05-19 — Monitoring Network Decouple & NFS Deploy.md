---
date: '2026-05-19'
phase: 'Phase 5: Docker Swarm & Virtualisation'
project_id: Homelab-2025
session_type: Code
status: Completed
tags:
  - Exportarr
  - Monitoring
  - NFS
  - Networking
  - Prometheus
  - SessionNotes
  - WireGuard
---
# Session Notes — 2026-05-19 — Monitoring Network Decouple & NFS Deploy

## 🎯 Session Goal

Decouple the `monitoring` overlay network from stack ownership so the
monitoring stack can be stopped or redeployed without tearing down
connections in the arr and controller stacks. Also finalise the NFS
benchmark and Prometheus VLAN 50 scrape work from PR #40 and deliver
a deploy script for worker-media-01.

---

## ✅ Completed

### Monitoring overlay network decoupled (PR #41)

`stack-monitoring.yml` now declares the `monitoring` network as
`external: true` — the network is no longer owned by the stack. A new
idempotent script `create-monitoring-network.sh` creates the standalone
overlay (subnet `10.200.1.0/24`, `--internal`) before first deploy and
documents the one-time migration steps.

Stacks that attach to `monitoring` were updated to remove the scoped name
override (`name: monitoring_monitoring`) that was patching around the old
stack-owned network name:
- `stack-arr.yml` — `name:` override removed
- `proxmox-swarm/stacks/controller/stack-controller.yml` — `monitoring_ext`
  alias renamed to `monitoring`; `name:` override removed

### Exportarr sidecars deployed to arr stack (PR #39)

`sonarr-exporter` (port 9707) and `radarr-exporter` (port 9708) sidecar
services added to `stack-arr.yml`. Both join the `monitoring` overlay.
Prometheus scrape jobs targeting `10.0.50.30:9707` and `10.0.50.30:9708`
added to `prometheus.yml`.

> [!important] `SONARR_API_KEY` and `RADARR_API_KEY` must be set as
> Portainer stack env vars before arr stack is redeployed — not in YAML.

### Gluetun migrated to WireGuard / NordLynx (PR #39)

`compose-vpn.yml` updated to use `VPN_TYPE=wireguard` / `VPN_SERVICE_PROVIDER=nordvpn`.
OpenVPN env_file commented out. `WIREGUARD_PRIVATE_KEY` placeholder and
extraction instructions added. Eliminates the single-threaded OpenVPN CPU
pin that caused 3–8 s Arr UI latency at download saturation.

### Prometheus VLAN 50 external scrape targets added (PR #40)

`node_exporter_external` static-config job added to `prometheus.yml`
targeting media VMs on VLAN 50 (e.g. `10.0.50.50:9100`). Enables
node-exporter scraping of `worker-media-01` and `worker-mediamanagement-01`
across VLAN boundaries without overlay network membership.

### nfs-bench.sh audited and cron reference file added (PR #40)

`nfs-bench.sh` reviewed and corrected. `proxmox-swarm/cron/nfs-bench`
reference cron file added for deployment to `worker-media-01`.
`nfs_bench_last_exit_code` metric added. NFS performance Grafana row merged
into `backup-status.json` dashboard. `textfile_collector` volume and flag
added to node-exporter in `stack-monitoring.yml`.

### restic deferred-run sentinel added (PR #39)

`restic-backup.sh` writes `/var/run/restic-deferred` before exiting on
deferral and removes it on success. `vm-backup.sh` checks for the sentinel
at the end of its run and exec's `restic-backup.sh` if present. A safety-net
cron entry at 08:00 added to `deploy-backup-scripts.sh` heredoc. Cron
schedule documentation in `cron-jobs.md` synced with actual timings.

### deploy-worker-scripts.sh script added and iteratively fixed

New script `proxmox-swarm/scripts/deploy-worker-scripts.sh` automates
deployment of `nfs-bench.sh` and the cron file to `worker-media-01` and
runs the benchmark once to seed the `.prom` file.

Four fix commits followed initial implementation:
- Added `--user` flag; default SSH user changed to `docker`; default host
  changed to `10.0.50.50` (hostname may not resolve on PVE) — commit `5875001`
- Added `--key` flag for SSH private key path; SSH opts array passed
  consistently to all `ssh`/`scp` calls — commit `c385854`
- Refactored to run locally on the target VM (copy files to `/tmp/`, then
  execute as root on the VM directly, no persistent SSH session) — commit `b9f63f9`
- Fixed expected path from `nfs-bench-cron` to `/tmp/nfs-bench` — commit `9136deb`

---

## 📋 Current State (End of Session)

| Component | Status |
|---|---|
| `monitoring` overlay — standalone external network | ✅ Deployed |
| `stack-arr.yml` — monitoring network name fix | ✅ Deployed |
| `stack-controller.yml` — alias rename | ✅ Deployed |
| Exportarr sonarr-exporter (9707) | ✅ Deployed & verified UP |
| Exportarr radarr-exporter (9708) | ✅ Deployed & verified UP |
| Gluetun WireGuard migration | ✅ Deployed & verified UP |
| Prometheus VLAN 50 scrape targets | ✅ Deployed |
| nfs-bench.sh on worker-media-01 | ✅ Deployed |
| Grafana NFS Performance panels | ✅ Deployed |
| pfSense VLAN60→VLAN50 TCP/9100 rule | ✅ Deployed |
| restic deferred sentinel | ✅ Deployed |
| PR #39 | ✅ Merged 2026-05-19 |
| PR #40 | ✅ Merged 2026-05-19 |
| PR #41 | ✅ Merged 2026-05-19 |

---

## 🎯 Next Session Priorities

- [x] Add pfSense firewall rule: VLAN 60 → VLAN 50, TCP port 9100 (node-exporter) [priority::1]
- [x] Run `create-monitoring-network.sh` on Proxmox host / manager-01, then redeploy monitoring stack [priority::1]
- [x] Redeploy arr stack — set `SONARR_API_KEY` + `RADARR_API_KEY` in Portainer env, set `WIREGUARD_PRIVATE_KEY` in `compose-vpn.yml` first [priority::1]
- [x] Redeploy controller stack after monitoring network migration [priority::2]
- [x] Run `deploy-worker-scripts.sh` on worker-media-01 to deploy nfs-bench and seed `.prom` file [priority::2]
- [x] Import updated `backup-status.json` Grafana dashboard (NFS bench row) [priority::2]
- [x] Re-run `deploy-backup-scripts.sh` to install restic deferred sentinel [priority::2]
- [x] Run nfs-bench.sh on worker-mediamanagement-01 for second data point [priority::3]

---

## 🔗 Related Notes

- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
- [[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]]
- [[01 Homelab Rebuild - Phase 7 Backup Verification Hub]]
- [[Docker Swarm Infrastructure Runbook]]
- [[Session Notes — 2026-05-17 — WireGuard Migration & Exportarr]]
- [[Session Notes — 2026-05-14 — Restic Crash Diagnosis & NFS Benchmark]]
