---
external_access: true
last_updated: '2026-06-02'
mount_path: /mnt/docker-data/traefik
note: >-
  Traefik is the proxy itself — traefik.enable=false means it does not register
  as a Traefik backend and has no router/entrypoints label. external_access=true
  because it publishes ports 80/443 in mode:host on the DMZ VLAN and terminates
  public HTTPS traffic from Cloudflare. traefik_entrypoint=none reflects the
  absence of label-based routing for this service record.
phase: 'Phase 5: Docker Swarm'
port: 443
project_id: Homelab-2025
service_name: Traefik
service_status: running
stack_file: /mnt/docker-swarm/stacks/traefik/stack.yml
swarm_constraint: node.labels.zone == public
tags:
  - DockerSwarm
  - Service
  - Networking
traefik_entrypoint: none
type: swarm-service
url_internal: 'https://traefik.home.purvishome.com'
vlan: 80
vm: traefik-dmz-01
zfs_dataset: rpool/docker-data/traefik
status: Completed
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

> [!success] Bug #21 — Asymmetric routing (permanently resolved)
> Dual default routes at metric 100 caused ECMP splitting of SYN-ACK replies.
> Permanent netplan fix implemented by adding `dhcp4-overrides: use-routes: false` to the `enp6s19` interface stanza. Outbound traffic now routes consistently via the primary management interface (`eth0`, VLAN 60).

> [!important] Overlay subnets pinned — 10.200.x.0/24
> All stacks redeployed after subnet collision with physical VLANs.
> `traefik-public` → `10.200.2.0/24` · `traefik_traefik-backend` → `10.200.3.0/24`
> See [[Docker Swarm Infrastructure Runbook]] Gotcha #32.

- [x] Asymmetric routing netplan fix consolidated — tracked on master VM page [[VM - traefik-dmz-01]] ✅ 2026-06-01

## History

- `acme.json` permissions corrected (755→600) on Proxmox host; SSL restored site-wide ✅ 2026-05-26 — See [[Docker Swarm Infrastructure Runbook]] Gotcha #50

## Related

- [[Traefik Routing Architecture]]
