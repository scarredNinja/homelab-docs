---
type: project
project_id: Homelab-2025
status: Research complete — build pending
tags:
  - Dashboard
  - Homepage
  - DockerSwarm
  - Monitoring
last_updated: 2026-05-07
---

# Home Dashboard — Research & Build Plan

> [!abstract] Status
> Research complete. **Homepage** selected. Build pending — deploy Uptime Kuma first, then proceed with stack.

---

## Requirements

| Requirement | Detail |
|---|---|
| Service links | All services grouped by category |
| Service status | Live status from Uptime Kuma |
| Arr widgets | Queue size, upcoming for Sonarr/Radarr/Prowlarr |
| Plex widget | Now playing, stream count |
| Grafana | Links to specific dashboards |
| Monitoring | Proxmox, UniFi, Portainer widgets |
| Config | YAML in repo — not database-backed |
| Deployment | Swarm stack, `zone=monitoring`, Traefik `internal` entrypoint |

---

## Options Evaluated (2026-05-07)

| Option | Uptime Kuma | Arr widgets | Config | Swarm | Decision |
|---|---|---|---|---|---|
| **Homepage** | ✅ Native | ✅ Full | ✅ YAML | ✅ Native | ✅ **Selected** |
| **Homarr** | ✅ Native | ✅ Full | ❌ DB/UI | ⚠️ Limited docs | ❌ |
| **Dashy** | ⚠️ Workaround | ✅ Status only | ⚠️ Hybrid | ✅ Compatible | ❌ |
| **Glance** | ❌ | ❌ | ✅ YAML | — | ❌ Wrong tool |

**Homepage** wins on: native Docker Swarm label-based discovery, strongest widget ecosystem for this stack, pure YAML config, largest community (29.9k stars, v1.12.3 active).

---

## On Notifications

Discord webhooks are outbound-only — no dashboard can receive from Discord. The practical equivalent:

- **Uptime Kuma** covers this: shows live status on the dashboard widget AND sends alerts to Discord simultaneously. One source, two outputs.
- **Alertmanager** (Phase 8 backlog) will eventually cover Prometheus alert rules → Discord + RSS feed.
- Reframe: "notifications" = Uptime Kuma status widget on dashboard + Discord for push alerts.

---

## Service List (for config)

**Media**
- Plex — `https://plex.purvishome.com`
- Tautulli — `https://tautulli.home.purvishome.com`
- Sonarr — `https://sonarr.home.purvishome.com`
- Radarr — `https://radarr.home.purvishome.com`
- Prowlarr — `https://prowlarr.home.purvishome.com`
- Transmission — `https://transmission.home.purvishome.com`

**Monitoring**
- Grafana — `https://grafana.home.purvishome.com`
- Prometheus — `https://prometheus.home.purvishome.com`
- Uptime Kuma — `https://uptime-kuma.home.purvishome.com`

**Management**
- Portainer — `https://portainer.home.purvishome.com`
- Traefik — `https://traefik.home.purvishome.com:8443`

**Home / Network**
- Home Assistant — `https://homeassistant.home.purvishome.com`
- UniFi — `https://unifi.home.purvishome.com`
- pfSense — internal IP direct link
- Pi-hole — internal IP direct link

**Infrastructure**
- Proxmox — internal IP direct link

---

## Grafana Dashboard Links (quick access)

| Dashboard | ID | Notes |
|---|---|---|
| Node Exporter Full | 1860 | Per-node host metrics |
| Traefik v3 | 17346 | Request rates, errors |
| Proxmox Cluster | 15356 | Host-level metrics via InfluxDB |
| Loki Container Logs | 13639 | Container log explorer |
| UniFi Poller UAP | 11311 | AP metrics |
| Backup Status | custom | `backup-status.json` — PR #18 |

---

## Build Plan

> [!important] Deploy Uptime Kuma first
> Homepage's Uptime Kuma widget connects to the Uptime Kuma API. Deploy and configure Uptime Kuma before building the dashboard so status widgets are live on first deploy.

### Step 1 — Deploy Uptime Kuma (prerequisite)
- Pre-deploy checklist: create `/mnt/docker-data/uptime-kuma` directory on Proxmox host
- Deploy `stack-uptime-kuma.yml` via Portainer (merged PR #16, pending deploy)
- Configure monitors for all services
- Verify `https://uptime-kuma.home.purvishome.com` loads

### Step 2 — Code session: build Homepage stack
Create in repo:
- `proxmox-swarm/stacks/dashboard/stack-homepage.yml` — Swarm stack with Traefik labels, `zone=monitoring` placement, manager node socket for label discovery
- `proxmox-swarm/stacks/dashboard/config/settings.yaml` — title, theme, layout
- `proxmox-swarm/stacks/dashboard/config/services.yaml` — full service list grouped by category (see above)
- `proxmox-swarm/stacks/dashboard/config/widgets.yaml` — Uptime Kuma, Plex, Sonarr, Radarr, Prowlarr, Proxmox, UniFi
- `proxmox-swarm/stacks/dashboard/config/bookmarks.yaml` — Grafana dashboard quick links

### Step 3 — Deploy and configure
- Copy config to Proxmox host virtiofs share (or use Portainer Git integration)
- Deploy stack via Portainer
- Verify all widgets connect and show live data
- Add Traefik dynamic config entry if needed

### Step 4 — Runbook update
- Add Homepage to Appendix C stack reference table
- Add `https://dashboard.home.purvishome.com` to infrastructure routes

---

## Notes

- Homepage requires access to the Docker socket for label-based service discovery — either mount `/var/run/docker.sock` directly or use `tecnativa/docker-socket-proxy` (already in use for Traefik). Use the socket proxy pattern for consistency and security.
- Homepage on Swarm: pin to a manager node (`node.role == manager`) for Docker socket access, or use `zone=monitoring` if socket proxy is used instead.
- `HOMEPAGE_FILE_*` env vars for any API keys/tokens — use Docker secrets pattern consistent with other stacks.

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]] — Appendix C (stack reference)
- [[Session Notes — 2026-05-07 — Backup Infrastructure Complete]]
- [[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]]
