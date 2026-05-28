---
date: 2026-05-27
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm — Dev Sandboxes & Ingestion Setup"
session_type: "Planning + Deploy"
status: "Partial — Finance Build Blocked"
tags:
  - SessionNotes
  - NewtonFit
  - Syncthing
  - Finance
  - DockerSwarm
  - Networking
---

# Session Notes — 2026-05-27 — Dev Sandboxes & Ingestion Setup

## 🎯 Session Goal
Establish isolated development sandbox environments on `dev-node-01` (VLAN 40) for **NewtonFit** and the **Next.js Household Finance App**, route them securely via Traefik secure ingress, and configure real-time automated phone-to-sandbox data synchronization using Syncthing.

---

## ✅ Completed

### 1. NewtonFit Dev Sandbox (100% Completed & Verified)
*   **Sandbox Stack**: Deployed `stack-dev-newtonfit.yml` pinned to run exclusively on `dev-node-01` (`node.labels.zone == dev`).
*   **Volume Isolation**: Mapped host ZFS path `/mnt/docker-data/newtonfit-dev` to `/app/data` in the container. Production stack dismantled.
*   **Traefik Ingress**: Exposed secure endpoint `https://dev-newtonfit.home.purvishome.com` with Cloudflare wildcard TLS resolver. Verified glassmorphic settings panel and biomarker trend charts load successfully.

### 2. Syncthing Ingestion & VLAN Bypass
*   **Ingestion Volume**: Updated `stack-syncthing.yml` to mount `/mnt/docker-data/newtonfit-dev/imports:/var/syncthing/newtonfit-dev-imports` to drop workout CSVs straight into NewtonFit's file watcher.
*   **Image Pin Corrected**: Reverted Syncthing container image from `:1.27` back to `:latest` to match the user's persistent `config.xml` (version 52 vs version 37).
*   **VLAN Firewall Rules**: Added a narrow TCP/UDP `PASS` rule on pfSense `VLAN10_HOME` to allow port `22000` traffic directly to the container host (`10.0.40.50`).
*   **Phone-to-Sandbox Sync**: Paired Samsung Galaxy S22 Ultra (`SM-S908E`) with the container using a direct TCP address mapping (`tcp://10.0.40.50:22000`) to bypass inter-VLAN multicast discovery blocks. Sync active and healthy.

### 3. Finance App Sandbox Preparation
*   **Swarm Stack Definition**: Created and committed `stack-dev-finance.yml` to `docker-swarm-home` repository `main` branch. Implements strict resource caps (0.5 CPU, 512MB RAM), PostgreSQL 16 local database mapping, overlay private networks, and Traefik labels at `https://dev-finance.home.purvishome.com`.
*   **Build Automation**: Created `build-and-push.ps1` inside the `webdev` repository to build and publish the container image to GHCR.
*   **Prisma Downgrade**: Downgraded `@prisma/client` and `prisma` to stable `v6.4.0` in `package.json` to avoid Prisma 7's breaking driver adapter requirements (`@prisma/adapter-pg`). Restored standard `url = env("DATABASE_URL")` in `schema.prisma`.

---

## 🐛 Bug #71 — Syncthing Config XML Version Crash
> [!bug] Root Cause
> Swarm Syncthing image pin `:1.27` crashed on startup because the persistent ZFS config file `config.xml` had been upgraded by a previous `:latest` instance to configuration version 52, which is backward-incompatible with v1.27's configuration parser (expects version 37).
>
> **Fix**: Reverted `stack-syncthing.yml` image tag back to `:latest` to match the active config.xml schema.

## 🐛 Bug #72 — Inter-VLAN Syncthing Discovery Block
> [!bug] Root Cause
> Syncthing multicast discovery (Local Discovery) is blocked by default across VLAN boundaries (Phone on VLAN 10, Dev node on VLAN 40).
>
> **Fix**: Configured static device path mapping `tcp://10.0.40.50:22000` in the phone's Syncthing client, and created a stateful pfSense `PASS` rule on port `22000` for `VLAN10_HOME` → `10.0.40.50`.

## 🐛 Bug #73 — Prisma 7 static build crash & lockfile mismatch
> [!bug] Root Cause
> Prisma 7 deprecates `url` from `schema.prisma` datasource block and throws a compilation validation error (P1012) when Next.js attempts static page data collection inside the Docker builder container without a live database or driver adapter.
>
> **Fix**: Pinned Prisma to `^6.4.0` in `package.json` and reverted to standard database URL parsing. However, the Docker build is currently failing because the build context contains an out-of-sync `package-lock.json` with Prisma 7 metadata, which forces `npm install` inside the container to resolve to Prisma 7.

---

## 📊 Current State (End of Session)

| VM Name | IP | Swarm Role | Status | Zones / VLANs |
|---|---|---|---|---|
| `manager-01` | `10.0.60.30` | Swarm Leader | 🟢 Active | Swarm Mgmt (VLAN 60) |
| `manager-02` | `10.0.60.31` | Swarm Manager | 🟢 Active | Swarm Mgmt (VLAN 60) |
| `manager-03` | `10.0.60.32` | Swarm Manager | 🟢 Active | Swarm Mgmt (VLAN 60) |
| `dev-node-01` | `10.0.40.50` | Swarm Worker | 🟢 Active | Dev Sandbox (VLAN 40) |

| Service Name | Stack Name | Ingress URL | Placement | Status |
|---|---|---|---|---|
| `dev-newtonfit` | `dev-newtonfit` | `dev-newtonfit.home.purvishome.com` | `dev-node-01` | 🟢 Healthy |
| `syncthing` | `syncthing` | `syncthing.home.purvishome.com` | `worker-general-01` | 🟢 Healthy |
| `dev-finance` | `dev-finance` | `dev-finance.home.purvishome.com` | `dev-node-01` | 🔲 Blocked (Build) |

---

## ➡️ Next Session Priorities
- [ ] Regenerate host `package-lock.json` in `webdev` repository by running `npm install` locally on the workstation to align it with Prisma v6.
- [ ] Run `.\build-and-push.ps1` to publish `ghcr.io/scarredninja/webdev:latest` to GHCR.
- [ ] SSH into `dev-node-01` (`10.0.40.50`) and pre-provision the database host volume directory:
  ```bash
  sudo mkdir -p /mnt/docker-data/finance-dev/db
  sudo chmod -R 777 /mnt/docker-data/finance-dev
  ```
- [ ] Add a local DNS A record in Pi-hole: `dev-finance.home.purvishome.com` -> `10.0.60.40`.
- [ ] Deploy the `dev-finance` stack in Portainer using path `proxmox-swarm/stacks/stack-dev-finance.yml` on your infra repo `main` branch.
- [ ] Open `https://dev-finance.home.purvishome.com` and verify database migrations and user login tables successfully load.

---

## 🔗 Related Notes
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
- [[05 Personal Fitness & Health - Phase 1 Ingestion & Target Sync Hub]]
- [[VM - dev-node-01]]
- [[Service - newtonfit]]
- [[Finance App - Deployment Notes]]
