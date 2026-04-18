---
type: swarm-vm
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - VM

vmid:
hostname: worker-general-01
vcpu: 2
ram_gb: 4
disk_gb: 20

vlan_primary: 50
vlan_secondary: 100
ip_primary:
ip_secondary:

swarm_role: worker
node_labels:
  - "node.labels.zone=homelab"
  - "node.labels.nas=nfs"

vm_status: not-created
post_boot_run: false
swarm_joined: false
last_updated: 2026-04-02
---

- [ ] Needs to be updated  [priority:: 2]
# worker-general-01

Homelab/dev worker. Dev, test, and general workloads on VLAN 40.
Future home for public-facing services (website etc.) if needed.

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Swarm Topology]]
