---
project_id: "Homelab-2025"
date: 2026-06-12
tags:
  - session-note
  - homelab
  - monitoring
status: Completed
phase: "Phase 8: Basic Monitoring"
---

# Session Notes — 2026-06-12 — InfluxDB and Pi-hole Exporter Config Fixes

## Session Goal
Resolve the InfluxDB migration conflict and configure Pi-hole Exporter to run inside an Alpine environment to read Docker secrets.

## Completed
- **Updated InfluxDB Image Tag**: Pinned the InfluxDB container image version to `2.9.0` (from `2.7`) in `stack-monitoring.yml` to resolve database migration conflicts.
- **Switched Pi-hole Exporter to Debian**: Migrated the container base image from `ekofr/pihole-exporter:v0.4.0` to `debian:stable-slim` in `stack-monitoring.yml` to resolve libc/glibc compatibility issues with the dynamically linked Go binary (fixing the `/app/pihole-exporter: not found` error seen on Alpine).
- **Configured Secrets Wrapper for Exporter**: Mounted the `pihole-exporter` binary from `/mnt/docker-data/pihole-exporter` to `/app:ro` and added a shell wrapper entrypoint to read the Docker secret (`pihole_password`) and execute the binary, dynamically stripping trailing newlines and carriage returns (`tr -d '\r\n'`) to prevent invalid API authentication.

## Related Notes
[[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]]
