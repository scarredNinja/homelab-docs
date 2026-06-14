---
type: swarm-vm
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
tags:
  - DockerSwarm
  - VM
  - Media
vmid: 204
hostname: worker-mediamanagement-01
vcpu: 6
ram_gb: 6
disk_gb: 20
vlan_primary: 50
vlan_secondary: 100
ip_primary: 10.0.50.51
ip_secondary: 10.0.100.41
swarm_role: worker
node_labels:
  - node.labels.zone=mediamanagement
  - node.labels.nas=nfs
vm_status: running
post_boot_run: true
swarm_joined: true
last_updated: '2026-05-21T00:00:00.000Z'
status: updated
---

# worker-mediamanagement-01

Swarm worker node. Media management stack — Seerr, Sonarr, Radarr, Prowlarr, Transmission + Gluetun VPN sidecar.

## Notes

- Dual-homed: VLAN 50 (Media) + VLAN 100 (Storage/NFS)
- NFS read-write mount to Synology NAS at 10.0.100.20 via VLAN 100 — `/mnt/media`
- Downloads disk: 500G local ZFS at `/mnt/downloads`
- VLAN 100 `eth1` using `dhcp4-overrides: use-routes: false` to prevent gateway override
- Transmission + Gluetun run as **Docker Compose** (`compose-vpn.yml`), NOT Swarm — `network_mode: service:X` unsupported in Swarm (Gotcha #37)
- VPN: NordVPN WireGuard (NordLynx) — successfully migrated from OpenVPN to resolve CPU scaling bottleneck and UI latency ✅ 2026-05-17
- `arr_arr_default` overlay is attachable — compose containers join overlay for Traefik routing ✅ 2026-04-25
- Arr stack deployed ✅ 2026-04-19; Gluetun/Transmission removed from Swarm stack ✅ 2026-04-25
- `compose-vpn` (gluetun + transmission) auto-starts at boot via robust `compose-vpn.service` systemd unit (enhanced with `docker rm -f gluetun` hook to prevent startup name conflicts) ✅ 2026-05-20. See Gotcha #58.
- vCPU bumped 4 → 6 ✅ 2026-05-08 — Radarr/Sonarr UI unresponsive during high-throughput sessions. Root cause: gluetun's OpenVPN was single-threaded and pinned a vCPU. Migration to NordLynx WireGuard on 2026-05-17 permanently resolved this CPU bottleneck, restoring swift UI responsiveness.
- Exportarr sidecars running on ports `9707` (Sonarr) and `9708` (Radarr) to export Prometheus metrics. Configured using prefix-free env vars (`URL`, `PORT`, `API_KEY_FILE`) and clean, newline-free Swarm secrets (`sonarr_apikey_v2` and `radarr_apikey_v2` created via `printf`) to bypass structural and regex validation bugs in the Go binary. Scraped successfully at `10.0.50.51:9707` and `10.0.50.51:9708` (UP and verified ✅ 2026-05-20).

## Resolved Issues

- [x] **Sonarr not binding on port 8989** — Resolved. Sonarr is successfully binding and serving requests through Traefik ✅ 2026-04-25.
- [x] **Sonarr → Transmission connection failing** — Resolved. Configured Sonarr's download client host parameter to point to the attachable overlay `gluetun` service name, enabling clean local routing within the `arr_arr_default` network ✅ 2026-04-25.

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Service - seerr]]
- [[Service - sonarr]]
- [[Service - radarr]]
- [[Service - prowlarr]]
- [[Service - transmission]]
