---
type: swarm-service
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
tags:
  - DockerSwarm
  - Service
  - Media
  - Monitoring
service_name: Tautulli
vm: worker-media-01
swarm_constraint: node.labels.type == media
vlan: 50
service_status: running
stack_file: proxmox-swarm/stacks/stack-plex.yml
comment: stack_file is repo-relative (Portainer Git deployment)
port: 8181
external_access: false
traefik_entrypoint: websecure
url_internal: 'https://tautulli.home.purvishome.com'
zfs_dataset: rpool/docker-data/tautulli
mount_path: /mnt/docker-data/tautulli
last_updated: '2026-05-08T00:00:00.000Z'
status: Completed
---

# Tautulli

Plex monitoring and statistics. Pinned to `worker-media-01` — needs local Plex access. SQLite DB — virtiofs only.

## Deployment

- ✅ Deployed — part of `stack-plex.yml`
- ✅ Portainer Git integration — `proxmox-swarm/stacks/stack-plex.yml` (main)
- Networks: `traefik-public`

## Access

- Internal: `https://tautulli.home.purvishome.com`
- Middleware: `internal-only@file`

## Notes

- DNS issue resolved 2026-05-08 — was unreachable due to missing DNS entry
- Plex token required in Tautulli config for stats collection
- Data at `/mnt/docker-data/tautulli/`

## Related

- [[Service - plex]]
- [[VM - worker-media-01]]
