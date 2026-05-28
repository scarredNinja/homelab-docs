---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
  - Media

service_name: Plex
vm: worker-media-01
swarm_constraint: "node.labels.type == media"
vlan: 50

service_status: running
stack_file: /mnt/docker-swarm/stacks/media/stack.yml
port: 32400
external_access: true
traefik_entrypoint: websecure
url_internal: https://plex.home.purvishome.com
url_external: https://plex.purvishome.com

zfs_dataset: rpool/docker-data/plex
mount_path: /mnt/docker-data/plex

last_updated: 2026-05-11
---

# Plex

Media server. Only service that requires `websecure` entrypoint — remote access needs inbound internet.

## Notes

- DNS fixed to bypass Traefik, transcode settings optimised, cpu=host pending reboot — 2026-05-11
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
- **`cloudflared` sidecar token (`CLOUDFLARE_TUNNEL_TOKEN`)** is persisted in **Portainer stack env vars** (Stacks → plex → Environment variables), NOT operator shell. Compose interpolation reads from the deploying shell — unset = silent empty token = container exit 255. See Gotcha #59. ✅ 2026-05-08
- **`cloudflare/cloudflared:latest` is distroless** — no `/bin/sh`, no debug tools. Entrypoint shims (`sh -c '...'`) won't work; only `cloudflared` itself runnable with flags/env vars. Docker-secret + shim approach attempted in PRs [#26](https://github.com/scarredNinja/docker-swarm-home/pull/26)/[#27](https://github.com/scarredNinja/docker-swarm-home/pull/27), reverted in [#28](https://github.com/scarredNinja/docker-swarm-home/pull/28).

## External Access Status

> [!note] External access in progress — 2026-04-21
> - ACME cert valid (15 certs confirmed in acme.json) ✅
> - Cloudflare DNS: `plex CNAME → ddns.purvishome.com` (DNS only mode) ✅
> - `Resolve-DnsName plex.purvishome.com -Server 1.1.1.1` resolving ✅
> - pfSense port forward: external 443 → traefik-dmz-01 — **needs verification**
> - PiHole cached negative — flush or wait for TTL expiry
> - **Browser test end-to-end: next session**
