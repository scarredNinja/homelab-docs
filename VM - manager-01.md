---
type: swarm-vm
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - VM
vmid: 200
hostname: manager-01
vcpu: 2
ram_gb: 4
disk_gb: 20
vlan_primary: 60
vlan_secondary:
ip_primary: 10.0.60.30
ip_secondary:
swarm_role: manager
node_labels:
  - node.role=manager
vm_status: provisioned
post_boot_run: true
swarm_joined: true
last_updated: 2026-04-02
---

# manager-01

Swarm manager node. Runs control plane services — Portainer, Grafana, Prometheus, InfluxDB.

## Notes

- Currently at `10.0.60.101` — DHCP reservation for MAC `52:54:fa:68:e3:8a` not applying
- post-boot blocked until IP resolves to `.30`
- Swarm not yet initialised — waiting on post-boot

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Swarm Topology]]
