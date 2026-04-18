---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - Media

service_name: Plex
vm: worker-media-01
swarm_constraint: "node.labels.type == media"
vlan: 50

service_status: pending
stack_file: /mnt/docker-swarm/stacks/media/stack.yml
port: 32400
external_access: true
traefik_entrypoint: websecure
url_internal: https://plex.home.purvishome.com

zfs_dataset: rpool/docker-data/plex
mount_path: /mnt/docker-data/plex

last_updated: 2026-04-02
---

# Plex

Media server. Only service that requires `websecure` entrypoint — remote access needs inbound internet.

## Notes

- Config/metadata on virtiofs docker-data — SQLite must not go on NFS
- Transcode cache on local ZFS zvol for IOPS
- Media files on NAS via NFS mount
- `PLEX_UID=1500`, `PLEX_GID=1500`
