---
disk_gb: 10
hostname: traefik-dmz-01
ip_primary: 10.0.60.40
ip_secondary: 10.0.80.20
last_updated: '2026-06-02'
node_labels:
  - node.labels.zone=public
phase: 'Phase 5: Docker Swarm'
post_boot_run: true
project_id: Homelab-2025
ram_gb: 1
swarm_joined: true
swarm_role: worker
tags:
  - DockerSwarm
  - VM
type: swarm-vm
vcpu: 2
vlan_primary: 60
vlan_secondary: 80
vm_status: running
vmid: 201
status: Active
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

> [!success] Permanent netplan fix implemented — survives reboot
> The permanent Netplan routing fix has been implemented by adding `dhcp4-overrides: use-routes: false` to the `enp6s19` network interface configuration.
> This prevents dual default routes, ensuring all non-ingress outbound traffic routes correctly via the primary interface (`eth0`, VLAN 60), eliminating asymmetric routing issues and surviving system reboots.

- [x] Temp asymmetric routing fix applied ✅ 2026-04-21
- [x] Permanent netplan fix — `dhcp4-overrides: use-routes: false` on `enp6s19` [priority:: 1] ✅ 2026-06-02

## Related

- [[Traefik Routing Architecture]]
- [[Docker Swarm Infrastructure Runbook]]
- [[Service - traefik]]
