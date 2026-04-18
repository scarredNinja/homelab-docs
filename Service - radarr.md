---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - Media

service_name: Radarr
vm: worker-media-01
swarm_constraint: "node.labels.type == media"
vlan: 50

service_status: pending
stack_file: /mnt/docker-swarm/stacks/media/stack.yml
port: 7878
external_access: false
traefik_entrypoint: internal
url_internal: https://radarr.home.purvishome.com

zfs_dataset: rpool/docker-data/radarr
mount_path: /mnt/docker-data/radarr

last_updated: 2026-04-02
---

# Radarr

Movie automation. SQLite DB — must stay on virtiofs, never NFS.
Pinned to media worker alongside Sonarr and Plex.
