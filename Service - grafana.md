---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - Monitoring
service_name: Grafana
vm: manager-01
swarm_constraint: node.role == monitoring
vlan: 60
service_status: deployed
stack_file: /mnt/docker-swarm/stacks/monitoring/stack.yml
port: 3000
external_access: false
traefik_entrypoint: internal
url_internal: https://grafana.home.purvishome.com
zfs_dataset: rpool/docker-data/grafana
mount_path: /mnt/docker-data/grafana
last_updated: 2026-04-02
---

# Grafana

Metrics dashboard. Connected to Prometheus and InfluxDB datasources.
Currently running on legacy stack — needs migration to virtiofs path post-Phase 2.
