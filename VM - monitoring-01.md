---
type: swarm-vm
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - VM
vmid: 200
hostname: monitoring-01
vcpu: 2
ram_gb: 4
disk_gb: 20
vlan_primary: 60
vlan_secondary:
ip_primary: 10.0.60.41
ip_secondary:
swarm_role: worker
node_labels:
  - node.role=monitoring
vm_status: provisioned
post_boot_run: true
swarm_joined: true
last_updated: 2026-04-12
---

# monitoring-01

Swarm worker node. Runs monitoring services — Grafana, Prometheus, InfluxDB.

## Notes



## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Swarm Topology]]
