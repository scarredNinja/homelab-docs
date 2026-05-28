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

service_status: running
stack_file: /mnt/docker-swarm/stacks/controller/stack-controller.yml
port: 8123
external_access: false
traefik_entrypoint: websecure
url_internal: https://homeassistant.home.purvishome.com

zfs_dataset: rpool/docker-data/homeassistant
mount_path: /mnt/docker-data/homeassistant

last_updated: 2026-04-28
---

# Home Assistant

Smart home automation. Needs VLAN 20 (IoT) access via secondary NIC on controller worker.
Single-instance — must never float to another node.

## Notes

- Run `ha core stop` before container shutdown — SQLite corruption risk on unclean stop
- Secondary NIC on VLAN 20 for direct IoT device reach
- Traefik file-provider route: `homeassistant.yml` in `dynamic/` → `http://10.0.60.42:8123`
- Config migrated from CT102 via rsync ✅ 2026-04-24
- Port 8123 published in `mode: host` — `network_mode: host` is silently ignored in Swarm (Gotcha #45) ✅ 2026-04-28
- `trusted_proxies` added to `configuration.yaml` covering `172.0.0.0/8` + `10.0.60.0/24` — required for Traefik proxying (Gotcha #46) ✅ 2026-04-28
- Prometheus scrape job active at 60s interval — bearer token stored as Docker secret `ha_prometheus_token` (Gotcha #47) ✅ 2026-04-28
- Redback solar integration working — upstream API log noise only (missing `ActiveExportedPowerInstantaneouskW` field)
- UI accessible at `homeassistant.home.purvishome.com` ✅ 2026-04-28

## Open Items

- [ ] Allow Home VLAN access — (1) pfSense rule: VLAN 10 (`10.0.10.0/25`) → `10.0.60.42:8123`; (2) add `10.0.10.0/25` to Traefik `internal-only` middleware allowlist in `middlewares.yaml`
- [ ] Disable analytics telemetry — Settings → System → Analytics (reduces Pi-hole ECONNREFUSED log noise)
- [ ] Check HACS for Redback integration update fixing `ActiveExportedPowerInstantaneouskW` KeyError
