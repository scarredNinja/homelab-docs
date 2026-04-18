---
date: 2026-04-07
session_type: Implementation
branch: claude/youthful-shaw
tags:
  - traefik
  - docker
  - swarm
  - debugging
---

# Session Notes — 2026-04-07
## Traefik v3 Config Consolidation + Docker Daemon Fixes

---

## What We Did

### 1. Traefik v3 Config — Full Rewrite

Audited all uploaded Traefik config files from the repo. Found multiple critical issues across every file. Consolidated and fixed into `proxmox-swarm/stacks/traefik/`.

#### Files changed

| File | What changed |
|---|---|
| `stack-traefik.yml` | Ports 80/443/8443 `mode: host`, `cf_api_token` as external Docker secret, no dashboard labels, restart `on-failure` with 5s delay |
| `traefik.yaml` | Fixed `providers.swarm.network: proxy` → `traefik-public`, added `web:80` → HTTPS redirect, added `forwardedHeaders.trustedIPs` with full Cloudflare IP list, added `internal` entrypoint on `:8443`, fixed `acme.json` path |
| `dynamic/middlewares.yaml` | Removed VLAN 80 (DMZ) from `internal-only` allowlist, fixed VLAN 60 CIDR to `/25`, merged `middlewares.yml` headers + pihole-redirect with correct domain (`purvishome.com` not `example.com`) |
| `dynamic/infrastructure.yml` | New consolidated file — all 6 routes (pihole, pfsense, proxmox, extreme, pdu, portainer) with correct `websecure` entrypoint, `internal-only` middleware on all, fixed pfSense IP to `10.0.60.1`, fixed YAML structure (`routers:` + `services:` keys) |
| `dynamic/tls.yaml` | TLS 1.2 minimum, `sniStrict: true`, 6 modern cipher suites only |

#### Files deleted (11 total)
- `dynamic/dynamic.yml` — consolidated into `infrastructure.yml`
- `dynamic/middlewares.yml` — merged into `middlewares.yaml`
- `dynamic/pfsense.yml` — broken YAML structure, consolidated
- `dynamic/pihole.yml` — broken YAML structure, consolidated
- `dynamic/portainer.yml` — broken YAML structure + wrong middlewares, consolidated
- `dynamic/proxmox.yml` — broken YAML structure, consolidated

#### Key bugs found in original files
- `providers.swarm.network: proxy` — wrong network name (should be `traefik-public`)
- All per-service files missing `routers:` and `services:` keys under `http:` — invalid YAML structure, silently ignored by Traefik
- All routes using entrypoint `https` — name didn't match defined entrypoint `websecure`, no routes loaded
- No `internal-only` middleware on any infra routes — Proxmox, pfSense, pihole all publicly reachable
- VLAN 80 (DMZ) in `internal-only` allowlist — defeated DMZ isolation
- `portainer.yml` had `pihole-headers` and `pihole-redirect` middlewares — copy-paste error
- `pihole-redirect` regex had `home.example.com` placeholder — redirect never fired
- No `forwardedHeaders.trustedIPs` — Cloudflare IPs would be blocked by `internal-only` middleware

---

### 2. `03-post-boot.sh` — Four Bugs Fixed

#### Bug 1 — Docker fails to start with fuse-overlayfs
- **Symptom:** `dockerd` exits with status 1 immediately after install
- **Root cause:** `fuse` kernel module not loaded before Docker attempts to use `fuse-overlayfs` storage driver
- **Fix:** Added `modprobe fuse` before `systemctl restart docker`

#### Bug 2 — Docker bridge conflict (`networks have same bridge name`)
- **Symptom:** Docker crash-loops with `error creating default "bridge" network: cannot create network ... networks have same bridge name`
- **Root cause:** Stale network ID in `/mnt/docker-data/network/` conflicted with `docker0` and `docker_gwbridge` bridges left in the kernel from a prior partial run
- **Fix:** Stop Docker + socket, `ip link delete docker0`, `ip link delete docker_gwbridge`, `rm -rf /mnt/docker-data/network/` before restarting
- **Note:** `/mnt/docker-data` is the virtiofs-backed `data-root` — network state files persist across reboots here, unlike a normal `/var/lib/docker` install

#### Bug 3 — `set -e` swallowing journalctl output on failure
- **Symptom:** Script exits silently when Docker restart fails, no diagnostic output
- **Root cause:** `systemctl restart docker` returns non-zero → `set -e` exits before journalctl dump runs
- **Fix:** Wrapped restart in `if ! systemctl restart docker; then journalctl -xeu docker.service --no-pager | tail -50; exit 1; fi`

#### Bug 4 — Swarm init not idempotent on re-runs
- **Symptom:** Script sees `LocalNodeState: "pending"` and tries to re-init an already-initialising Swarm
- **Root cause:** `LocalNodeState` returns `"pending"` for ~15s after Docker restarts while the Swarm overlay network re-establishes
- **Fix:** Poll for stable state (up to 45s) after Docker restart; treat `"pending"` as already-in-swarm in `init_swarm` and `handle_swarm` functions

---

### 3. Docker Daemon Debugging — manager-01

Live debugging session on the actual VM revealed the bridge conflict issue. Steps taken:

```bash
# What failed
sudo systemctl stop docker docker.socket
sudo ip link set docker0 down && sudo ip link delete docker0
sudo ip link set docker_gwbridge down && sudo ip link delete docker_gwbridge
sudo rm -rf /var/lib/docker/network/   # NOTE: on this VM data-root is /mnt/docker-data
sudo systemctl start docker
```

Eventually resolved by nuking both VMs and reprovisioning with the fixed `03-post-boot.sh`.

---

### 4. Swarm Bootstrap — Manual Steps Required Post-Reprovision

After `manager-01` came up clean, Swarm was initialised manually (script timing issue with `pending` state). Tokens need to be saved:

```bash
# On manager-01 — save tokens to virtiofs-backed shared location
mkdir -p /mnt/docker-swarm/swarm
docker swarm join-token worker -q > /mnt/docker-swarm/swarm/worker_token.txt
docker swarm join-token manager -q > /mnt/docker-swarm/swarm/manager_token.txt
chmod 600 /mnt/docker-swarm/swarm/*.txt
```

These are read by `02-provision-vm.sh` when provisioning subsequent workers.

---

## Current State (end of session)

| Item | Status |
|---|---|
| `manager-01` | Running at `10.0.60.100`, Docker healthy, fuse-overlayfs confirmed, Swarm initialised |
| Swarm tokens | Need saving to `/mnt/docker-swarm/swarm/` |
| `traefik-public` overlay network | Needs creating before Portainer deploy |
| Portainer stack | Not yet deployed — next step |
| `traefik-dmz-01` | Not yet reprovisioned |
| Traefik config files | Fixed and committed to `claude/youthful-shaw` |

---

## Next Steps (next session)

1. Save swarm tokens to `/mnt/docker-swarm/swarm/`
2. Create `traefik-public` overlay network on `manager-01`
3. Deploy Portainer stack (Appendix C)
4. Verify Portainer at `https://10.0.60.100:9443`
5. Reprovision `traefik-dmz-01` with fixed `03-post-boot.sh`
6. Deploy Traefik stack via Portainer
7. Update pfSense static mapping for `manager-01` — current IP `10.0.60.100` vs intended `10.0.60.30`

---

## Gotchas Discovered This Session

| Symptom | Cause | Fix |
|---|---|---|
| Docker bridge conflict after reprovision | `data-root` on virtiofs preserves stale network state across VM rebuilds | Delete `/mnt/docker-data/network/` + kernel bridges before Docker restart |
| `docker_gwbridge` survives VM destruction | Swarm overlay bridge persists in virtiofs-backed data-root | Always clean both `docker0` and `docker_gwbridge` on fresh deploy |
| pfSense static mapping shows `(Incomplete)` | VM booted before mapping was saved/propagated | Verify IP with `ip addr` after boot before running script 3; use `networkctl renew eth0` to re-request |
| `networkctl renew` doesn't get static IP | pfSense still has dynamic lease recorded; renew extends existing lease | Delete dynamic lease in pfSense → Status → DHCP Leases, then renew |
| Swarm `LocalNodeState: pending` causes re-init | ~15s propagation delay after Docker restart | Poll for stable state before acting on Swarm status |
