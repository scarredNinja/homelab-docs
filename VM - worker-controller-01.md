---
type: swarm-vm
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - VM

vmid:
hostname: worker-controller-01
vcpu: 2
ram_gb: 4
disk_gb: 20

vlan_primary: 60
vlan_secondary: 20
ip_primary:
ip_secondary:

swarm_role: worker
node_labels:
  - "node.labels.zone=controller"
  - "node.labels.nas=nfs"

vm_status: not-created
post_boot_run: false
swarm_joined: false
last_updated: 2026-04-02
---

# worker-controller-01

Controller worker. Dual-homed — VLAN 60 management, VLAN 20 IoT access.
Runs Home Assistant and UniFi. Both need direct IoT VLAN reach.

## Notes

- HA must stop cleanly before container shutdown — SQLite corruption risk
- UniFi uses MongoDB — local virtiofs dataset, not NFS

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Swarm Topology]]
