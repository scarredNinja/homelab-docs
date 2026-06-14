---
type: swarm-vm
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
tags:
  - DockerSwarm
  - VM
vmid: 205
hostname: worker-controller-01
vcpu: 2
ram_gb: 8
disk_gb: 20
vlan_primary: 60
vlan_secondary: 20
ip_primary: 10.0.60.42
ip_secondary: null
swarm_role: worker
node_labels:
  - node.labels.zone=controller
  - node.labels.nas=nfs
vm_status: running
post_boot_run: true
swarm_joined: true
last_updated: '2026-05-21T00:00:00.000Z'
status: Active
---

# worker-controller-01

Controller worker. Dual-homed — VLAN 60 management, VLAN 20 IoT access.
Runs Home Assistant and UniFi. Both need direct IoT VLAN reach.

## Services

- Home Assistant (deploying) — Traefik file-provider route: homeassistant.home.purvishome.com → http://10.0.60.42:8123
- UniFi Controller (deploying) — using unifi-controller (bundled MongoDB). Migration to unifi-network-application deferred.

## Notes

- HA must stop cleanly before container shutdown — SQLite corruption risk
- UniFi uses bundled MongoDB in unifi-controller image — local virtiofs docker-db dataset
- Post-rsync chown applied: UniFi chown 1000:1000 /mnt/docker-db/unifi/ (Gotcha #36)

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Swarm Topology]]
- [[Service - home-assistant]]
- [[Service - unifi]]
