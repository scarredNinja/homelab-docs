---
date: 2026-04-28
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
session_type: debugging, infrastructure
status: complete
tags:
  - DockerSwarm
  - Traefik
  - HomeAssistant
  - UniFi
  - Monitoring
---

# Session Notes — 2026-04-28 — HA Routing Fix & Traefik Config Split

> [!abstract] Overview
> Two service issues resolved post-migration: Transmission deletes not removing files (UID mismatch from LXC migration), and Home Assistant returning bad gateway via Traefik. Traefik dynamic config refactored from a single monolith into per-service files. Prometheus scrape job added for HA with solar/Redback integration confirmed working. UniFi stack labels corrected.

---

## Session Goal

Resolve post-migration issues with Transmission, Home Assistant, and UniFi. Audit and split Traefik dynamic config into manageable per-service files.

---

## Completed

### 1. Transmission Delete Not Removing Files

**Root cause:** Downloaded files migrated from old LXC (UID 1001) via `mv` across ZFS datasets. Transmission container runs as UID 1000 — kernel rejects `unlink()` on files it doesn't own. Torrent record removed from Transmission DB but directory remained on disk.

**Fix:** `chown -R 1000:1000 /mnt/downloads/complete/`

Also identified and cleaned Sonarr/Radarr import staging dirs (`__XXXXXX` suffix, mode `700`):
```bash
find /mnt/downloads/complete/ -maxdepth 1 -type d -name '*__*' -exec rm -rf {} +
```

See Gotcha #36 — pre-existing UID mismatch pattern is covered there.

---

### 2. Home Assistant — Bad Gateway

**Root cause 1:** No file-provider route existed in Traefik `dynamic/` for HA. The `infrastructure.yml` entry had HA on the wrong entrypoint and with incorrect middleware (pihole-redirect applied to HA router — copy/paste error from original file). A temporary `homeassistant.yml` was created in `dynamic/` to restore routing immediately.

**Root cause 2:** `network_mode: host` is silently ignored in Docker Swarm (See Gotcha #45). HA was not actually on host networking — it was on the overlay — meaning port 8123 was not published. Fixed by adding port 8123 in `mode: host` to the service definition.

**Root cause 3:** HA returns HTTP 400 for requests proxied without trusted proxy config (See Gotcha #46). Fixed by adding `trusted_proxies` to `/mnt/docker-data/homeassistant/configuration.yaml`.

> [!bug] Gotcha #45 — `network_mode: host` silently ignored in Swarm
> See Appendix D. Fix: publish required ports explicitly in `mode: host`.

> [!bug] Gotcha #46 — HA returns 400 without trusted_proxies config
> See Appendix D. Fix: add `homeassistant.trusted_proxies` block to `configuration.yaml`.

---

### 3. Traefik Dynamic Config — Refactor (PR #13)

`infrastructure.yml` monolith split into per-service files. All files live in `/mnt/docker-data/traefik/data/dynamic/`.

| Old | New files |
|-----|-----------|
| `infrastructure.yml` (deleted) | `homeassistant.yml`, `portainer.yml`, `traefik-dashboard.yml`, `network-devices.yml` |
| `middlewares.yml` (deleted — stale duplicate) | `middlewares.yaml` (authoritative) |

> [!note] Entrypoint decision
> All routes remain on `websecure` entrypoint. The `internal` entrypoint was deliberately removed in PR #11 — internal vs external separation is enforced by the `internal-only` IP allowlist middleware instead of separate entrypoints. This is intentional.

> [!warning] `dynamic.yml` at root level is dead
> `/mnt/docker-data/traefik/data/dynamic.yml` is not mounted by `stack-traefik.yml` — only the `dynamic/` directory is mounted. The file is retained with a header comment to avoid confusion.

**`network-devices.yml`** covers: pihole, pfsense, proxmox, extreme switch, PDU — all static-IP devices with self-signed certs using `insecureTransport` serversTransport.

---

### 4. UniFi Stack Labels Fixed

`stack-controller.yml` deploy labels updated:
- Added `internal-only` middleware to UniFi router
- Added `insecureTransport@file` serversTransport (UniFi serves HTTPS with self-signed cert on port 8443)

Stack redeployed after label changes. UniFi controller UI is accessible via Traefik.

**Pending — AP re-adoption:** The access point has not yet re-adopted to the new controller. Needs to be informed of the new controller IP (`10.0.60.42:8080`) — either via SSH forget/re-adopt or by setting the inform URL manually on the device.

**Pending — UniFi metrics:** UniFi Prometheus metrics not yet verified. Confirm the SNMP or UniFi Poller scrape job is active in Prometheus and data is flowing to Grafana.

---

### 5. Prometheus — HA Scrape Job Added

Prometheus scrape job added for Home Assistant at 60s interval. Bearer token stored as Docker secret `ha_prometheus_token` — never committed to git.

> [!bug] Gotcha #47 — HA Prometheus bearer token must be a Docker secret
> See Appendix D.

Solar/Redback integration confirmed working post-migration. Redback custom integration has upstream API bugs (missing `ActiveExportedPowerInstantaneouskW` field, `NoneType` arithmetic) — not blocking, log noise only. Check HACS for updates.

---

### 6. HA Analytics Noise — Pi-hole Blocking

`analytics-api.home-assistant.io` is blocked by Pi-hole, causing `ECONNREFUSED` errors every hour in HA logs.

**Fix:** Settings → System → Analytics → disable all telemetry.

---

## Current State

| Service | Status |
|---------|--------|
| Home Assistant | ✅ Running — accessible at `homeassistant.home.purvishome.com` |
| UniFi | ⚠️ Accessible — AP not yet re-adopted, metrics not verified |
| Traefik dynamic config | ✅ Refactored — per-service files, monolith deleted |
| Prometheus HA scrape | ✅ Active — bearer token secured as Docker secret |
| Redback solar integration | ✅ Working — upstream API bug is cosmetic log noise |
| Transmission file deletion | ✅ Fixed — chown applied to migrated files |

---

## Next Session Priorities

- [x] Re-adopt UniFi AP to new controller — set inform URL to `http://10.0.60.42:8080/inform` (SSH or device reset)
- [x] Verify UniFi Prometheus metrics — confirm scrape job active and data visible in Grafana
- [x] Update stacks to deploy directly from Git repo (remove manual copy step) — see runbook task added this session ✅ 2026-05-08
- [x] Allow Home VLAN access to HA — (1) pfSense rule: VLAN 10 (`10.0.10.0/25`) → `10.0.60.42:8123`; (2) add `10.0.10.0/25` to `internal-only` middleware allowlist in `middlewares.yaml` ✅ 2026-05-08
- [x] Redback HACS update — check for version fixing `ActiveExportedPowerInstantaneouskW` KeyError ✅ 2026-05-08
- [x] `traefik-dmz-01` permanent asymmetric routing fix — `use-routes: false` in netplan (survives reboot) ✅ 2026-05-08
- [x] Disable HA analytics telemetry (Settings → System → Analytics) ✅ 2026-05-08

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]]
- [[Session Notes — 2026-04-27 — Monitoring Recovery, Arr Stack & Storage Cleanup]]
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
