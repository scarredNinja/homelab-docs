---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - Dashboard
service_name: Homepage
vm: worker-monitoring-01
swarm_constraint: node.labels.zone == monitoring
vlan: 60
service_status: running
stack_file: proxmox-swarm/stacks/stack-homepage.yml
comment: stack_file is repo-relative (Portainer Git deployment)
port: 3000
external_access: false
traefik_entrypoint: websecure
url_internal: https://homepage.home.purvishome.com
zfs_dataset: rpool/docker-data/homepage
mount_path: /mnt/docker-data/homepage
last_updated: 2026-05-08
---

# Homepage

Self-hosted application dashboard (gethomepage.dev). Pinned to `worker-monitoring-01`. Provides a unified view of all homelab services with status widgets.

## Deployment

- 🟡 In progress — branch `claude/elegant-lehmann-00bda3` (4 commits, no PR yet)
  - `e158ea3` — stack + scripts
  - `48c7355` — tag fix
  - `e2e0e4c` — healthcheck/resources
  - `1f9a3a6` — Traefik network label fix (untested)
- Container healthy on `worker-monitoring-01` ✅
- Traefik routing not yet confirmed — pending test of last commit
- Stack: `proxmox-swarm/stacks/stack-homepage.yml`
- Config files: `/mnt/docker-data/homepage/` (virtiofs from Proxmox host)
- Networks: `traefik-public`
- Middleware: `internal-only@file`

## Config Files

| File | Purpose |
|------|---------|
| `settings.yaml` | Theme, layout, general settings |
| `services.yaml` | Service tiles and groupings |
| `widgets.yaml` | Top-level info widgets (date, stats) |
| `bookmarks.yaml` | Quick-access bookmarks |
| `docker.yaml` | Docker integration (optional) |

## Planned Service Layout

| Group | Services |
|-------|---------|
| Monitoring | Grafana, Prometheus, Uptime Kuma |
| Management | Portainer, Proxmox, pfSense, Traefik |
| Media | Plex, Tautulli |
| Downloads | Sonarr, Radarr, Prowlarr, Transmission |
| Home | Home Assistant, UniFi |

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[VM - monitoring-01]]
- [[Service - uptime-kuma]]
- [[Service - grafana]]
