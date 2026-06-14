---
type: swarm-vm
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
tags:
  - DockerSwarm
  - VM
vmid: null
hostname: worker-general-01
vcpu: 2
ram_gb: 4
disk_gb: 20
vlan_primary: 50
vlan_secondary: 100
ip_primary: null
ip_secondary: null
swarm_role: worker
node_labels:
  - node.labels.zone=homelab
  - node.labels.nas=nfs
vm_status: not-created
post_boot_run: false
swarm_joined: false
last_updated: '2026-04-25T00:00:00.000Z'
status: Active
---

# worker-general-01

> [!warning] Superseded — this VM was never provisioned. The role was split: media management workloads went to [[VM - worker-mediamanagement-01]] (VLAN 50). This note is retained for historical reference only.

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Swarm Topology]]
