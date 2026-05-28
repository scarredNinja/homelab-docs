---
type: swarm-vm
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - VM
vmid: 211
hostname: manager-03
vcpu: 2
ram_gb: 4
disk_gb: 20
vlan_primary: 60
vlan_secondary:
ip_primary: 10.0.60.32
ip_secondary:
swarm_role: manager
node_labels:
  - node.role=manager
vm_status: running
post_boot_run: true
swarm_joined: true
last_updated: 2026-05-25
---

# manager-03

Swarm manager (Reachable). Third manager provisioned 2026-05-25 to achieve 3-manager quorum. No services pinned — pure control plane redundancy.

## Notes

- Provisioned 2026-05-25 via `02-provision-vm.sh --role manager --vmid-min 211 --memory 4096 --cores 2 --disk-size 20G`
- Joined swarm via `03-post-boot.sh` using manager token from `/mnt/docker-swarm/swarm/manager-token`
- Swarm state: Reachable (manager-01 remains Leader)
- All three managers on single Proxmox host — physical HA deferred until second host acquired

## Related

- [[VM - manager-01]]
- [[VM - manager-02]]
- [[Docker Swarm Infrastructure Runbook]]
- [[Swarm Topology]]
