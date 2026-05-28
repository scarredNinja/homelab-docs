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
service_status: running
stack_file: /mnt/docker-swarm/stacks/traefik/stack.yml
port: 443
external_access: true
traefik_entrypoint: none
note: "Traefik is the proxy itself — traefik.enable=false means it does not register as a Traefik backend and has no router/entrypoints label. external_access=true because it publishes ports 80/443 in mode:host on the DMZ VLAN and terminates public HTTPS traffic from Cloudflare. traefik_entrypoint=none reflects the absence of label-based routing for this service record."
url_internal: https://traefik.home.purvishome.com
zfs_dataset: rpool/docker-data/traefik
mount_path: /mnt/docker-data/traefik
last_updated: 2026-05-26
---

# Traefik

Reverse proxy. Single instance on traefik-dmz-01.
- Single `websecure` entrypoint on `:443` handles all services
- `web` entrypoint on `:80` redirects to `websecure`
- `metrics` entrypoint on `:8080` for Prometheus scrape only (not published)
- Internal vs public separation is enforced by the internal-only IP allowlist middleware, not by a separate entrypoint


## Pre-deploy checklist

- [x] `docker network create --driver overlay --attachable traefik-public` ✅ 2026-04-08
- [x] `install -m 600 /dev/null /mnt/docker-data/traefik/data/acme.json` ✅ 2026-04-08
- [x] CF API token written to `/mnt/docker-data/traefik/cf_api_token.txt` chmod 600 ✅ 2026-04-08
- [x] `docker node update --label-add zone=public traefik-dmz-01` ✅ 2026-04-08
- [x] HTTPS confirmed working end-to-end ✅ 2026-04-21

## Session 2026-04-21 — HTTPS Fix

> [!bug] Bug #21 — Asymmetric routing (resolved temporarily)
> Dual default routes at metric 100 caused ECMP splitting of SYN-ACK replies.
> Temp fix applied (route deleted). Permanent netplan fix outstanding on `enp6s19`.
> See [[Docker Swarm Infrastructure Runbook]] Gotcha #31.

> [!important] Overlay subnets pinned — 10.200.x.0/24
> All stacks redeployed after subnet collision with physical VLANs.
> `traefik-public` → `10.200.2.0/24` · `traefik_traefik-backend` → `10.200.3.0/24`
> See [[Docker Swarm Infrastructure Runbook]] Gotcha #32.

- [ ] Permanent asymmetric routing fix — netplan `use-routes: false` on `enp6s19` [priority:: 1]

## History

- `acme.json` permissions corrected (755→600) on Proxmox host; SSL restored site-wide ✅ 2026-05-26 — See [[Docker Swarm Infrastructure Runbook]] Gotcha #50

## Related

- [[Traefik Routing Architecture]]
