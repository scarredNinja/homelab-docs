---
type: swarm-vm
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - VM
vmid: 201
hostname: traefik-dmz-01
vcpu: 2
ram_gb: 1
disk_gb: 10
vlan_primary: 60
vlan_secondary: 80
ip_primary: 10.0.60.40
ip_secondary: 10.0.80.20
swarm_role: worker
node_labels:
  - node.labels.zone=public
vm_status: running
post_boot_run: true
swarm_joined: true
last_updated: 2026-04-25
---

# traefik-dmz-01

DMZ worker node. Dual-homed — VLAN 60 for management, VLAN 80 for external ingress.
Runs Traefik only. Only VM with a public-facing NIC.

## Notes

- Dual-homed: `eth0` VLAN 60 (10.0.60.40) for Swarm/management, `enp6s19` VLAN 80 (10.0.80.20) for external ingress
- Runs Traefik only — `zone=public` placement constraint
- `acme.json` at `/mnt/docker-data/traefik/data/acme.json` — chmod 600 applied ✅ 2026-04-25; Cloudflare cert resolver now active
- File-provider routes loading correctly (YAML parse error in `infrastructure.yml` fixed ✅ 2026-04-25)

## Session 2026-04-21 — Asymmetric Routing Fix

> [!bug] Bug #21 — Dual default routes causing ECMP split
> Both NICs received default routes via DHCP at metric 100. Linux ECMP split return traffic — ~50% of SYN-ACK replies left via `enp6s19` (VLAN 80) bypassing pfSense, breaking TLS handshake.

> [!warning] Permanent fix outstanding — does not survive reboot
> Temporary fix applied: deleted VLAN 80 default route.
> Permanent fix: add `dhcp4-overrides: use-routes: false` to `enp6s19` stanza in netplan.
> Same pattern as media VMs on VLAN 100 (see [[Docker Swarm Infrastructure Runbook]] Gotcha #25).

- [x] Temp asymmetric routing fix applied ✅ 2026-04-21
- [ ] Permanent netplan fix — `dhcp4-overrides: use-routes: false` on `enp6s19` [priority:: 1]

## Related

- [[Traefik Routing Architecture]]
- [[Docker Swarm Infrastructure Runbook]]
- [[Service - traefik]]
