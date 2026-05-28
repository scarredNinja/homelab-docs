---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - Monitoring
service_name: Prometheus
vm: worker-monitoring-01
swarm_constraint: node.labels.zone == monitoring
vlan: 60
service_status: running
stack_file: /mnt/docker-swarm/stacks/monitoring/stack.yml
port: 9090
external_access: false
traefik_entrypoint: websecure
url_internal: https://prometheus.home.purvishome.com
zfs_dataset: rpool/docker-tsdb/prometheus
mount_path: /mnt/docker-tsdb/prometheus
last_updated: 2026-04-24
---

# Prometheus

Metrics scraper. High-write TSDB — pinned to monitoring worker, local virtiofs tsdb dataset.
History loss acceptable on migration — rebuilds from scrape targets within hours.

## Notes

- Solar history (5.4 GB) migrated from CT109 ✅ 2026-04-24
- Post-rsync chown 65534:65534 applied (Gotcha #36) ✅ 2026-04-24
- Data sources in Grafana may need updating to reflect new Swarm service names
