---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - Media
  - Monitoring

service_name: Tautulli
vm: worker-media-01
swarm_constraint: "node.labels.type == media"
vlan: 50

service_status: pending
stack_file: /mnt/docker-swarm/stacks/media/stack.yml
port: 8181
external_access: false
traefik_entrypoint: internal
url_internal: https://tautulli.home.purvishome.com

zfs_dataset: rpool/docker-data/tautulli
mount_path: /mnt/docker-data/tautulli

last_updated: 2026-04-02
---

# Tautulli

Plex monitoring and statistics. Pinned to media worker — needs local Plex access.
SQLite DB — virtiofs only.
