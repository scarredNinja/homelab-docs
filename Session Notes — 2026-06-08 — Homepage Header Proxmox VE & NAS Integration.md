---
type: session-note
project_id: Homelab-2025
tags:
  - SessionNotes
  - Homepage
  - Proxmox
  - Monitoring
last_updated: '2026-06-08T00:00:00.000Z'
---

# Session Notes — 2026-06-08 — Homepage Header Proxmox VE & NAS Integration

## Session Goal

Integrate physical hardware resource monitoring (Proxmox VE host, primary/backup Synology NAS status) and live interface bandwidth rates into the Homepage dashboard, configure an anime-style Japanese bridge background image, and style the panels with a modern glassmorphism aesthetic.

## Completed

- **Proxmox VE & Synology NAS Status Integration:**
  - Configured Proxmox node metrics widget to report CPU, memory, uptime, and disk usage for the hypervisor host.
  - Integrated Synology `diskstation` widgets for both primary and backup Synology NAS units using Docker secrets to retrieve and display volumes status, disk temperatures, and system health.
- **Prometheus Metric widgets for Interface Bandwidth rates:**
  - Added custom Prometheus Metric widgets at the top of the dashboard querying traffic rates on host interfaces `vmbr1` (main bridge) and `enp5s0` (physical network card) from `node-exporter` metrics.
  - Formatted metrics to display RX/TX rates dynamically in bytes/sec or Mbps.
- **Custom Styling & Glassmorphism Theme:**
  - Configured custom anime-style Japanese bridge background image via `/images/background.png` volume mount.
  - Applied frosted-glass panel effects, thin borders, subtle shadows, and hover animations in `custom.css` to achieve a clean, modern glassmorphism aesthetic.
- **Branch Checkout & Git Push:**
  - Checked out the new feature branch `feature/homepage-proxmox-integration`.
  - Staged, committed, and pushed the updated Homepage configuration files and stack definition.

## Related Notes

- [[Service - homepage]]
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
