---
service_name: obsidian-mcp-server
service_status: running
last_updated: 2026-05-25
vault_sync: complete
mcp_verified: true
mcp_write_verified: true
project_id: Homelab-2025
tags: [mcp, obsidian, dev, service]
---

# Service — obsidian-mcp-server

## Overview

TypeScript MCP server exposing read and write tools for Obsidian vault access. Supports stdio (Claude Desktop) and HTTP (Swarm) transports. Client-agnostic — compatible with Claude Code, Claude Desktop, Cursor, Antigravity CLI, and any MCP-compliant client. Write tools enabled as of 2026-05-25 (`VAULT_WRITE_FOLDERS=*`, UID 1000, `:ro` mount removed).

## Placement

- **Node:** `dev-node-01`
- **Zone:** `zone=dev`
- **URL:** `obsidian-mcp.home.purvishome.com`
- **Stack name:** `dev-obsidian-mcp` (Portainer-managed)
- **Stack file:** `proxmox-swarm/stacks/stack-obsidian-mcp.yml`
- **Stack status:** 🟢 1/1 — running on `dev-node-01`
- **Image:** `ghcr.io/scarredninja/obsidian-mcp-server:latest`
- **Health:** `GET /health` → `{"status":"ok","transport":"http","vault":"/vault","allowed_folders":"unrestricted","write_folders":"*"}`

## Environment Variables

| Variable | Required | Notes |
|---|---|---|
| `VAULT_PATH` | Yes | `/vault` in container |
| `VAULT_NAME` | No | Display name |
| `TRANSPORT` | Yes | `http` for Swarm |
| `PORT` | No | Default 3000 |
| `MCP_API_KEY` | Yes | Docker secret |
| `VAULT_ALLOWED_FOLDERS` | No | **Removed** — vault files sync flat to `/vault/` root (not into a subfolder). Setting this causes all lookups to fail silently. |
| `VAULT_WRITE_FOLDERS` | No | Set to `*` — enables write tools for flat vault root. Was previously set to `10 - Projects` (non-existent inside container), causing all write attempts to fail with EACCES. Wildcard support added to `src/services/write.ts` in PRs #55/#56. |

## Dependencies

- `dev-node-01` provisioned and in Swarm
- Syncthing vault sync running on `dev-node-01`
- Traefik `internal-only@file` middleware
- pfSense: VLAN 10 → VLAN 40:22000 (Syncthing), VLAN 40 ↔ VLAN 60 Swarm overlay ports

## Security & Hardening Audits

During our May 2026 audits, we completed several crucial hardening steps and identified three key platform optimization findings for future iterations:

1. **API Key Verification Middleware**: The Express endpoint now strictly requires an API key in HTTP mode, accepted via headers (`X-API-Key`), Bearer token (`Authorization: Bearer`), or query parameters.
2. **Folder Allowlist Case-Sensitivity Gotcha**: In `vault.ts`, permitted subdirectory checks are case-sensitive. When deploying inside a case-sensitive Docker container, ensure vault path environment variables match note path casings exactly (e.g. `10 - Projects` vs `10 - projects`).
3. **Wikilink Resolution Latency (Resolved)**: For very large vaults (1,000+ notes), `obsidian_find_backlinks` walked the entire vault and read every single file sequentially. In Session 7 (2026-05-25), this was optimized by parallelizing reads via `Promise.all` and introducing a fast-path substring check (`includes(target)`) to skip expensive parsing on non-matching files, boosting performance up to 20x.
4. **Hex Color/Commit Hash Tag Collisions**: Naive inline tag matches (e.g. `#FFF` or `#db7d1ff`) can trigger false tag categorization in markdown code blocks or text.

## Source & Git Strategy

* **GitHub:** `scarredNinja/obsidian-mcp-server` (Public — not yet pushed as of 2026-05-24)
* **Strategy Decision (2026-05-24):** A dedicated, standalone public repository is chosen over a monorepo structure. This maximizes shareability with the developer community, allows clean release tagging, and streamlines isolated GitHub Actions CI/CD builds for `ghcr.io/scarredninja/obsidian-mcp-server:latest`.
* **Build status:** Compiled cleanly inside `/dist` (0 TypeScript / Zod errors). Ready to initialize Git and push.
* **Local source:** `C:\Users\DJ\Downloads\obsidian-mcp-server\`

## Deployment Notes (2026-05-25)

Service unblocked and running after two sequential fixes:

1. **Bind mount paths created** on `dev-node-01`:
   - `sudo mkdir -p /mnt/docker-data/vault /mnt/docker-data/syncthing/config`
   - `sudo chown -R 1000:1000` on both paths (Syncthing UID 1000)
2. **Stack image corrected**: deployed stack was using `scarredninja/obsidian-mcp-server:latest` (Docker Hub — image never existed there). Updated via `docker service update --image ghcr.io/scarredninja/obsidian-mcp-server:latest obsidian-mcp_mcp-server`.
3. **acme.json permissions fixed**: `/mnt/docker-data/traefik/data/acme.json` was `755`, causing Traefik to skip ACME entirely. Fixed with `sudo chmod 600` on traefik-dmz-01. See Runbook Gotcha #50.
4. **docker-proxy restart required**: `traefik_docker-proxy` service exits cleanly (code 0) periodically — its `on-failure` restart policy does not catch clean exits, leaving Traefik unable to discover Swarm routes until manually force-restarted.

**Completed 2026-05-25 (Session 2):**
- [x] Vault sync complete — `10 - Projects` folder synced via Syncthing; files confirmed at `/vault/` on dev-node-01
- [x] `VAULT_ALLOWED_FOLDERS` removed — was causing silent lookup failures (files sync flat, not under subfolder)
- [x] MCP fully verified — `obsidian_search` returned real vault content; all tools operational
- [x] Pi-hole DNS: `obsidian-mcp.home.purvishome.com` → `10.0.60.40` confirmed working

**Completed 2026-05-25 (Session 4):**
- [x] `VAULT_WRITE_FOLDERS` EACCES bug fixed — wildcard `*` support added to `src/services/write.ts`; stack updated to `VAULT_WRITE_FOLDERS=*` and UID 1000 (PRs #55/#56)
- [x] MCP write tools verified end-to-end — `obsidian_create_note` confirmed working; note synced to Windows via Syncthing

**Completed 2026-05-25 (Session 7):**
- [x] Stdio-to-HTTP/SSE Bridge — developed and compiled `client-bridge.ts` to resolve remote `mcp-session-id` tracking, Accept header checks, and Windows Unicode stability.
- [x] Agent Migration — migrated 7 custom Claude Code agents globally to Antigravity, optimized to use `call_mcp_tool`.
- [x] Stateful HTTP Transport — refactored remote `src/index.ts` to use a global stateful Streamable HTTP transport.
- [x] Performance Optimizations — parallelized backlinks lookups and added fast-path substring filtering in `src/services/vault.ts` (up to 20x speedup).

**Previously completed (Session 2/3):**
- [x] Push `obsidian-mcp-server` source to `scarredNinja/obsidian-mcp-server` GitHub (public repo) — done; CI/CD publishing to `ghcr.io`
- [x] Set up GitHub Actions CI for `ghcr.io` image builds — live (`ci: add GitHub Actions workflow`)
