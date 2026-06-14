---
type: swarm-vm
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
tags:
  - DockerSwarm
  - VM
  - Media
vmid: 203
hostname: worker-media-01
vcpu: 8
ram_gb: 10
disk_gb: 20
vlan_primary: 50
vlan_secondary: 100
ip_primary: 10.0.50.50
ip_secondary: 10.0.100.40
swarm_role: worker
node_labels:
  - node.labels.type=media
vm_status: running
post_boot_run: true
swarm_joined: true
last_updated: '2026-05-21T00:00:00.000Z'
status: Active
---

# worker-media-01

Swarm worker node. Media playback stack — Plex + Tautulli.

## Notes

- Dual-homed: VLAN 50 (Media) + VLAN 100 (Storage/NFS)
- NFS read-only mount to Synology NAS at 10.0.100.20 via VLAN 100 — `/mnt/media/{Movies,TVShows,Animation}` via **autofs direct-mount triggers** (NOT a single parent NFS mount). Docker bind mounts MUST point at the share directly (`/mnt/media/Movies:/media/Movies`), not the parent — see Gotcha #60.
- Stale autofs trigger `/mnt/media/tvshows` (lowercase) with no backing NFS export — flagged for cleanup, not blocking anything.
- NFS performance tested: **128 MB/s** cold read ✅
- VLAN 100 `eth1` using `dhcp4-overrides: use-routes: false` to prevent default route override
- Transcode disk: 100G local ZFS zvol at `/mnt/transcode`
- Plex stack deployed, libraries intact (config rsync'd from old server) ✅ 2026-04-19
- Plex external access: DNS configured, browser test outstanding

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Service - plex]]
- [[Service - tautulli]]
- [[Swarm Topology]]
