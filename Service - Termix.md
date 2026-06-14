---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 9: External Access"
tags:
  - Termix
  - SSH
  - AdminAccess
  - Docker-Swarm
service_name: Termix
vm: worker-controller-01
service_status: active
deployment: Docker Swarm stack-termix.yml
last_updated: 2026-06-14T05:30:00.000Z
status: Completed
---

# Termix - SSH Portal

Termix is a self-hosted, web-based server management platform and SSH portal deployed on the Docker Swarm cluster to replace Termius. It provides secure terminal ingress to cluster nodes and other local hosts over a browser.

## Deployment

| Property | Value |
|---|---|
| Host VM | `worker-controller-01` (controller zone) |
| Stack file | `proxmox-swarm/stacks/stack-termix.yml` in [scarredNinja/docker-swarm-home](https://github.com/scarredNinja/docker-swarm-home) |
| Companion service | `guacd` (Guacamole daemon v1.6.0) for terminal rendering |
| Persistent Data | `/mnt/docker-data/termix` |
| Domain | `termix.home.purvishome.com` |
| Ingress Whitelist | Traefik `internal-only` (allows local subnets & Tailscale VPN `100.64.0.0/10`) |
| Status | Deployed and active |

## Architecture

```
Admin device (LAN / Tailscale VPN)
     (HTTPS Request to termix.home.purvishome.com)
Traefik Ingress Router (Ports 80/443, dynamic wildcard SSL cert)
     (Restricted via internal-only IPAllowList)
Termix Web App Service (Port 8080)
     (via stack-local overlay network 'termix-net')
guacd rendering daemon (Port 4822)
     (via VLAN 60 management network)
Target Homelab Hosts (SSH on Port 22)
```

## Migration Path from Termius

1. **Export connections from Termius:**
   - In Termius, export your connections database as JSON or standard SSH config.
2. **Import private keys:**
   - Import your default administration SSH key (e.g. `homelab_ed25519`) into Termix's credential manager.
3. **Map Connection Profiles:**
   - Create a dummy connection in Termix and export as JSON to view Termix's import format.
   - Translate Termius' connection JSON to the Termix JSON schema and upload via the Bulk Import feature.

## Troubleshooting

1. **Permission Denied on /app/data**
   - Fix: `sudo chown -R 1000:1000 /mnt/docker-data/termix` and `sudo chmod -R 755 /mnt/docker-data/termix` on the host VM.
2. **Invalid Credential Creation Data Validation Failed when Importing SSH Private Keys**
   - Cause: Node `ssh2` rejects PuTTY `.ppk` keys (convert to OpenSSH/PEM format via PuTTYgen) and fails if Windows CRLF line endings are pasted (convert key file to Unix LF line endings before copy-pasting).

## Related Notes

- [[01 Homelab Rebuild - Phase 9 External Access Hub]]
- [[Service - Tailscale]]
- [[Docker Swarm Infrastructure Runbook]]
