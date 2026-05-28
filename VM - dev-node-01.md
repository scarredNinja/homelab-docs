---
type: swarm-vm
vm_status: running
vmid: 206
hostname: dev-node-01
ip_primary: 10.0.40.50
vlan_primary: 40
vlan_secondary: ""
post_boot_run: true
swarm_joined: true
project_id: Homelab-2025
last_updated: 2026-05-25
tags: [swarm-vm, dev, devnode]
---

# VM — dev-node-01

## Spec

| Field | Value |
|-------|-------|
| VMID | 206 |
| Hostname | `dev-node-01` |
| IP | `10.0.40.50` |
| VLAN | 40 (Dev/AI) |
| RAM | 4096 MB |
| Cores | 4 vCPU |
| Disk | 32 GB (rpool) |
| OS | Ubuntu 24.04 (cloud-init) |
| Docker | 29.5.2 |
| Swarm role | Worker |
| Swarm labels | `zone=dev`, `env=development` |
| Provisioned | 2026-05-24 |

## Purpose

Dedicated worker node for dev/AI workloads. Hosts services that require access to the Obsidian vault or local data not suitable for shared NFS. Isolated on VLAN 40.

## Services Pinned to This Node

| Service | Stack | Status |
|---------|-------|--------|
| [[Service - obsidian-mcp-server]] | `dev-obsidian-mcp` (Portainer) | 🟢 Running 1/1 |
| syncthing | `dev-syncthing` (Portainer) | 🟢 Running 1/1 |

## Storage Layout

| Host Path | Purpose |
|-----------|---------|
| `/mnt/docker-data/vault` | Obsidian vault (synced via Syncthing) |
| `/mnt/docker-data/syncthing/config` | Syncthing config persistence |

> [!info] Both services running 1/1 as of 2026-05-25. Vault synced via Syncthing (`10 - Projects` folder). MCP verified — `obsidian_search` returning real vault content.

## Network

| Field | Value |
|-------|-------|
| Primary IP | `10.0.40.50` |
| Gateway | `10.0.40.1` |
| DNS | `10.0.40.1` (pfSense) |
| Swarm overlay | 10.200.x.0/24 (auto) |
| Syncthing sync port | `22000` TCP/UDP (host mode) |
| Syncthing discovery | `21027` UDP (host mode) |

## Provisioning Notes

- First provisioning attempt timed out at 300 s (guest agent IP stall during cloud-init on VLAN 40). Fixed by reprovisioning with corrected netplan static config.
- SSH from Proxmox host (`10.0.90.50`) was blocked due to missing VLAN 90→40 route. Post-deploy script uses manager-01 (`10.0.60.30`) as a jump host.

## Related Notes

- [[Session Notes — 2026-05-24 — dev-node-01 Provision & MCP Deploy]]
- [[Service - obsidian-mcp-server]]
- [[Swarm Topology]]
- [[Docker Swarm Infrastructure Runbook]]
