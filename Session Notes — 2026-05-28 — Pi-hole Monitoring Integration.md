---
date: '2026-05-28T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 8: Basic Monitoring'
session_type: Code
status: Completed
tags:
  - SessionNotes
  - Monitoring
  - Prometheus
  - pihole
  - DockerSwarm
---

# Session Notes — 2026-05-28 — Pi-hole Monitoring Integration

## 🎯 Session Goal

Add full monitoring coverage for the two Pi-hole Raspberry Pi instances (`pihole1` at `10.0.60.20`, `pihole2` at `10.0.60.21`). Two layers targeted:
1. **System metrics** — `node_exporter` installed natively on each Pi (CPU, RAM, disk, network).
2. **Pi-hole DNS/ad-block metrics** — `pihole-exporter` (`eko/pihole-exporter`) deployed as a Docker Swarm service in the monitoring stack.

---

## ✅ Completed

### 1. Prometheus Config Updated — `config/monitoring/prometheus.yml`

- Added `pihole1` and `pihole2` as static targets in the existing `node_exporter_external` job under a new `vlan60` label block (port `9100`).
- Added new `pihole-exporter` scrape job targeting the Swarm internal DNS name `tasks.monitoring_pihole-exporter:9617`.

### 2. Monitoring Stack Updated — `proxmox-swarm/stacks/stack-monitoring.yml`

- Added `pihole-exporter` service (`ekofr/pihole-exporter:v0.4.0`):
  - Monitors both Pi-holes via comma-separated `PIHOLE_HOSTNAME: "10.0.60.20,10.0.60.21"`.
  - Shell `entrypoint` wrapper reads `pihole_password` Docker Secret into `PIHOLE_PASSWORD` at runtime (image does not natively support `_FILE` suffix).
  - Pinned to `zone=monitoring`, connected to `monitoring` overlay network, port `9617`.
- Registered `pihole_password` as an external Docker Secret in the secrets block.

### 3. Bug Fix — Prometheus Removed Flag

> [!bug] Root Cause
> Prometheus was crash-looping with `unknown long flag '--storage.tsdb.max-bytes'`. This flag was removed from Prometheus and does not exist in `v2.51.2`. The replacement is `--storage.tsdb.retention.size`.
>
> **Fix:** `stack-monitoring.yml` updated from `--storage.tsdb.max-bytes=8GB` → `--storage.tsdb.retention.size=8GB`. Both `retention.time` and `retention.size` flags are now set; Prometheus honours whichever limit is hit first.

### 4. Documentation Updated

- `Pi-Hole Installation Steps.md` — tasks marked complete; new **Monitoring Setup** section added documenting both layers with install commands and Prometheus config snippets.
- `01 Homelab Rebuild - Phase 8 Basic Monitoring Hub.md` — Pi-hole tasks marked complete; Pi-hole row added to the app-specific exporters table.

### 5. Git — PR #66 Merged

- Branch `feature/pihole-monitoring` created, committed, pushed, PR raised and squash-merged to `main`.
- Branch deleted post-merge.

---

## ⚠️ Outstanding

> [!warning] pihole-exporter has no logs yet
> After deploying the updated monitoring stack, `docker service logs monitoring_pihole-exporter` returned empty output. The service may not be starting correctly. Likely causes:
> - `pihole_password` Docker Secret not yet created on the Swarm.
> - `node_exporter` not yet installed on the Pis (port `9100` unreachable, may cause exporter to error on startup).
> - Possible binary path mismatch in the shell entrypoint (`/app/pihole-exporter`).
> Deferred to next session.

---

## 📊 Current State (End of Session)

| Component | Status | Notes |
|---|---|---|
| `pihole-exporter` Swarm service | ⚠️ Deployed, logs TBC | Service running but metrics not yet verified |
| `node_exporter` on pihole1 | ✅ Healthy | Confirmed UP in Prometheus targets 2026-05-29 |
| `node_exporter` on pihole2 | ✅ Healthy | Confirmed UP in Prometheus targets 2026-05-29 |
| `pihole_password` Docker Secret | ✅ Created | Required for pihole-exporter |
| Prometheus `--storage.tsdb.max-bytes` bug | ✅ Fixed | Replaced with `--storage.tsdb.retention.size=8GB` |
| Prometheus config (`prometheus.yml`) | ✅ Updated | vlan60 targets + pihole-exporter job added |
| PR #66 | ✅ Merged to main | Squash merge, branch deleted |

---

## ➡️ Next Session Priorities

- [x] SSH into `pihole1` (10.0.60.20) and `pihole2` (10.0.60.21) — install `prometheus-node-exporter` via `apt` and verify port `9100` is responding
- [x] Create the `pihole_password` Docker Secret on a Swarm manager
- [x] Verify both Pi-holes appear as UP targets in Prometheus at `https://prometheus.home.purvishome.com/targets` ✅ 2026-05-29
- [ ] Confirm `pihole-exporter` Swarm service is scraping DNS metrics — check `docker service logs monitoring_pihole-exporter` and `curl` port `9617`
- [ ] Import Grafana dashboard ID **10176** (Pi-hole Exporter) once metrics are confirmed flowing

---

## 🔗 Related Notes

- [[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]]
- [[Pi-Hole Installation Steps]]
- [[Service - prometheus]]
- [[Session Notes — 2026-05-22 — Pi-hole Gravity Sync]]
