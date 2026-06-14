---
date: '2026-05-25'
phase: 'Phase 7: Backup Verification'
project_id: Homelab-2025
status: Reference
tags:
  - homelab
  - roadmap
  - planning
  - DockerSwarm
  - infrastructure
---

# Homelab Next Phase — Review & Roadmap
**Path:** Homelab Next Phase — Review & Roadmap.md
**Modified:** 2026-06-01T23:52:34.435Z
**Tags:** Automation, DockerSwarm, NewtonFit, homelab, infrastructure, planning, roadmap
**Frontmatter:**
  - date: "2026-05-25"
  - phase: 'Phase 7: Backup Verification'
  - project_id: "Homelab-2025"
  - status: "reference"
  - tags: ["homelab","roadmap","planning","DockerSwarm","infrastructure"]

---

# Homelab Next Phase — Review & Roadmap

> Full homelab state review + next phase planning. Generated 2026-05-25.
> Reference: [[Swarm Topology]] · [[Docker Swarm Infrastructure Runbook]] · [[Master Component Inventory List]]

---

## 📋 Easy Wins (Do These First)

> [!tip] These are all low-effort, high-value. Each under 30 minutes.

| #   | Item                                                      | Action                                                                                                                                                                                                         | Effort    |
| --- | --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| 1   | ~~**Pi-hole wildcard DNS**~~ ✅                           | ~~Add `address=/.home.purvishome.com/10.0.60.40` to `/etc/dnsmasq.d/02-homelab-wildcard.conf` — fixes homepage, alertmanager, and all future subdomains in one shot~~ | ✅ Done    |
| 2   | ~~**Pi-hole DNS fix on dev-node-01**~~ ✅                 | ~~Change upstream from `100.100.100.1` upstream to `10.0.60.40` (wrong Tailscale IP set at provision time)~~ Verified pointing to local DNS | ✅ Done    |
| 3   | **Uptime Kuma status page**                               | Create public status page with a slug, update `widgets.yaml` kuma slug in Homepage config                                                                                                                      | 20 min    |
| 4   | **Grafana backup-status v5 re-import**                    | Re-import `configs/monitoring/dashboards/backup-status.json` (may be running v4)                                                                                                                               | 10 min    |
| 5   | ~~**Fix `(( idx++ ))` bug in 02-provision-vm.sh:376**~~ ✅ | ~~Change to `idx=$((idx+1))` — exits silently under `set -e` when idx=0~~ Fixed in [PR #57](https://github.com/scarredNinja/docker-swarm-home/pull/57) — all `(( ++ ))` patterns replaced with safe arithmetic | ✅ Done    |
| 6   | ~~**Syncthing peering**~~ ✅                               | ~~Complete Windows client pairing on dev-node-01~~ Peered 2026-05-25                                                                                                                                           | ✅ Done    |
| 7   | ~~**Alert rule: backup job staleness**~~ ✅                      | ~~Add Prometheus alert firing if backup textfile metric not updated in >26h~~ | ✅ Done    |
| 8   | **UniFi 404 diagnostics**                                 | `docker service logs traefik_traefik \| grep -i unifi` on traefik-dmz-01                                                                                                                                       | 30 min    |
| 9   | **Tailscale ERR_ADDRESS_UNREACHABLE**                     | Ping `10.0.60.1` from phone; check IP forwarding + pfSense firewall logs                                                                                                                                       | 30-60 min |

---

## Current Stack Snapshot (2026-05-25)

### Hardware

| Asset | Detail |
|---|---|
| **Primary server** | HPE DL360p Gen8 — 2 Intel Xeon, 128 GB RAM, 1U rack |
| **Storage** | LD01 RAID 0 (558 GiB), LD02 RAID 5 (1117 GiB), LD03 RAID 0 (838 GiB), LD04 RAID 5 999 GiB SSD |
| **NAS** | Synology ~16.2 TB, VLAN 100, 10.0.100.20 |
| **Switch** | Extreme X440-48p (48 1 Gbps, PoE, LACP capable) |
| **Firewall** | pfSense — edge routing, DHCP, VLAN segmentation |

### VM Roster

| VM | VLAN | Role | Status |
|---|---|---|---|
| manager-01 | 60 | Swarm manager (Leader), Portainer, NetBox | ✅ Running |
| manager-02 | 60 | Swarm manager (Reachable) | ✅ Running |
| manager-03 | 60 | Swarm manager (Reachable) | ✅ Running |
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
- [x] **Single Swarm manager** — ✅ Resolved 2026-05-25: manager-02 (210) + manager-03 (211) provisioned; 3-manager quorum active. Note: all three on one Proxmox host — physical HA deferred to Phase C.
- [ ] **LD01 + LD03 are RAID 0** — zero redundancy; single disk failure = data loss on those arrays.

### Security
- [ ] **unpoller password in plaintext** — `up.conf` bind-mounted with plain-text UniFi password; should use Docker secret
- [x] **restic-rest-server `--no-auth`** — replaced with `--htpasswd-file` Docker secret ✅ 2026-05-26 — PR #61
- [ ] **Plex has no Traefik auth middleware** — any internal VLAN device can reach the Plex endpoint unauthenticated at the proxy layer

### Architecture
- [ ] **Home Assistant in Swarm** — `network_mode: host` silently ignored in Swarm mode; mDNS/Bonjour IoT discovery broken. Should run as plain `docker compose` with host networking.
- [x] **NetBox on manager node** — stateful Postgres + Redis on the management plane. Move to worker once provisioned. [priority:: 1] [[NetBox & Compose-VPN Review Plan]] ✅ 2026-06-05
- [x] **compose-vpn is a Swarm blind spot** — systemd unit on worker-mediamanagement-01; not auto-restored by Swarm on reprovision. [priority:: 2] [[NetBox & Compose-VPN Review Plan]] ✅ 2026-06-05
- [x] **No backup staleness alerting** — backup textfile metrics exist but no alert fires on silence >26h. Alert rule configured in backups.yml ✅

---

## Active Roadmap

> [!tip] These are all doable with current hardware — no purchases required.

### ~~Phase A — Swarm Manager HA~~ ✅ Done 2026-05-25

manager-02 (VMID 210, 10.0.60.31) and manager-03 (VMID 211, 10.0.60.32) provisioned and joined. Swarm: Leader + 2 Reachable. `vm-backup.sh` refactored to registry-driven VM discovery ([PR #57](https://github.com/scarredNinja/docker-swarm-home/pull/57), [PR #58](https://github.com/scarredNinja/docker-swarm-home/pull/58)).

---

### ~~Phase B — Synology Dual NIC~~ ✅ Done 2026-05-29

Split media streaming and backup traffic onto separate physical links. Synology already has two NICs — just needs cabling and config.

| Option | Verdict |
|---|---|
| **LACP bonding** (802.3ad) | Easy, X440-48p supports it ✅, but doesn't truly separate traffic — just more bandwidth |
| **Dedicated IPs per traffic type** | Recommended — NIC 1 (VLAN 100, 10.0.100.20) for media; NIC 2 (VLAN 60, 10.0.60.45) for restic backups |
| **Multipath NFS** | Overkill for homelab |

**Steps (dedicated IP approach):**
- [x] Implement Synology NIC updates [priority::2] ✅ 2026-05-29
1. Plug Synology NIC 2 into switch port tagged VLAN 60
2. Assign static IP `10.0.60.45`
3. Update `stack-restic.yml` NFS volume to use `10.0.60.45`
4. Leave all media mounts (plex, arr) pointing to `10.0.100.20`
5. Add pfSense rule: VLAN 60 → `10.0.60.45` TCP/2049

---

### Phase C — Environment Segregation

- dev-node-01 on VLAN 40 ✅ correctly isolated
- [x] Add `dev-public` overlay (10.200.10.0/24) — dev stacks attach here, not `traefik-public` ✅ 2026-06-03
- [x] Second Traefik entrypoint (port 8444) with `*.dev.home.purvishome.com` wildcard ✅ 2026-06-03
- [ ] Restrict VLAN 40 → VLAN 60 to specific ports only in pfSense
- [ ] Migrate Home Assistant to plain `docker compose` with `network_mode: host` for mDNS
- [ ] Move NetBox off manager-01 to a worker node (move to general or redeploy)

---

### Phase D — Media Stack Additions

Software-only additions to the existing media stack — no new hardware required.

- [x] **Seerr** — request management UI, rebranded successor of Overseerr. Add to `stack-arr.yml` Then look at profile setups ✅ 2026-06-03
- [ ] **Bazarr** — subtitle automation, integrates with Sonarr/Radarr
- [ ] **Recyclarr** — syncs TRaSH Guide quality profiles to Sonarr/Radarr as a cron container

---

### 📦 Detached Advanced Projects

To prevent scope creep and maintain absolute focus on core Swarm stability, advanced services have been migrated into their own independent, detached project boards:

- 🛠️ **Developer Toolchain & CI/CD Platform:** Handled in [[06 Self-Hosted Developer Platform — Master Hub]]
- 🧠 **Advanced Applications & ML Platforms:** Handled in [[07 Advanced Application Hosting — Master Hub]]

---

## Future State — Hardware Required

> [!note] Revisit when second physical host or additional hardware is acquired. No action needed until then.

### Second Physical Host

| Option | Best for | Power | Est. Cost |
|---|---|---|---|
| **HPE DL360p/DL380p Gen9** | Second full Proxmox node — iLO, NVMe, DDR4, rack-mount | ~150-200W idle | $200-500 eBay |
| **Dell PowerEdge R730** | GPU passthrough, max RAM, PCIe expansion | ~180-250W idle | $300-600 eBay |
| **N100 mini PC × 3** (Beelink EQ12) | Additional Swarm workers, dev nodes — silent, low power | ~10-15W each | $150-200 each |
| **Raspberry Pi 5** | Corosync QDevice, Pi-hole hardware, network probes | ~5W | $80 |

**Recommended path:**
1. **Short term:** 1× N100 mini PC → move manager-02 or -03 to bare-metal for physical Swarm quorum split (~$150)
2. **Medium term:** HPE Gen9 or Dell R730 as second full Proxmox node → form Proxmox cluster with Pi QDevice

**Proxmox Cluster notes:** Use ZFS send/receive for dataset replication. Pi 4 or N100 mini PC as corosync QDevice.

| | Extended Swarm | Proxmox Cluster |
|---|---|---|
| **HA level** | Container/service (~30-60s reschedule) | VM level (~2-5 min auto-restart on surviving host) |
| **Requires** | Join VMs from second host to Swarm | Shared storage OR ZFS replication + corosync + QDevice |
| **Complexity** | Low | Medium-high |
| **Right for now?** | ✅ Yes — do this first | Not yet |

---

### Hardware Transcoding

DL360p Gen8 has no iGPU. Options when ready:
- Low-profile **NVIDIA T400/P400** in PCIe slot → NVENC passthrough to Plex/Jellyfin container
- **N100 mini PC** as dedicated media node → built-in Intel QuickSync at ~15W (cheaper, more efficient)

---

## Redundancy Summary

| Area | Current | Target |
|---|---|---|
| Physical compute | 1 host (full SPOF) | 2 hosts + Pi QDevice |
| Swarm managers | ~~1~~ **3** (all on single host) ✅ | 3 split across physical hosts (Phase C) |
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

- [x] Migrate testing sandbox for NewtonFit to dev-node-01 (`stack-dev-newtonfit.yml`) and verify health and sync [priority:: 1] #NewtonFit ✅ 2026-05-28
- [x] Clear/remove the existing production `newtonfit` stack from `worker-monitoring-01` until sandbox testing completes [priority:: 2] #NewtonFit ✅ 2026-05-28
- [x] Add obsidian-mcp-server integration to **Google Antigravity** project ✅ 2026-05-25
- [x] Implement Git post-commit hooks triggering `session-note-writer` subagent to automate future session note-taking [priority:: 1] #Automation ✅ 2026-05-28
- [x] Configure webhook-triggered vault sync to run `vault-sync` subagent automatically on Git pushes [priority:: 2] #Automation ✅ 2026-05-28
