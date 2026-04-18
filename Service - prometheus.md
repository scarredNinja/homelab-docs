---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - Monitoring
service_name: Prometheus
vm: manager-01
swarm_constraint: node.role == monitoring
vlan: 60
service_status: deployed
stack_file: /mnt/docker-swarm/stacks/monitoring/stack.yml
port: 9090
external_access: false
traefik_entrypoint: internal
url_internal: https://prometheus.home.purvishome.com
zfs_dataset: rpool/docker-tsdb/prometheus
mount_path: /mnt/docker-tsdb/prometheus
last_updated: 2026-04-02
---

# Prometheus

Metrics scraper. High-write TSDB — pinned to manager, local virtiofs tsdb dataset.
History loss acceptable on migration — rebuilds from scrape targets within hours.
