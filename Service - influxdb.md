---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - Monitoring
service_name: InfluxDB
vm: worker-monitoring-01
swarm_constraint: node.labels.zone == monitoring
vlan: 60
service_status: running
stack_file: /mnt/docker-swarm/stacks/monitoring/stack.yml
port: 8086
external_access: false
traefik_entrypoint: websecure
url_internal: https://influxdb.home.purvishome.com
zfs_dataset: rpool/docker-tsdb/influxdb
mount_path: /mnt/docker-tsdb/influxdb
last_updated: 2026-04-24
---

# InfluxDB

Time-series DB for Proxmox and network metrics. High-write — local tsdb dataset.
ZFS recordsize 8K matches InfluxDB write block size.
