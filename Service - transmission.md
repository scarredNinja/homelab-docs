---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - Media

service_name: Transmission
vm: worker-media-01
swarm_constraint: "node.labels.type == media"
vlan: 50

service_status: pending
stack_file: /mnt/docker-swarm/stacks/media/stack.yml
port: 9091
external_access: false
traefik_entrypoint: internal
url_internal: https://transmission.home.purvishome.com

zfs_dataset: rpool/docker-data/transmission
mount_path: /mnt/docker-data/transmission

last_updated: 2026-04-02
---

# Transmission

Torrent client. Runs behind Gluetun VPN sidecar — shares Gluetun network namespace.
Downloads to NAS NFS share. Config/resume state on virtiofs.

## Notes

- `network_mode: service:gluetun` — Transmission port 9091 exposed via Gluetun
- NordVPN kill switch enforced via pfSense VLAN 50 rules
- Do not put Prowlarr behind the VPN — needs tracker access
