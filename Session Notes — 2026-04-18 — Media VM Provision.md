---
date: 2026-04-18
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
session_type: Deploy + Debug
status: Partial — VLAN 100 blocking media NFS
tags:
  - SessionNotes
  - DockerSwarm
  - Networking
  - Traefik
  - Media
  - Migration
---

# Session Notes — 2026-04-18

## Objective

Unblock `worker-media-01` VLAN 100 / NFS issue, provision `worker-mediamanagement-01`, begin data migration from old LXC containers, and fix Traefik cert resolver.

---

## Key Decisions & Learnings

### VLAN 100 / NFS Architecture Confirmed

- `vmbr1` and `eno2.100` were never created — VLAN 100 runs via `vmbr0` with `tag=100`. Correct, since `vmbr0` is VLAN-aware (`bridge-vids 2-4094`). No dedicated bridge needed.
- Root cause of eth1 no-IP: VLAN 100 was not tagged on Extreme switch port 37 (Proxmox LAG). Added and confirmed.
- pfSense VLAN 100 interface already existed (NAS scope). DHCP works but must not push a gateway — VLAN 100 is storage-only.
- `dhclient eth1` on the VM overwrote the default route via eth0, killing SSH. Netplan fix required.
- **Fix**: `dhcp4-overrides: use-routes: false, use-dns: false` in `/etc/netplan/51-eth1-nas.yaml`. This is now automated in `03-post-boot.sh`.

### worker-media-01 Nuked and Reprovisioned

- Original VM (VMID 203) destroyed after SSH loss from route overwrite.
- Reprovisioned cleanly with fixed scripts from code session on branch `claude/focused-feynman-ea3eba`.
- Four bugs fixed in provisioning scripts during code session (see gotchas #25–28 below).

### worker-mediamanagement-01 Provisioned

- Provisioned with `--vlan 50 --nic2-vlan 100 --disk2-size 500 --disk2-mount /mnt/downloads --label zone=mediamanagement`.
- All verification checks passed: single default route, eth1 `10.0.100.x`, NAS reachable, downloads disk mounted.
- Label `zone=mediamanagement` confirmed.

### Data Migration — Old LXC Containers

- Old data lives in LXC containers on `MainStorage` pool (not imported by default).
- Imported safely: `zpool import -f MainStorage`. Pool was in use by old system — force import overwrites hostid stamp. Must `zpool export MainStorage` before rebooting to old system.
- Data found via filesystem scan (not `pct list` — containers not registered in this Proxmox instance):

|Service|Source Path|
|---|---|
|Plex (61G)|`subvol-101/home/docker/plex/`|
|Tautulli|`subvol-101/home/docker/tautulli/`|
|Sonarr|`subvol-104/home/docker/sonarr/`|
|Radarr|`subvol-104/home/docker/radarr/`|
|Prowlarr|`subvol-102/home/ha/docker/`|
|Transmission|`subvol-102/home/ha/docker/transmission/`|

- rsync started for Plex + Tautulli + Sonarr + Radarr. Prowlarr + Transmission paths still to confirm.
- All old services confirmed off before rsync — no corruption risk.

### Plex External Access

- Cloudflare CNAME `plex.purvishome.com` → `purvishome.com` created, proxy disabled (grey cloud / DNS only).
- Existing pfSense DDNS maintains WAN IP — no separate DDNS entry needed.
- Dual Traefik router pattern planned in `stack-plex.yml`: `plex-internal` (internal entrypoint) + `plex-external` (websecure).
- Post-deploy action: Plex Settings → Remote Access → Custom server access URLs → `https://plex.purvishome.com:443`.

### Traefik cert resolver debugging

- Traefik `traefik.yaml` had multiple encoding issues: corrupt binary bytes, UTF-8 em-dash in comments, tab indentation, wrong `propagation.delayBeforeChecks` structure.
- The correct Traefik v3 syntax is `delayBeforeCheck: "10s"` directly under `dnsChallenge` (not wrapped in `propagation:`).
- File was rewritten multiple times via heredoc. Final version written on `traefik-dmz-01` directly (virtiofs share is per-VM — writes on Proxmox host and the VM go to the same dataset but confusion arose about which node was active).
- Cert resolver still not loading at session end — confirmed file content is correct, investigating whether container is reading the updated file.
- Root cause of repeated encoding issues: editing YAML in a terminal/chat context introduces non-ASCII characters. Always write via heredoc with `'EOF'` (single-quoted) to prevent substitution.

### Stack Files Built (code session — branch `claude/recursing-nightingale-4e74b2`)

- `stack-plex.yml` — Plex + Tautulli, dual Traefik routers, NFS read-only media mount.
- `stack-arr.yml` — Sonarr/Radarr/Prowlarr/Gluetun+Transmission, NFS read-write, downloads local disk.
- `traefik.yaml` — fixed, `metrics` entrypoint on `:8082` added.
- `stack-traefik.yml` — `monitoring` overlay network added for Prometheus scraping.
- `prometheus.yml` — Traefik scrape job updated to port 8082.

---


## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Session Notes — 2026-04-12 — Monitoring Deploy]]