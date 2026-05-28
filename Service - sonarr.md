---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - Media
service_name: Sonarr
vm: worker-mediamanagement-01
swarm_constraint: node.labels.zone == mediamanagement
vlan: 50
service_status: running
stack_file: /mnt/docker-swarm/stacks/arr/stack-arr.yml
port: 8989
external_access: false
traefik_entrypoint: websecure
url_internal: https://sonarr.home.purvishome.com
zfs_dataset: rpool/docker-data/sonarr
mount_path: /mnt/docker-data/sonarr
last_updated: 2026-05-20
---

# Sonarr

TV series automation. SQLite DB — must stay on virtiofs, never NFS.
Pinned to `worker-mediamanagement-01` — concurrent SQLite access from multiple nodes = corruption.

## Notes

- Download client: **host = `gluetun`** (not `transmission`) — port 9091 lives on the Gluetun network namespace ✅ 2026-04-25
- Metrics Exporter: **`exportarr` sidecar container** deployed on port `9707` in `stack-arr.yml`. Configured using prefix-free environment variables (`PORT`, `URL`, `API_KEY_FILE`) and clean `sonarr_apikey_v2` secret. Prometheus scrapes metrics at `10.0.50.51:9707` (UP and verified ✅ 2026-05-20).
- Status: **Fully operational and monitored ✅ 2026-05-20**.
