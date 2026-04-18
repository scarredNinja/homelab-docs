---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - SmartHome

service_name: Home Assistant
vm: worker-controller-01
swarm_constraint: "node.labels.zone == controller"
vlan: 60

service_status: pending
stack_file: /mnt/docker-swarm/stacks/controllers/stack.yml
port: 8123
external_access: false
traefik_entrypoint: internal
url_internal: https://ha.home.purvishome.com

zfs_dataset: rpool/docker-data/homeassistant
mount_path: /mnt/docker-data/homeassistant

last_updated: 2026-04-02
---

# Home Assistant

Smart home automation. Needs VLAN 20 (IoT) access via secondary NIC on controller worker.
Single-instance — must never float to another node.

## Notes

- Run `ha core stop` before container shutdown — SQLite corruption risk on unclean stop
- Secondary NIC on VLAN 20 for direct IoT device reach
