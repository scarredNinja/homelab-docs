---
date: '2026-04-08T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
session_type: Deploy
status: Completed
branch: main
tags:
  - SessionNotes
  - Traefik
  - Docker
  - DockerSwarm
  - Debugging
---

# Session Notes — 2026-04-08 — Traefik Deployment + pfSense DynDNS

## Summary

Corrected Docker Swarm provisioning issues on first manager and worker. Debugged orphaned Docker bridge networks causing daemon startup failures. Got Traefik deployed in Portainer via git repo integration. Hit Docker API version issue and Swarm socket issue on worker node — both resolved. DNS config in progress. Monitoring deployment is next.

---

## Infrastructure Changes

### Provisioning Script Fixes (`02-provision-vm.sh`, `03-post-boot.sh`)

From merged branches `claude/mystifying-greider` and `claude/youthful-shaw`:

**`02-provision-vm.sh`:**
- VM name now accepted as bare positional arg
- Secondary NICs (`--nic2-vlan` / `--nic3-vlan`) write `ipconfig1`/`ipconfig2` to cloud-init on first boot
- Executable bit set on all scripts
- `--label key=value` flag added — applied via `docker node update` after join

**`03-post-boot.sh`:**
- First manager: force-leaves stale `pending` Swarm state before re-init
- Worker join: validates correct swarm before skipping
- `verify_provisioning`: polls properly for Docker `active` state; fixed datasource check
- Post-swarm-init: polls up to 45s for `LocalNodeState == active`
- Systemd Docker unit: `After=/Requires=` on virtiofs mount (fixes boot race)
- `ExecStartPre` drop-in: flushes `docker0`/`docker_gwbridge` before every Docker start
- `configure_secondary_nics()`: finds interface by MAC, writes persistent netplan DHCP config
- Portainer stack auto-redeployed after each worker joins

---

## Traefik Stack Deployment

### Architecture

**Stack file:** `proxmox-swarm/stacks/traefik/stack-traefik.yml`

Key decisions:
- `traefik:latest` — Docker 29.4 rejects API 1.24 used by older pinned images (Bug 16)
- `mode: host` on ports 80/443/8443 — preserves real Cloudflare source IPs through Swarm
- `tecnativa/docker-socket-proxy` on manager — `traefik-dmz-01` is a worker and can't query Swarm API from its local socket; proxy exposes read-only API over `traefik-backend` overlay (Bug 17)
- `traefik-backend` overlay — connects socket proxy (manager) to Traefik (DMZ worker)
- `traefik-public` overlay — external; services attach to be discovered

**Entrypoints:**
- `:80` → redirects to `websecure`
- `:443` → `websecure` (Cloudflare trustedIPs configured)
- `:8443` → `internal` (VLAN 60 management access only)

### Bugs Fixed During Deployment

| Bug | Symptom | Fix |
|---|---|---|
| Docker 29.4 + old Traefik | Traefik rejects API 1.24 | Switch to `traefik:latest` |
| `EntryPoint doesn't exist: https` | Dynamic configs using old entrypoint name | Rename all to `websecure` in dynamic configs |
| Worker can't list Swarm services | Traefik on worker can't see services | Add `docker-socket-proxy` on manager, set endpoint to `tcp://docker-proxy:2375` |
| Live `traefik.yml` stale | Wrong endpoint/network/acme path | `sed` patches applied to live file |

### Dynamic Config Consolidation

All infrastructure routes merged into `traefik/dynamic/infrastructure.yml`:
- Routes: pihole, pfsense, proxmox, extreme switch, PDU, portainer
- `insecureSkipVerify: true` transports for self-signed certs (pfsense, proxmox)
- Individual stubs deleted
- All stacks moved to `proxmox-swarm/stacks/`
- `CLAUDE.md` created documenting repo conventions

---

## pfSense DynDNS — Cloudflare Fix

**Error:** `Could not route to /client/v4/zones/token/dns_records, perhaps your object identifier is invalid?`

**Root cause:** pfSense CE 2.8.0 substitutes the Username field literally into the Cloudflare API URL path instead of resolving the Zone ID from the domain name.

**Fix:** Use Zone ID as the Username field.

| Field | Value |
|---|---|
| Username | Zone ID (from Cloudflare Dashboard → domain → Overview → right sidebar) |
| Password | API token (Zone:DNS:Edit permission) |
| Hostname | `ddns` |
| Domain | `purvishome.com` |

**Status:** Working ✅

---

## Infrastructure State at Session End

| VM | Status |
|---|---|
| `manager-01` | Running, Swarm Leader, Portainer deployed |
| `traefik-dmz-01` | Running, Swarm worker, Traefik deployed |
| `worker-media-01` | Not provisioned |
| `worker-controller-01` | Not provisioned |
| `worker-general-01` | Not provisioned |

| Service | Status |
|---|---|
| Portainer | ✅ Running on manager-01 |
| Traefik | ✅ Deployed on traefik-dmz-01, pending DNS verification |
| Monitoring | ❌ Not yet deployed |
| pfSense DynDNS | ✅ Working |

---

## Outstanding Items

1. **Traefik DNS verification** — confirm `traefik.home.purvishome.com` dashboard loads and internal services (pihole, pfsense, proxmox) are routing correctly
2. **DMZ isolation test** — verify `traefik-dmz-01` cannot reach VLAN 60 directly
3. **External HTTPS test** — `curl -I https://purvishome.com` from mobile data, expect `CF-Ray` header
4. **Monitoring stack** — deploy Prometheus + Grafana + node-exporter + cAdvisor on `manager-01` (see Phase 1.5 in runbook)
5. **Remaining VMs** — provision `worker-media-01`, `worker-controller-01`, `worker-general-01` after monitoring is stable
6. **Open PR to main** — branch is ahead with no open PR

---

## Next Session Starting Point

1. Verify Traefik dashboard and internal service routing
2. Run external HTTPS test
3. Deploy monitoring stack (runbook Phase 1.5)
4. Provision remaining worker VMs (runbook Phase 2)
