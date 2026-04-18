---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - Networking
service_name: Traefik
vm: traefik-dmz-01
swarm_constraint: node.labels.zone == public
vlan: 80
service_status: deployed
stack_file: /mnt/docker-swarm/stacks/traefik/stack.yml
port: 443
external_access: true
traefik_entrypoint: websecure
url_internal: https://traefik.home.purvishome.com
zfs_dataset: rpool/docker-data/traefik
mount_path: /mnt/docker-data/traefik
last_updated: 2026-04-08
---

# Traefik

Reverse proxy. Single instance on traefik-dmz-01, two entrypoints on separate NICs.
`websecure` (:443) on VLAN 80 — Plex only. `internal` (:8443) on VLAN 60 — all other services.

## Pre-deploy checklist

- [x] `docker network create --driver overlay --attachable traefik-public` ✅ 2026-04-08
- [x] `install -m 600 /dev/null /mnt/docker-data/traefik/data/acme.json` ✅ 2026-04-08
- [x] CF API token written to `/mnt/docker-data/traefik/cf_api_token.txt` chmod 600 ✅ 2026-04-08
- [x] `docker node update --label-add zone=public traefik-dmz-01` ✅ 2026-04-08


### 08/04/26

Currently have a dns issue and need to test further
## Related

- [[Traefik Routing Architecture]]
