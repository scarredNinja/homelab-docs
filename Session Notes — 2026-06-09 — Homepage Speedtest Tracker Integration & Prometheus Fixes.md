---
type: session-note
project_id: Homelab-2025
tags:
  - SessionNotes
  - Homepage
  - Monitoring
  - Speedtest
last_updated: '2026-06-09T00:00:00.000Z'
---

# Session Notes — 2026-06-09 — Homepage Speedtest Tracker Integration & Prometheus Fixes

## Session Goal

Deploy a containerized Speedtest Tracker service (`lscr.io/linuxserver/speedtest-tracker:latest`) within the Docker Swarm monitoring zone, integrate its real-time download/upload/ping metrics into the Homepage dashboard, scrape metrics into Prometheus for Grafana visualization, and troubleshoot integration errors.

## Completed

- **Speedtest Tracker Deployment:**
  - Deployed `stack-speedtest.yml` running on `worker-monitoring-01` with Traefik routing at `speedtest.home.purvishome.com` (using `internal-only@file` middleware).
  - Setup SQLite database backend mapped to `/mnt/docker-data/speedtest`.
  - Scheduled speed tests to run every 2 hours at minute 30 (`30 */2 * * *`) in timezone `Pacific/Auckland`.
- **Homepage Widget Debugging & Integration:**
  - Encountered Filament login page redirect issues (`Unexpected token '<'`) due to native widget authentication formatting.
  - Switched the Homepage widget to `customapi` type to explicitly send `Accept: application/json` and `Authorization: Bearer <key>` headers.
  - Recreated `speedtest_api_key` Docker Secret cleanly using `printf '%s'` on the Swarm manager to prevent trailing newlines.
  - Resolved an 8x throughput display mismatch (`12 Mbps` vs `93.51 Mbps`) by changing customapi mapping from `data.download`/`data.upload` (bytes/sec) to `data.download_bits`/`data.upload_bits` (bits/sec).
- **Prometheus Metrics Fix:**
  - Resolved `404 Not Found` scraping errors by changing `metrics_path` in `prometheus.yml` from `/prometheus` to `/metrics` (the default endpoint exposed by Speedtest Tracker v2).

## Follow-up Tasks

- [ ] **Throughput Limitation Investigation:**
  - Check why speed tests show ~93 Mbps download / ~85 Mbps upload (under 100 Mbps) when the actual internet connection is expected to be much higher (e.g. 300+ Mbps or 1 Gbps). [priority::1]
  - *Investigation Checklist:*
    - [ ] Run `ethtool` or check link speed on the physical host NIC (`enp5s0` or equivalent) on the Proxmox node to verify it is negotiated at 1 Gbps/10 Gbps and not 100 Mbps Fast Ethernet.
    - [ ] Check the VM virtual NIC configuration in Proxmox (ensure it is set to `VirtIO (paravirtualized)` and not a legacy 100M device).
    - [ ] Run a manual speedtest directly from the host command line (or another VM/LXC) to isolate container overhead.
    - [ ] Verify if the select speedtest server inside the container UI is bottlenecking.

## Related Notes

- [[Service - homepage]]
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
