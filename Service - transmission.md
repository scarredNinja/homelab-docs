---
type: swarm-service
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
tags:
  - DockerSwarm
  - Service
  - Media
service_name: Transmission
vm: worker-mediamanagement-01
swarm_constraint: node.labels.zone == mediamanagement
vlan: 50
service_status: running
stack_file: /mnt/docker-swarm/stacks/arr/compose-vpn.yml
port: 9091
external_access: false
traefik_entrypoint: websecure
url_internal: 'https://transmission.home.purvishome.com'
zfs_dataset: rpool/docker-data/transmission
mount_path: /mnt/docker-data/transmission
last_updated: '2026-05-08T00:00:00.000Z'
status: Completed
---

# Transmission

Torrent client. Runs behind Gluetun VPN sidecar as **Docker Compose** (not Swarm).
`network_mode: service:gluetun` is only supported in Docker Compose — silently ignored in Swarm (Gotcha #37).
Downloads to NAS NFS share. Config/resume state on virtiofs.

## Notes

- **Not a Swarm service** — deployed via `docker compose -f compose-vpn.yml up -d` on `worker-mediamanagement-01`
- `network_mode: service:gluetun` — Transmission shares Gluetun network namespace; port 9091 exposed via Gluetun container
- VPN: NordVPN OpenVPN (Mullvad WireGuard key registration blocked by Cloudflare, switched ✅ 2026-04-25)
- Traefik route via file-provider `vpn-transmission.yml` → `transmission.home.purvishome.com`
- Joins `arr_arr_default` overlay (attachable) so Traefik on `traefik-dmz-01` can reach it
- Sonarr/Radarr download client must point to `gluetun` (not `transmission`) as host — port 9091 is on the Gluetun namespace
- Do not put Prowlarr behind the VPN — needs direct tracker access
- **Auto-start at boot:** managed by `compose-vpn.service` systemd unit (not `restart: unless-stopped` — Swarm overlay gossip race, see Gotcha #58). Unit waits for `Swarm.LocalNodeState=active` + probe-attaches to `arr_arr_default` and `traefik-public` before `docker compose up -d` (5× retry). Compose project name pinned to `compose-vpn`. Unit + compose file copied via virtiofs by `copy-swarm-config.sh`; installed by `03-post-boot.sh install_compose_units` step. PR [#25](https://github.com/scarredNinja/docker-swarm-home/pull/25) ✅ 2026-05-08
- **NordVPN secrets:** `/etc/gluetun/vpn-secrets.env` requires *service credentials* (not account login) — nordvpn.com → Services → NordVPN → Set up manually → Service credentials. Format: `OPENVPN_USER=...` / `OPENVPN_PASSWORD=...`
- **Gluetun `UPDATER_PERIOD: 24h`** keeps NordVPN server list fresh — stale list surfaces as TLS handshake / EHOSTUNREACH errors. Transient EHOSTUNREACH during connect is normal (gluetun rotates servers automatically).
