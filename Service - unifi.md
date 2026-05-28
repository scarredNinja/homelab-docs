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

service_status: running
stack_file: /mnt/docker-swarm/stacks/controller/stack-controller.yml
port: 8443
external_access: false
traefik_entrypoint: websecure
url_internal: https://unifi.home.purvishome.com

zfs_dataset: rpool/docker-db/unifi
mount_path: /mnt/docker-db/unifi

last_updated: 2026-04-28
---

# UniFi Controller

WiFi and AP management. Uses bundled MongoDB (`unifi-controller` image) — local virtiofs docker-db, never NFS.
Needs VLAN 20 reach for device adoption.
Pinned to controller worker — device adoption state is local.

## Notes

- Using `lscr.io/linuxserver/unifi-controller` (bundled MongoDB). Migration to `unifi-network-application` deferred — see Gotcha #38.
- Config migrated from CT110 to `/mnt/docker-db/unifi/` ✅ 2026-04-24
- chown 1000:1000 applied post-rsync (Gotcha #36)
- Traefik labels fixed 2026-04-28: added `internal-only` middleware + `insecureTransport@file` serversTransport (port 8443 self-signed cert)
- UI accessible at `unifi.home.purvishome.com` ✅ 2026-04-28

## Open Items

- [ ] Re-adopt AP to new controller — set inform URL to `http://10.0.60.42:8080/inform` (SSH or device reset)
- [ ] Verify Prometheus metrics — confirm scrape job active and data visible in Grafana
