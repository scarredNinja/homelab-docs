---
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
tags:
  - homelab
  - roadmap
  - planning
  - DockerSwarm
  - infrastructure
date: '2026-05-25'
status: reference
---
# Homelab Next Phase — Review & Roadmap

> Full homelab state review + next phase planning. Generated 2026-05-25.
> Reference: [[Swarm Topology]] · [[Docker Swarm Infrastructure Runbook]] · [[Master Component Inventory List]]

---

## ✅ Easy Wins (Do These First)

> [!tip] These are all low-effort, high-value. Each under 30 minutes.

| # | Item | Action | Effort |
|---|---|---|---|
| 1 | **Pi-hole wildcard DNS** | Add `address=/.home.purvishome.com/10.0.60.40` to `/etc/dnsmasq.d/02-homelab-wildcard.conf` — fixes homepage, alertmanager, and all future subdomains in one shot | 10 min |
| 2 | **Pi-hole DNS fix on dev-node-01** | Change upstream from `100.100.100.1` → `10.0.60.40` (wrong Tailscale IP set at provision time) | 5 min |
| 3 | **Uptime Kuma status page** | Create public status page with a slug, update `widgets.yaml` kuma slug in Homepage config | 20 min |
| 4 | **Grafana backup-status v5 re-import** | Re-import `configs/monitoring/dashboards/backup-status.json` (may be running v4) | 10 min |
| 5 | **Fix `(( idx++ ))` bug in 02-provision-vm.sh:376** | Change to `idx=$((idx+1))` — exits silently under `set -e` when idx=0 | 5 min |
| 6 | **Syncthing peering** | Complete Windows client pairing on dev-node-01 | 20 min |
| 7 | **Alert rule: backup job staleness** | Add Prometheus alert firing if backup textfile metric not updated in >26h | 30 min |
| 8 | **UniFi 404 diagnostics** | `docker service logs traefik_traefik \| grep -i unifi` on traefik-dmz-01 | 30 min |
| 9 | **Tailscale ERR_ADDRESS_UNREACHABLE** | Ping `10.0.60.1` from phone; check IP forwarding + pfSense firewall logs | 30-60 min |

---

## Current Stack Snapshot (2026-05-25)

### Hardware

| Asset | Detail |
|---|---|
| **Primary server** | HPE DL360p Gen8 — 2× Intel Xeon, 128 GB RAM, 1U rack |
| **Storage** | LD01 RAID 0 (558 GiB), LD02 RAID 5 (1117 GiB), LD03 RAID 0 (838 GiB), LD04 RAID 5 999 GiB SSD |
| **NAS** | Synology ~16.2 TB, VLAN 100, 10.0.100.20 |
| **Switch** | Extreme X440-48p (48× 1 Gbps, PoE, LACP capable) |
| **Firewall** | pfSense — edge routing, DHCP, VLAN segmentation |

### VM Roster

| VM | VLAN | Role | Status |
|---|---|---|---|
| manager-01 | 60 | Swarm manager, Portainer, NetBox | ✅ Running |
| traefik-dmz-01 | 60+80 | Reverse proxy, TLS, Cloudflare DNS-01 | ✅ Running |
| worker-monitoring-01 | 60 | Prometheus, Grafana, Loki, Alertmanager, restic | ✅ Running |
| worker-media-01 | 50 | Plex, Tautulli, cloudflared | ✅ Running |
| worker-mediamanagement-01 | 50 | Sonarr, Radarr, Prowlarr, Transmission+Gluetun | ✅ Running |
| worker-controller-01 | 60+20 | Home Assistant, UniFi, unpoller | ✅ Running |
| dev-node-01 | 40 | Syncthing, obsidian-mcp-server | ✅ Running |

All core stacks operational: `traefik` · `portainer` · `monitoring` · `plex` · `arr` · `compose-vpn` · `controller` · `restic` · `uptime-kuma` · `homepage` · `syncthing` · `obsidian-mcp` · `netbox` · `newtonfit`

---

## Known Flaws & Issues

### Critical (SPOFs)
- [ ] **Single Proxmox host** — total outage if DL360p dies. No physical failover.
- [ ] **Single Swarm manager** — manager-01 down = no new task scheduling. Need 3 managers for quorum.
- [ ] **LD01 + LD03 are RAID 0** — zero redundancy; single disk failure = data loss on those arrays.

### Security
- [ ] **unpoller password in plaintext** — `up.conf` bind-mounted with plain-text UniFi password; should use Docker secret
- [ ] **restic-rest-server `--no-auth`** — relies entirely on pfSense rules; no credential layer on backup data
- [ ] **Plex has no Traefik auth middleware** — any internal VLAN device can reach the Plex endpoint unauthenticated at the proxy layer

### Architecture
- [ ] **Home Assistant in Swarm** — `network_mode: host` silently ignored in Swarm mode; mDNS/Bonjour IoT discovery broken. Should run as plain `docker compose` with host networking.
- [ ] **NetBox on manager node** — stateful Postgres + Redis on the management plane. Move to worker once provisioned.
- [ ] **compose-vpn is a Swarm blind spot** — systemd unit on worker-mediamanagement-01; not auto-restored by Swarm on reprovision.
- [ ] **No backup staleness alerting** — backup textfile metrics exist but no alert fires on silence >26h.

---

## Next Phase Roadmap

### Phase A — Immediate (no new hardware)

#### Spawn manager-02 and manager-03

Gives 3-manager Swarm quorum. Protects against VM-level manager failure. Migrate one to a second physical host when it arrives for true physical HA.

```bash
# manager-02
./02-provision-vm.sh --name manager-02 --vmid-min 210 \
  --role manager --vlan 60 --memory 4096 --cores 2 --disk-size 20G

# manager-03
./02-provision-vm.sh --name manager-03 --vmid-min 211 \
  --role manager --vlan 60 --memory 4096 --cores 2 --disk-size 20G
```

> [!warning] Three managers on one host = Swarm HA, NOT physical HA. A host failure still takes all three down. This is a stepping stone.

#### Proxmox Cluster vs Extended Swarm

**Recommendation: Extended Swarm first, Proxmox Cluster later.**

| | Extended Swarm | Proxmox Cluster |
|---|---|---|
| **HA level** | Container/service (~30-60s reschedule) | VM level (~2-5 min auto-restart on surviving host) |
| **Requires** | Join VMs from second host to Swarm | Shared storage OR ZFS replication + corosync + QDevice |
| **Complexity** | Low | Medium-high |
| **Right for now?** | ✅ Yes | Not yet — revisit with second host |

When you form a Proxmox Cluster: use **ZFS send/receive** for dataset replication and a **Pi 4 or N100 mini PC** as the corosync QDevice.

---

### Phase B — Synology Dual NIC

Split media streaming and backup traffic onto separate physical links.

| Option | Verdict |
|---|---|
| **LACP bonding** (802.3ad) | Easy, X440-48p supports it ✅, but doesn't truly separate traffic — just more bandwidth |
| **Dedicated IPs per traffic type** | Recommended — NIC 1 (VLAN 100, 10.0.100.20) for media; NIC 2 (VLAN 60, 10.0.60.45) for restic backups |
| **Multipath NFS** | Overkill for homelab |

**Steps (dedicated IP approach):**
1. Plug Synology NIC 2 into switch port tagged VLAN 60
2. Assign static IP `10.0.60.45`
3. Update `stack-restic.yml` NFS volume to use `10.0.60.45`
4. Leave all media mounts (plex, arr) pointing to `10.0.100.20`
5. Add pfSense rule: VLAN 60 → `10.0.60.45` TCP/2049

---

### Phase C — Second Physical Host

| Option | Best for | Power | Est. Cost |
|---|---|---|---|
| **HPE DL360p/DL380p Gen9** | Second full Proxmox node — iLO, NVMe, DDR4, rack-mount | ~150-200W idle | $200-500 eBay |
| **Dell PowerEdge R730** | GPU passthrough, max RAM, PCIe expansion | ~180-250W idle | $300-600 eBay |
| **N100 mini PC × 3** (Beelink EQ12) | Additional Swarm workers, dev nodes — silent, low power | ~10-15W each | $150-200 each |
| **Raspberry Pi 5** | Corosync QDevice, Pi-hole hardware, network probes | ~5W | $80 |

**Recommended path:**
1. **Short term:** 1× N100 mini PC → run manager-02 or -03 as a bare-metal Docker node for physical Swarm quorum split (~$150)
2. **Medium term:** HPE Gen9 or Dell R730 as second full Proxmox node → form Proxmox cluster with Pi QDevice

---

### Phase D — Media Server Improvements

- [ ] **Hardware transcoding** — DL360p Gen8 has no iGPU. Options: low-profile NVIDIA T400/P400 in PCIe slot for NVENC passthrough; OR N100 mini PC media node with built-in QuickSync at ~15W
- [ ] **Jellyfin** — consider as Plex companion; no Plex Pass needed for HW transcoding, better subtitle/chapter support
- [ ] **Overseerr** — request management UI, integrates with Sonarr/Radarr. Add to `stack-arr.yml`
- [ ] **Bazarr** — subtitle automation, integrates with Sonarr/Radarr
- [ ] **Recyclarr** — syncs TRaSH Guide quality profiles to Sonarr/Radarr as a cron container

---

### Phase E — Environment Segregation

- dev-node-01 on VLAN 40 ✅ correctly isolated
- [ ] Add `dev-public` overlay (10.200.10.0/24) — dev stacks attach here, not `traefik-public`
- [ ] Second Traefik entrypoint (port 8444) with `*.dev.home.purvishome.com` wildcard
- [ ] Restrict VLAN 40 → VLAN 60 to specific ports only in pfSense
- [ ] Migrate Home Assistant to plain `docker compose` with `network_mode: host` for mDNS
- [ ] Move NetBox off manager-01 to a worker node

---

### Phase F — Dev Environment Expansion

| Service | Purpose | Priority |
|---|---|---|
| **Gitea / Forgejo** | Self-hosted Git — mirror GitHub repos, trigger CI | High |
| **Woodpecker CI** | Lightweight CI/CD — auto shellcheck/yamllint on every PR | High |
| **Harbor** | Self-hosted container registry — replace ghcr.io for custom images | Medium |
| **code-server** | Browser-based VS Code — code from any device on LAN/VPN | Medium |
| **n8n** | Workflow automation — Samsung Health, Plex events, Discord hooks | Medium |
| **Vaultwarden** | Self-hosted Bitwarden password manager | Medium |
| **Immich** | Self-hosted Google Photos — Syncthing backup path | Low |
| **MinIO** | S3-compatible object storage for dev + restic off-site target | Low |

---

## Redundancy Summary

| Area | Current | Target |
|---|---|---|
| Physical compute | 1 host (full SPOF) | 2 hosts + Pi QDevice |
| Swarm managers | 1 | 3 (split across physical hosts) |
| Storage LD01/LD03 | RAID 0 | Migrate critical data to LD02/LD04 |
| DNS | Pi-hole 1+2 + Gravity Sync ✅ | Done |
| Internet | Single ISP | 4G/5G failover via pfSense + USB LTE |
| Network switch | Single X440-48p | Second managed switch for uplink redundancy |
| Backup off-site | Restic → Synology ✅ | Add Backblaze B2 as second restic remote |

---

## Related Notes

- [[Swarm Topology]]
- [[Docker Swarm Infrastructure Runbook]]
- [[Docker Swarm Scaling Strategy]]
- [[Master Component Inventory List]]
- [[VLAN and Subnet Summary Sheet]]
- [[ZFS Configuration and Setup]]

---

## Next Session — Quick Pickup

- [ ] Add obsidian-mcp-server integration to **Google Antigravity** project