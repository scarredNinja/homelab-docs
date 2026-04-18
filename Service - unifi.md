---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - Networking

service_name: UniFi Controller
vm: worker-controller-01
swarm_constraint: "node.labels.zone == controller"
vlan: 60

service_status: pending
stack_file: /mnt/docker-swarm/stacks/controllers/stack.yml
port: 8443
external_access: false
traefik_entrypoint: internal
url_internal: https://unifi.home.purvishome.com

zfs_dataset: rpool/docker-data/uniFi
mount_path: /mnt/docker-data/uniFi

last_updated: 2026-04-02
---

# UniFi Controller

WiFi and AP management. Uses MongoDB — local virtiofs, never NFS.
Needs VLAN 20 reach for device adoption.
Pinned to controller worker — device adoption state is local.
