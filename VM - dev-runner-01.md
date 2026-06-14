---
type: swarm-vm
vm_status: running
vmid: 207
hostname: dev-runner-01
ip_primary: 10.0.40.51
vlan_primary: 40
vlan_secondary: ''
post_boot_run: true
swarm_joined: true
project_id: Homelab-2025
last_updated: '2026-06-14T10:30:00.000Z'
tags:
  - DockerSwarm
  - VM
  - dev
  - runner
status: Active
phase: 'Phase 5: Docker Swarm'
---

# VM — dev-runner-01

## Spec

| Field | Value |
|-------|-------|
| VMID | 207 |
| Hostname | `dev-runner-01` |
| IP | `10.0.40.51` |
| VLAN | 40 (Dev/AI) |
| RAM | 4096 MB |
| Cores | 4 vCPU |
| Disk | 40 GB (rpool) |
| OS | Ubuntu 24.04 (cloud-init) |
| Docker | 29.5.3 |
| Swarm role | Worker |
| Swarm labels | `zone=runner` |
| Provisioned | 2026-06-14 |

## Purpose

Dedicated worker node for untrusted continuous integration (CI) runners (specifically Woodpecker Agent). 

> [!CAUTION] Security Sandboxing
> Unlike other VMs in the swarm, `dev-runner-01` is provisioned with **no ZFS virtiofs host mounts attached** (using the `--no-virtiofs` flag). Untrusted build steps execute as local containers in the VM's local Docker daemon via `/var/run/docker.sock`, completely isolated from any host datasets to prevent directory traversal or malicious container escapes from reaching the host's `/mnt/docker-data` or `/mnt/docker-db`.

## Services Pinned to This Node

| Service | Stack | Status |
|---------|-------|--------|
| woodpecker-agent | `woodpecker` | 🔲 Planned |

## Storage Layout

| Host Path | Purpose |
|-----------|---------|
| Local VM Disk only | Local scratch workspace for builds. Zero host dataset mounts. |

## Network

| Field | Value |
|-------|-------|
| Primary IP | `10.0.40.51` |
| Gateway | `10.0.40.1` |
| DNS | `10.0.40.1` (pfSense) |
| Swarm overlay | 10.200.x.0/24 (auto) |

## Provisioning Notes

- Provisioned using `02-provision-vm.sh --no-virtiofs` on 2026-06-14.
- Joined to swarm and configured using `03-post-boot.sh`.
- Firewall rules in pfSense allow communication from `10.0.40.51` to manager nodes on Swarm control plane ports (`2377/tcp`, `7946/tcp/udp`, `4789/udp`) and to Woodpecker Server on `9000/tcp`.

## Related Notes

- [[Session Notes — 2026-06-14 — Gitea Forgejo and Woodpecker CI Setup]]
- [[06 Self-Hosted Developer Platform — Master Hub]]
- [[Docker Swarm Infrastructure Runbook]]
