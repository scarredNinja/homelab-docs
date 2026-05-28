---
date: 2026-04-25
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
session_type: Diagnostic + Fix
status: Complete
tags:
  - SessionNotes
  - DockerSwarm
  - Portainer
  - Traefik
  - Transmission
  - Gluetun
  - VPN
  - Overlay
---

# Session Notes — 2026-04-25 — Portainer Overlay Fix & Transmission VPN Split

> [!note] This note is frozen. For current cluster state see [[Swarm Topology]].

## 🎯 Session Goals

1. Fix Portainer agent overlay connectivity — agents not reaching manager after recent reboot
2. Resolve Transmission/Gluetun VPN setup — `network_mode: service:gluetun` not supported in Swarm (Bug #26)
3. Fix Traefik file-provider routes (YAML parse error blocking all file-based routes)

---

## ✅ Completed

### Session 1 — Portainer Overlay Network Fix

**Root cause:** Two compounding issues after `manager-01` reboot:
1. Stale VXLAN interfaces left on `manager-01` — ghost interfaces from the previous Swarm overlay that Docker didn't clean up
2. Stale `local-kv.db` on all nodes — overlay key-value state out of sync with the rebuilt network

**Fix — applied on manager-01:**
```bash
# List stale VXLAN interfaces
ip link show type vxlan

# Delete each stale interface
ip link delete <vxlan-interface>

# Restart Docker to rebuild overlay state
systemctl restart docker
```

**Fix — applied on all nodes:**
```bash
# Stop Docker, remove stale KV store, restart
systemctl stop docker
rm -f /var/lib/docker/network/files/local-kv.db
systemctl start docker
```

**After cleanup:**
```bash
# Force-update Portainer agent to trigger overlay re-registration
docker service update --force portainer_portainer_agent
```

Portainer overlay confirmed healthy. All agents visible in Portainer UI ✅.

> [!important] Overlay ping tests require nsenter
> `docker network inspect` only shows containers local to the queried node — this is misleading for non-attachable overlays. To verify overlay reachability between nodes, enter the network namespace on the manager:
> ```bash
> # Get short network ID (first 10 chars of network ID)
> docker network ls
> sudo nsenter --net=/var/run/docker/netns/1-<short-id> ping <overlay-ip>
> ```
> Do not ping overlay IPs from the host — they are not routable from the host network namespace.

**Inter-VM SSH configured:**
Generated `~/.ssh/id_ed25519` on `manager-01`, distributed public key to all workers. Passwordless SSH now works: `ssh ubuntu@worker-media-01`, `worker-mediamanagement-01`, etc.

---

### Session 2 — Transmission/Gluetun VPN Split + Traefik Fixes

#### Transmission + Gluetun moved to Docker Compose

`network_mode: service:gluetun` is silently ignored in Swarm mode (see Gotcha #37 and Bug #26). Swarm cannot share network namespaces between services.

**Solution:** Split Transmission + Gluetun out of the Swarm `arr` stack and run them as a plain Docker Compose stack on `worker-mediamanagement-01`.

| Before | After |
|--------|-------|
| `stack-arr.yml` included Gluetun + Transmission as Swarm services | `stack-arr.yml` contains only Sonarr, Radarr, Prowlarr |
| `network_mode: service:gluetun` on Transmission (silently ignored) | Transmission joins Gluetun network namespace correctly under Docker Compose |
| Compose VPN file: none | `/mnt/docker-swarm/stacks/arr/compose-vpn.yml` |

**arr_arr_default made attachable:**
The `arr_arr_default` overlay network was updated to `attachable: true` in `stack-arr.yml`. This allows the `compose-vpn` containers (running outside Swarm) to join the overlay and be reachable by Traefik for routing.

```yaml
# In stack-arr.yml
networks:
  arr_default:
    driver: overlay
    attachable: true
    ipam:
      config:
        - subnet: 10.200.5.0/24
```

#### Traefik file-provider route added for Transmission

Created `/mnt/docker-data/traefik/data/dynamic/vpn-transmission.yml` — Traefik file-provider route that forwards `transmission.home.purvishome.com` to the Gluetun container IP on the `arr_arr_default` overlay.

#### Traefik YAML parse error fixed (infrastructure.yml)

Missing colon on `homeassistant-service` line in `infrastructure.yml` — Traefik silently dropped all file-based routes when this file failed to parse. Fix: added missing colon.

```yaml
# Before (broken)
homeassistant-service
  loadBalancer:

# After (fixed)
homeassistant-service:
  loadBalancer:
```

All file-provider routes now loading ✅. Home Assistant, pfSense, Proxmox, UniFi routes all active.

#### Gluetun: Mullvad WireGuard → NordVPN OpenVPN

WireGuard key registration (`POST /v1/wireguard/keys`) blocked by Cloudflare — Mullvad's registration endpoint is behind Cloudflare, and requests from the Proxmox host IP were blocked.

Switch to NordVPN OpenVPN resolved the issue. Gluetun VPN tunnel confirmed active.

#### Sonarr/Radarr download client host updated

Changed host from `transmission` → `gluetun` in Sonarr/Radarr download client settings. This is required because Transmission's port 9091 is exposed via the Gluetun container's network namespace under Docker Compose.

#### Traefik acme.json permissions fixed

`acme.json` was `755` — Traefik requires `600`. Fix: `chmod 600 /mnt/docker-data/traefik/data/acme.json` on Proxmox host. Cert resolver Cloudflare now active ✅.

---

## 📊 Current State

> [!note] See [[Swarm Topology]] for the live dashboard — this section is a point-in-time snapshot.

| Service | VM | Status |
|---------|-----|--------|
| Portainer agent | all nodes | ✅ Overlay healthy |
| Traefik v3 | traefik-dmz-01 | ✅ Cert resolver active, file routes loading |
| arr (Sonarr/Radarr/Prowlarr) | worker-mediamanagement-01 | ✅ Swarm stack running |
| Transmission + Gluetun | worker-mediamanagement-01 | ✅ Docker Compose, NordVPN OpenVPN |
| Plex | worker-media-01 | ✅ Running |

**Still pending:**
- Sonarr/Radarr download client wiring (blocked: 440 GB transfer in progress)
- Home Assistant / UniFi container health verification
- Uptime Kuma deployment
- Grafana data source updates

---

## 🔗 Related Notes

- [[Swarm Topology]] ← current state
- [[Docker Swarm Infrastructure Runbook]] ← recovery procedures (Appendix G, H)
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
