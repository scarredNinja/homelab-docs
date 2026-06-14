---
type: swarm-service
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
tags:
  - DockerSwarm
  - Service
  - Media
service_name: Seerr
vm: worker-mediamanagement-01
swarm_constraint: node.labels.zone == mediamanagement
vlan: 50
service_status: running
stack_file: /mnt/docker-swarm/stacks/arr/stack-arr.yml
port: 5055
external_access: true
traefik_entrypoint: websecure
url_internal: 'https://seerr.home.purvishome.com'
zfs_dataset: rpool/docker-data/seerr
mount_path: /mnt/docker-data/seerr
last_updated: '2026-05-29T00:00:00.000Z'
url_external: 'https://seerr.purvishome.com'
status: Completed
---

# Seerr

Request management UI for Plex, integrating with Sonarr and Radarr. Rebranded successor of Jellyseerr/Overseerr.
Runs on Node.js/Next.js. Config database must stay on local storage (virtiofs), never NFS.
Pinned to `worker-mediamanagement-01` via `node.labels.zone == mediamanagement`.

## Notes

* **Traefik Route:** Exposed at `https://seerr.home.purvishome.com` with TLS (Cloudflare certresolver).
* **Config Storage:** Placed in local ZFS dataset mount `/mnt/docker-data/seerr/config` (mounted to `/app/config` in the container) to prevent SQLite corruption over network filesystems.
* **Networks:**
  - `arr_default` for Sonarr/Radarr API communication.
  - `media` for Plex media server communication.
  - `traefik-public` for reverse proxy access.
* **Status:** **Planning** (Added service definition, pending stack redeployment and initial UI wizard setup).

## Configuration Guidelines & Profile Setups

Detailed Guide: [[Quality Profiles & Recyclarr Setup]]

### 1. Quality & Size Profiles (Sonarr/Radarr)
To ensure optimal performance and storage efficiency, configure your Sonarr and Radarr settings as follows:
* **Release Exclusions:** Add release profile restrictions (Settings → Profiles → Release Profiles in Sonarr, Settings → Custom Formats in Radarr) to explicitly block the following terms:
  - `CAM`, `TELESYNC`, `TS`, `TELECINE`, `TC`, `SCREENER`, `SCR`, `WORKPRINT`.
* **Size Limits:** Set maximum file-size-per-hour limits to prevent huge, bloated releases that can saturate network connections or cause Plex server CPU transcoding lag during playback.
* **Unified Root Folders:** All requests route to unified directories (`/tv`, `/movies`, `/animation`) without separating 4K and 1080p onto different physical links or paths.
* **Freeleech Prioritization:** Use indexers with freeleech tags enabled where possible, and customize release profiles to score releases containing `freeleech` or preferred tracker terms higher, preserving your seeding ratio.

### 2. Seerr Approvals
* **Default Settings:** Guest user requests will require manual approval to ensure strict quality, size, and format filtering.
* **Admin Bypass:** Administrator requests are set to auto-approve.
