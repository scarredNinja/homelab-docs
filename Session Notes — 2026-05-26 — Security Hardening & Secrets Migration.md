---
project_id: Homelab-2025
date: '2026-05-26T00:00:00.000Z'
session_type: Planning + Deploy
status: Completed
tags:
  - homelab
  - security
  - docker-swarm
  - session-note
  - syncthing
  - dns
  - dev-node-01
  - newtonfit
phase: 'Phase 5: Docker Swarm'
---

# Session Notes — 2026-05-26 — Security Hardening & Secrets Migration

## Session 1: Security Hardening & Secrets Migration

### Completed

- **PR #61 — Security hardening: image pins, secrets migration, restic auth** (`d244624`)
  - Traefik log level DEBUG → INFO; Prometheus scrape/eval interval 15s → 30s
  - Prometheus TSDB `--storage.tsdb.max-bytes=8GB` added to cap disk usage
  - Traefik compress middleware added to dynamic config
  - All `:latest` image tags pinned to stable versions: `traefik:v3.2`, `influxdb:2.7`, `portainer/portainer-ce:2.21.0`, `restic/rest-server:0.13.0`, `syncthing/syncthing:1.27`, `jacobalberty/unifi:8.6.9`, `golift/unifi-poller:v2.11`, `cloudflare/cloudflared:2025.1.0`, `onedr0p/exportarr:v2.0`, `tecnativa/docker-socket-proxy:0.2`
  - Grafana admin credentials migrated from env vars to Docker secrets (`GF_SECURITY_ADMIN_USER__FILE` / `GF_SECURITY_ADMIN_PASSWORD__FILE`)
  - NetBox `DB_PASSWORD` + `SECRET_KEY` migrated to Docker secrets via shell wrapper + `POSTGRES_PASSWORD_FILE`
  - `cloudflared` `--token` CLI arg replaced with `TUNNEL_TOKEN` env var sourced from Docker secret
  - Restic REST server `--no-auth` replaced with `--htpasswd-file` backed by Docker secret
  - Swarm state dir `SWARM_STATE_DIR` chmod 700; worker token removed from provision logs

- **PR #59 — Fix uptime-kuma widget slug** (`f65e134`)
  - Homepage `widgets.yaml` uptime-kuma slug corrected to `homelab`

- **Pi-hole wildcard DNS** — added `*.home.purvishome.com → 10.0.60.40` as a local DNS record in Pi-hole UI; removed redundant duplicate A record entries. Investigated dev-node-01 DNS: already resolving via `10.0.60.20`/`10.0.60.21` through DHCP — no fix needed.

- **Grafana backup-status dashboard** — re-imported backup-status v5 dashboard via Grafana UI from `configs/monitoring/dashboards/backup-status.json`

- **acme.json SSL fix** — diagnosed `ERR_SSL_UNRECOGNIZED_NAME_ALERT` site-wide; root cause was `acme.json` permissions `755` (Traefik requires `600`, silently skips ACME resolver otherwise). Fixed with `chmod 600 /mnt/docker-data/traefik/data/acme.json` on Proxmox host; restarted Traefik via `docker service update --force traefik_traefik` from `manager-01`. SSL restored site-wide.

- **Uptime Kuma status page** — created public status page with slug `homelab` at `uptime-kuma.home.purvishome.com/status/homelab`; added monitor groups: Server (Proxmox, Portainer, NAS, Home Assistant, Controller, etc.), Media, Metrics

---

## Session 2: Base Setup & Sandbox Migration

### Session Goal
Consolidate homelab notes to map development environments, formulate a prioritized development roadmap, establish the base development networking layer (Syncthing and Windows Defender firewall whitelisting), and pivot the development sandboxes to run isolated on `dev-node-01` rather than production nodes.

### Completed
- **Ecosystem Audit conducted:** Analyzed Next.js Finance App (Akahu API & Akut AI billing heuristic queries), .NET 9 distroless Native AOT sandbox, NewtonFit wellness aggregator, and `dev-node-01` isolated virtualization VM via a swarm of specialized background agents.
- **Firewall exceptions added:** Whitelisted Syncthing ports TCP `22000` and UDP `21027` on the Windows Workstation to resolve cross-VLAN synchronization blocks.
- **Vault sync verified:** Confirmed that your entire 190-file Obsidian vault (`10 - Projects`) successfully synced flat to `/mnt/docker-data/vault/` on `dev-node-01`.
- **DNS routing checked:** Verified systemd-resolved on `dev-node-01` VM is fully healthy, pointing to local Pi-holes (`10.0.60.20` & `10.0.60.21`), and correctly routing local hosts to Traefik.
- **NewtonFit Sandbox isolated:** Created `/mnt/docker-data/newtonfit-dev` directory on the dev node VM, seeded the baseline `fitness_history.json` database, and set ownership PUID/PGID permissions to `1000:1000`.
- **Created `stack-dev-newtonfit.yml`:** Created the declarative sandbox Swarm stack configuration file under `proxmox-swarm/stacks/` to host NewtonFit exclusively on the dev node VM (`node.labels.zone == dev`).
- **Obsidian checklists updated:** Appended the sandbox migration and production stack teardown tasks to both the Personal Fitness Phase 1 Hub and the Homelab Next Session checklists.

---

## Next Session

- **Homepage kuma widget "Missingkuma" (top priority)** — PR #60 (`fix/obsidian-mcp-vault-uid` branch) switches the Uptime Kuma widget URL from the Traefik hostname to internal Docker service URL (`http://uptime-kuma:3001`) to bypass the `internal-only@file` middleware that blocks API access. Steps after merge:
  1. Merge PR #60
  2. Redeploy homepage stack via Portainer
  3. If widget still broken: `docker service update --force homepage_homepage`
  4. Verify API response: `wget -qO- http://uptime-kuma:3001/api/status-page/homelab` from within the homepage container

- **NewtonFit Sandbox Deployment (VLAN 40)**
  1. Deploy the new sandbox stack `dev-newtonfit` via Portainer using `stack-dev-newtonfit.yml`.
  2. Tear down the existing production `newtonfit` stack on `worker-monitoring-01` to prevent data collision.
  3. Set up mobile client-side Syncthing to replicate Samsung Health JSON files to the dev VM.
  4. Test CSV/JSON automated watchers and verify glassmorphic trend chart rendering.
  5. Debug the Pi-hole dnsmasq wildcard domain overrides showing Tailscale placeholder responses.

## Open Tasks

- [x] Homepage dashboard — verify Traefik routing to `dashboard.home.purvishome.com`; Pi-hole DNS entry and router label still unresolved ✅ 2026-06-01
- [x] Verify arr stack `_v2` Docker secrets deployed to Swarm and `stack-arr.yml` redeployed with API key secrets ✅ 2026-06-01
- [x] Homepage kuma widget shows "Missingkuma" — fixed 2026-05-27 (`54db0ab`); switched to internal URL `http://uptime-kuma:3001` ✅
- [x] Update Homepage `widgets.yaml` kuma slug from `homelab` to the actual public status page slug [priority:: 2] #Later ✅ 2026-06-01
- [x] **unpoller password** — `up.conf` bind-mounted with plaintext UniFi password; migrate to Docker secret ✅ 2026-06-01
- [ ] **Plex** — no Traefik auth middleware; internal VLAN devices can reach Plex unauthenticated at the proxy layer
- [x] Alert rule: backup job staleness — Prometheus alert if backup textfile metric not updated in >26h ✅ 2026-06-01
- [x] Home Assistant Prometheus integration enabled [priority:: 3] ✅ 2026-06-01
- [x] Deploy node_exporter on pihole1/pihole2; add scrape targets to `prometheus.yml` [priority:: 3] #Later ✅ 2026-06-01
- [x] Migrate testing sandbox for NewtonFit to dev-node-01 (`stack-dev-newtonfit.yml`) and verify health and sync [priority:: 1] #NewtonFit ✅ 2026-06-01
- [x] Clear/remove the existing production `newtonfit` stack from `worker-monitoring-01` until sandbox testing completes [priority:: 2] #NewtonFit ✅ 2026-06-01

## Related Notes

- [[01 Homelab Rebuild - Phase 3 Network Config Hub]]
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
- [[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]]
- [[Homelab Next Phase — Review & Roadmap]]
- [[Service - grafana]]
- [[VM - dev-node-01]]
- [[Service - newtonfit]]
- [[Service - obsidian-mcp-server]]
