---
type: swarm-vm
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - VM

vmid:
hostname: worker-media-01
vcpu: 4
ram_gb: 8
disk_gb: 20

vlan_primary: 50
vlan_secondary: 100
ip_primary:
ip_secondary:

swarm_role: worker
node_labels:
  - "node.labels.type=media"
  - "node.labels.zone=private"

vm_status: not-created
post_boot_run: false
swarm_joined: false
last_updated: 2026-04-02
---
- [ ] Needs to be updated  [priority:: 2]
# worker-media-01

Media stack worker. Runs Plex, Sonarr, Radarr, Transmission, Prowlarr, Tautulli.
All media services are pinned here — SQLite databases must not float to other nodes.

## Notes

- Transmission runs behind Gluetun VPN sidecar
- Media files on NAS via NFS — config/DB on virtiofs docker-data
- Plex transcode cache should be local ZFS zvol, not virtiofs

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Swarm Topology]]
