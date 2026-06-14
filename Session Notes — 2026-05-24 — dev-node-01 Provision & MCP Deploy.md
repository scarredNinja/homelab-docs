---
date: '2026-05-24T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm — Dev Node & MCP Deployment'
session_type: Deploy
status: Completed
tags:
  - SessionNotes
  - DockerSwarm
  - Provisioning
  - devnode
  - obsidian-mcp
  - Syncthing
---

# Session Notes — 2026-05-24 — dev-node-01 Provision & MCP Deploy

## 🎯 Session Goal

Provision `dev-node-01` (VMID 206, VLAN 40, IP `10.0.40.50`) as a new Docker Swarm worker node dedicated to dev/AI tooling, and deploy the `obsidian-mcp` and `syncthing` stacks onto it. Also finalised the `obsidian-mcp-server` project strategy (dedicated public GitHub repo vs monorepo).

---

## ✅ Completed

### 1. Obsidian MCP Server — Repo Strategy Decided

- Agreed to create a **dedicated, standalone public GitHub repository** (`scarredNinja/obsidian-mcp-server`) rather than including it in the homelab monorepo.
- Rationale: shareability with the OSS community, clean release tagging, isolated CI/CD pipeline for `ghcr.io/scarredninja/obsidian-mcp-server:latest`.
- Source lives in `C:\Users\DJ\Downloads\obsidian-mcp-server\` locally.
- Build verified: compiles cleanly, 0 TypeScript/Zod errors.

### 2. obsidian-mcp Stack Authored

- Stack file: `proxmox-swarm/stacks/stack-obsidian-mcp.yml`
- Image: `scarredninja/obsidian-mcp-server:latest`
- Placement constraint: `node.labels.zone == dev` (pins to `dev-node-01`)
- Bind mounts: `/mnt/docker-data/vault` → `/vault` (read-only)
- Docker secrets: `mcp_api_key`
- Traefik: `obsidian-mcp.home.purvishome.com` via `internal-only@file` middleware
- Env: `TRANSPORT=http`, `VAULT_PATH=/vault`, `VAULT_ALLOWED_FOLDERS=10 - Projects`

### 3. Syncthing Stack Authored

- Stack file: `proxmox-swarm/stacks/stack-syncthing.yml`
- Placement: `node.labels.zone == dev`
- Host-mode ports: `22000/tcp`, `22000/udp`, `21027/udp` (high-performance sync across VLANs)
- Bind mounts: `/mnt/docker-data/syncthing/config` + `/mnt/docker-data/vault`
- Traefik: `syncthing.home.purvishome.com` via `internal-only@file`

### 4. dev-node-01 Provisioned

- VMID: 206
- Hostname: `dev-node-01`
- IP: `10.0.40.50` (VLAN 40, static)
- RAM: 4096 MB, Cores: 4, Disk: 32G (rpool)
- Provisioned via `01-create-vm.sh` + `02-provision.sh` on PVE host

### 5. Initial Provisioning Timeout — Diagnosed & Fixed

> [!bug] Guest agent IP timeout — cloud-init DNS stall
> The first provision attempt timed out at 300 s waiting for the QEMU guest agent IP. Root cause: cloud-init `DataSourceNoCloud` was attempting DNS lookups during first boot on VLAN 40 before the default route/DNS was fully established. The VM had a static IP but the guest agent was slow to report.
>
> **Fix:** Added `network: {config: disabled}` override in the cloud-init user-data snippet to skip network reconfiguration, and ensured the static IP/gateway were written directly into the `99-pve-static.yaml` netplan file during VM creation. Reprovisioning succeeded on second attempt.

### 6. Post-Provision SSH Timeout — Diagnosed & Fixed

> [!bug] SSH timeout from Proxmox host — VLAN routing asymmetry
> After provisioning, the post-deploy script SSHing from PVE (`10.0.90.50`) to `dev-node-01` (`10.0.40.50`) timed out. `tcpdump` on `dev-node-01` showed TCP packets arriving from `10.0.4.85` (Windows workstation) but not from `10.0.90.50`. The Proxmox host was on VLAN 90 and had no route to VLAN 40.
>
> **Fix:** SSH from `dev-node-01` to a VLAN 60 jump host worked fine. The post-deploy script was updated to use the manager-01 (`10.0.60.30`) as an SSH jump host when running from PVE.

### 7. dev-node-01 Joined the Swarm ✅

- Node joined successfully with labels: `env=development`, `zone=dev`
- Confirmed via `docker node ls`:

| Node | Status | Availability | Role |
|------|--------|-------------|------|
| dev-node-01 | Ready | Active | Worker |
| manager-01 | Ready | Active | Leader |
| traefik-dmz-01 | Ready | Active | Worker |
| worker-controller-01 | Ready | Active | Worker |
| worker-media-01 | Ready | Active | Worker |
| worker-mediamanagement-01 | Ready | Active | Worker |
| worker-monitoring-01 | Ready | Active | Worker |

### 8. Pi-hole Gravity Sync Deployed ✅

- Gravity sync service deployed and running during this session (separate stack).

---

## 🐛 Bug #1 — Syncthing & obsidian-mcp Bind Mount Paths Missing on dev-node-01

> [!bug] `/mnt/docker-data/vault` and `/mnt/docker-data/syncthing/config` do not exist on `dev-node-01`
> Both `syncthing_syncthing` and `obsidian-mcp_mcp-server` are stuck at `0/1` replicas with error:
> `"invalid mount config for type "bind": bind source path does not exist: /mnt/docker-data/vault"`
> The node was freshly provisioned and has no pre-existing data directories. These paths must be created and the vault populated before the services can start.

> [!warning] Outstanding — next session
> 1. SSH into `dev-node-01` and create the required directories:
>    - `/mnt/docker-data/syncthing/config`
>    - `/mnt/docker-data/vault`
> 2. Configure Syncthing to sync the Obsidian vault from the Windows PC (VLAN 10/4.85) to `/mnt/docker-data/vault/`
> 3. Once vault is populated, `obsidian-mcp_mcp-server` should auto-recover (already in `Preparing` state)
> 4. Push `obsidian-mcp-server` source to `scarredNinja/obsidian-mcp-server` on GitHub and set up CI to publish `ghcr.io/scarredninja/obsidian-mcp-server:latest`

---

## 📊 Current State (End of Session)

### Swarm Nodes

| Node | Status | Labels |
|------|--------|--------|
| dev-node-01 | ✅ Ready / Active | `zone=dev`, `env=development` |
| manager-01 | ✅ Ready / Leader | — |
| traefik-dmz-01 | ✅ Ready / Active | — |
| worker-controller-01 | ✅ Ready / Active | — |
| worker-media-01 | ✅ Ready / Active | — |
| worker-mediamanagement-01 | ✅ Ready / Active | — |
| worker-monitoring-01 | ✅ Ready / Active | — |

### Key Service Status

| Service | Replicas | Notes |
|---------|----------|-------|
| obsidian-mcp_mcp-server | 🔴 0/1 | Bind mount `/mnt/docker-data/vault` missing on dev-node-01 |
| syncthing_syncthing | 🔴 0/1 | Bind mounts `/mnt/docker-data/vault` + `syncthing/config` missing |
| All other services | ✅ 1/1 or n/n | Healthy |

---

## ➡️ Next Session Priorities

- [x] SSH into `dev-node-01` and create bind-mount directories (`/mnt/docker-data/vault`, `/mnt/docker-data/syncthing/config`) [priority:: 1] #Syncthing #devnode ✅ 2026-05-31
- [x] Configure Syncthing: pair PC (`10.0.4.85:22000`) ↔ dev-node-01 (`10.0.40.50:22000`); set sync folder to `/mnt/docker-data/vault` [priority:: 1] #Syncthing ✅ 2026-05-31
- [x] Verify obsidian-mcp service recovers to 1/1 once vault is populated [priority:: 1] #obsidian-mcp ✅ 2026-05-31
- [x] Push obsidian-mcp-server source to GitHub (`scarredNinja/obsidian-mcp-server`) and configure GitHub Actions CI to publish image to `ghcr.io` [priority:: 2] #obsidian-mcp ✅ 2026-05-31
- [x] Update stack-obsidian-mcp.yml image reference to `ghcr.io/scarredninja/obsidian-mcp-server:latest` once CI is live [priority:: 2] #obsidian-mcp ✅ 2026-05-31
- [x] Resolve Syncthing phone-to-homelab sync (Android → VLAN 40:22000) for NewtonFit Samsung Health data [priority:: 3] #Syncthing #NewtonFit ✅ 2026-05-31
- [x] Create `VM - dev-node-01.md` note with full spec (see [[VM - dev-node-01]]) [priority:: 3] #devnode ✅ 2026-05-31

---

## 🔗 Related Notes

- [[VM - dev-node-01]]
- [[Service - obsidian-mcp-server]]
- [[Service - syncthing]] 
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
- [[Docker Swarm Infrastructure Runbook]]
- [[Swarm Topology]]
