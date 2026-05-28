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
vm_status: running
post_boot_run: true
swarm_joined: true
last_updated: 2026-05-21
---

# manager-01

Swarm manager (leader). Runs control plane services — Portainer server, `docker-socket-proxy` sidecar for Traefik service discovery.

## Notes

- Swarm initialised and stable — leader since 2026-04-05
- Portainer deployed in agent mode, accessible at `https://portainer.home.purvishome.com`
- Runs `docker-socket-proxy` sidecar — exposes read-only Swarm API to `traefik-dmz-01` over `traefik-backend` overlay
- Inter-VM SSH configured ✅ 2026-04-25 — `~/.ssh/id_ed25519` → all workers (password-free)
- Portainer agent overlay stale VXLAN + `local-kv.db` cleared and confirmed healthy ✅ 2026-04-25

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Swarm Topology]]
- [[Service - portainer]]
