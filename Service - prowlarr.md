---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - Media

service_name: Prowlarr
vm: worker-media-01
swarm_constraint: "node.labels.type == media"
vlan: 50

service_status: pending
stack_file: /mnt/docker-swarm/stacks/media/stack.yml
port: 9696
external_access: false
traefik_entrypoint: internal
url_internal: https://prowlarr.home.purvishome.com

zfs_dataset: rpool/docker-data/prowlarr
mount_path: /mnt/docker-data/prowlarr

last_updated: 2026-04-02
---

# Prowlarr

Indexer manager for Sonarr/Radarr. Must NOT go behind VPN — needs direct tracker access.
Runs on standard WAN routing, not Gluetun namespace.
