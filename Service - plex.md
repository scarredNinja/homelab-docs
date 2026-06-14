---
type: swarm-service
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
tags:
  - DockerSwarm
  - Service
  - Media
service_name: Plex
vm: worker-media-01
swarm_constraint: node.labels.type == media
vlan: 50
service_status: running
stack_file: /mnt/docker-swarm/stacks/media/stack.yml
port: 32400
external_access: true
traefik_entrypoint: websecure
url_internal: 'https://plex.home.purvishome.com'
url_external: 'https://plex.purvishome.com'
zfs_dataset: rpool/docker-data/plex
mount_path: /mnt/docker-data/plex
last_updated: '2026-06-14'
status: In Progress
---
# Plex

Media server. Only service that requires `websecure` entrypoint — remote access needs inbound internet.

## Notes

- DNS pointed to Traefik (`10.0.60.40`) to restore standard HTTPS domain access; direct LAN streaming preserved by adding `http://10.0.50.50:32400` to Custom Server Access URLs in Plex settings (bypassing Traefik/pfSense hops for local clients) — 2026-06-01
- Transitioned external access from Cloudflare Tunnel to direct WAN port forwarding to Traefik DMZ IP (`10.0.80.20:443`) via a purchased static IP to bypass CGNAT — 2026-06-14
- Config/metadata on virtiofs docker-data — SQLite must not go on NFS
- Transcode cache on local ZFS zvol for IOPS
- Media files on NAS via NFS mount
- `PLEX_UID=1500`, `PLEX_GID=1500`
- Stack deployed, libraries intact (config rsync'd from old server) ✅ 2026-04-19
- **Media mounts must bind each NFS share directly under `/media/<share>`, NOT bind the parent `/mnt/media`** — host uses autofs direct-mount triggers; Docker's default `rprivate` bind propagation captures parent trigger directories but NOT the underlying NFS mounts, so container sees empty `/media/*`. Fixed in PR [#30](https://github.com/scarredNinja/docker-swarm-home/pull/30) ✅ 2026-05-08. See Gotcha #60.
  ```yaml
  - /mnt/media/Movies:/media/Movies:ro
  - /mnt/media/TVShows:/media/TVShows:ro
  - /mnt/media/Animation:/media/Animation:ro
  ```
- **`cloudflared` sidecar tunnel removed** to bypass bandwidth throttling and enable direct-stream capability. ✅ 2026-06-14

## External Access Status

> [!warning] Transitioning to Direct WAN Access (Static IP) — 2026-06-14
> - Cloudflare Tunnel removed to avoid free-tier media throttling.
> - Cloudflare DNS for `plex.purvishome.com` set to **DNS-Only** (grey-clouded) CNAME to `ddns.purvishome.com`.
> - pfSense WAN NAT port forward configured to map port 443 TCP to Traefik's DMZ interface (`10.0.80.20:443`).
> - Currently waiting for ISP static IP activation to bypass CGNAT (`100.71.224.X`) and verify end-to-end connectivity.

## Open Items

- [ ] Plex Traefik Security — currently has no Traefik authentication middleware; add auth rules so internal VLAN devices cannot bypass proxy layer security
