---
type: swarm-service
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
tags:
  - DockerSwarm
  - Service
  - Monitoring
service_name: Uptime Kuma
vm: worker-monitoring-01
swarm_constraint: node.labels.zone == monitoring
vlan: 60
service_status: running
stack_file: proxmox-swarm/stacks/stack-uptime-kuma.yml
comment: stack_file is repo-relative (Portainer Git deployment)
port: 3001
external_access: false
traefik_entrypoint: websecure
url_internal: 'https://uptime-kuma.home.purvishome.com'
zfs_dataset: rpool/docker-data/uptime-kuma
mount_path: /mnt/docker-data/uptime-kuma
last_updated: '2026-05-26T00:00:00.000Z'
status: Completed
---

# Uptime Kuma

Service uptime monitoring dashboard. Pinned to `worker-monitoring-01`.

## Deployment

- ✅ Deployed 2026-05-08 via Portainer; redeployed 2026-05-21 after routing fix
- ✅ Portainer Git integration — `proxmox-swarm/stacks/stack-uptime-kuma.yml` (main)
- Networks: `traefik-public`, `monitoring_monitoring`, `arr_arr_default`

## Monitors Active

| Monitor | URL / Target | Type |
|---------|-------------|------|
| Prometheus | `http://monitoring_prometheus:9090` | HTTP |
| Grafana | `http://monitoring_grafana:3000` | HTTP |
| Transmission | `http://gluetun:9091` (via arr_arr_default) | HTTP |
| Sonarr | `http://arr_sonarr:8989` | HTTP |
| Radarr | `http://arr_radarr:7878` | HTTP |
| Prowlarr | `http://arr_prowlarr:9696` | HTTP |
| pfSense | `https://10.0.60.1` | HTTP |
| Home Assistant | `http://10.0.60.42:8123` | HTTP |

> [!note] Monitor pattern
> Use internal Swarm DNS (`stack_service:port`) for overlay-networked services. Use VM IP for host-networked services (HA, pfSense). See Gotcha #55 in Runbook.

## History

- Stack file fixed and moved `stacks/monitoring/` → `stacks/` — PR #16 ✅ 2026-04-30
- `arr_arr_default` network + InfluxDB fix — PR `claude/exciting-brahmagupta-e8f897` ✅ 2026-05-08
- Migrated to Portainer Git integration ✅ 2026-05-08
- Fixed hostname (`uptime.home.purvishome.com` → `uptime-kuma.home.purvishome.com`), added `traefik.swarm.network=traefik-public` label, added internal-only middleware — PR #49 ✅ 2026-05-21
- Fixed Homepage widget slug to correctly reference the status page — PR #59 ✅ 2026-05-26
- Created public status page with slug `homelab` at `/status/homelab` ✅ 2026-05-26

## Related

- [[Docker Swarm Infrastructure Runbook]] — Gotcha #55
- [[VM - monitoring-01]]
