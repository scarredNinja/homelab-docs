---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - Monitoring
service_name: InfluxDB
vm: manager-01
swarm_constraint: node.role == monitoring
vlan: 60
service_status: deployed
stack_file: /mnt/docker-swarm/stacks/monitoring/stack.yml
port: 8086
external_access: false
traefik_entrypoint: internal
url_internal: https://influxdb.home.purvishome.com
zfs_dataset: rpool/docker-tsdb/influxDB
mount_path: /mnt/docker-tsdb/influxdb
last_updated: 2026-04-02
---

# InfluxDB

Time-series DB for Proxmox and network metrics. High-write — local tsdb dataset.
ZFS recordsize 8K matches InfluxDB write block size.
