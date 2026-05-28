---
date: 2026-05-28
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm — Dev Sandboxes & Ingestion Setup"
session_type: "Code"
status: "Complete"
tags:
  - SessionNotes
  - Finance
  - DockerSwarm
  - Compilation
---

# Session Notes — 2026-05-28 — Finance App Container Compilation Complete

## 🎯 Session Goal
Resolve the remaining Prisma compilation and TypeScript type-checking roadblocks to successfully build and publish the production-ready container image for the **Next.js Household Finance App** (`webdev`) to the GitHub Container Registry.

---

## ✅ Completed

### 1. Workstation Docker Backend Service Launch
*   **Daemon Restored**: Identified that Docker Desktop was stopped on the workstation. Successfully started the background daemon service executable directly (`com.docker.backend.exe`) and verified online socket connectivity via `docker ps`.

### 2. Lockfile & TypeScript Resolution
*   **Lockfile Alignment**: Ran `npm install --package-lock-only` on the host workstation to safely regenerate `package-lock.json` and pin all dependencies to stable **Prisma v6.4.0** without triggering `node_modules` EPERM locking conflicts.
*   **Prisma Type-Checking Fix**: Resolved a Next.js compilation failure inside the legacy `prisma.config.ts` by adding a fallback empty string `|| ""` to `process.env["DATABASE_URL"]`, preventing a strict TypeScript mismatch.

### 3. Build & Publication [SUCCESS 🟢]
*   **Docker Build**: Compiled the full multi-stage Next.js production standalone bundle cleanly. All static pages prerendered successfully, and all API routes type-checked.
*   **Registry Push**: Successfully pushed the finalized image to **`ghcr.io/scarredninja/webdev:latest`** (digest: `sha256:d1efc8346c818207086cfa7b904a4790d87fe9b50b7314de0693993d7f1b4326`).

### 4. Traefik v3 Static Config Migration
*   **Config Updated**: Audited and migrated `config/traefik/traefik.yaml` to comply with Traefik v3 ACME DNS challenge schema, resolving the `propagation` block parse crash. Committed and pushed directly to origin `main`.

---

## 🐛 Bug #74 — Traefik v3 dnsChallenge 'propagation' field not found
> [!bug] Root Cause
> After upgrading Traefik to v3 on the `main` branch, the container crashed on startup with `error=command traefik error: field not found, node: propagation`. In Traefik v3, the `propagation` block under `dnsChallenge` is deprecated and removed. Its internal child setting `delayBeforeChecks` is renamed to `delayBeforeCheck` (singular) and moved directly under `dnsChallenge`.
>
> **Fix**: Updated `config/traefik/traefik.yaml` from:
> ```yaml
>       dnsChallenge:
>         provider: cloudflare
>         resolvers:
>           - "1.1.1.1:53"
>           - "1.0.0.1:53"
>         propagation:
>           delayBeforeChecks: 10
> ```
> to:
> ```yaml
>       dnsChallenge:
>         provider: cloudflare
>         resolvers:
>           - "1.1.1.1:53"
>           - "1.0.0.1:53"
>         delayBeforeCheck: 10
> ```

---

## 📊 Current State (End of Session)

| Service Name | Stack Name | Ingress URL | Placement | Status |
|---|---|---|---|---|
| `dev-newtonfit` | `dev-newtonfit` | `dev-newtonfit.home.purvishome.com` | `dev-node-01` | 🟢 Healthy |
| `dev-finance` | `dev-finance` | `dev-finance.home.purvishome.com` | `dev-node-01` | 🟢 Healthy & Verified |

---

## ➡️ Next Session Priorities
- [x] Log into GitHub Profile and navigate to **Packages** -> **`webdev`** -> **Package settings** -> **Change visibility** and toggle it to **Public**.
- [x] SSH into `dev-node-01` (`10.0.40.50`) and pre-provision the database host volume directory:
  ```bash
  sudo mkdir -p /mnt/docker-data/finance-dev/db
  sudo chmod -R 777 /mnt/docker-data/finance-dev
  ```
- [x] Add a local DNS A record in Pi-hole: `dev-finance.home.purvishome.com` -> `10.0.60.40`.
- [x] Deploy the `dev-finance` stack in Portainer using path `proxmox-swarm/stacks/stack-dev-finance.yml` on your infra repo `main` branch.
- [x] Open `https://dev-finance.home.purvishome.com` and verify the Next.js database migration executes successfully.

---

## 🔗 Related Notes
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
- [[Session Notes — 2026-05-27 — Dev Sandboxes & Ingestion Setup]]
- [[Finance App - Deployment Notes]]
