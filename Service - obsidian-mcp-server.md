---
service_name: obsidian-mcp-server
service_status: developed
last_updated: 2026-05-24
project_id: Homelab-2025
tags: [mcp, obsidian, dev, service]
---

# Service — obsidian-mcp-server

## Overview

TypeScript MCP server exposing read-only tools for Obsidian vault access. Supports stdio (Claude Desktop) and HTTP (Swarm) transports. Client-agnostic — compatible with Claude Code, Claude Desktop, Cursor, Antigravity CLI, and any MCP-compliant client.

## Placement

- **Node:** `dev-node-01`
- **Zone:** `zone=dev`
- **URL:** `obsidian-mcp.home.purvishome.com`
- **Stack file:** `stacks/obsidian-mcp/stack-obsidian-mcp.yml` (pending)

## Environment Variables

| Variable | Required | Notes |
|---|---|---|
| `VAULT_PATH` | Yes | `/vault` in container |
| `VAULT_NAME` | No | Display name |
| `TRANSPORT` | Yes | `http` for Swarm |
| `PORT` | No | Default 3000 |
| `MCP_API_KEY` | Yes | Docker secret |
| `VAULT_ALLOWED_FOLDERS` | Yes | Comma-separated allowlist |

## Dependencies

- `dev-node-01` provisioned and in Swarm
- Syncthing vault sync running on `dev-node-01`
- Traefik `internal-only@file` middleware
- pfSense: VLAN 10 → VLAN 40:22000 (Syncthing), VLAN 40 ↔ VLAN 60 Swarm overlay ports

## Security & Hardening Audits

During our May 2026 audits, we completed several crucial hardening steps and identified three key platform optimization findings for future iterations:

1. **API Key Verification Middleware**: The Express endpoint now strictly requires an API key in HTTP mode, accepted via headers (`X-API-Key`), Bearer token (`Authorization: Bearer`), or query parameters.
2. **Folder Allowlist Case-Sensitivity Gotcha**: In `vault.ts`, permitted subdirectory checks are case-sensitive. When deploying inside a case-sensitive Docker container, ensure vault path environment variables match note path casings exactly (e.g. `10 - Projects` vs `10 - projects`).
3. **Wikilink Resolution Latency**: For very large vaults (1,000+ notes), `obsidian_find_backlinks` walks the entire vault and reads every single file sequentially. This is a potential CPU/disk bottleneck, and a local cache map or SQLite index is recommended if scaling.
4. **Hex Color/Commit Hash Tag Collisions**: Naive inline tag matches (e.g. `#FFF` or `#db7d1ff`) can trigger false tag categorization in markdown code blocks or text.

## Source & Git strategy

* **GitHub:** `scarredNinja/obsidian-mcp-server` (Public)
* **Strategy Decision (2026-05-24):** A dedicated, standalone public repository is chosen over a monorepo structure. This maximizes shareability with the developer community, allows clean release tagging, and streamlines isolated GitHub Actions CI/CD builds for `ghcr.io/scarredninja/obsidian-mcp-server:latest`.
* **Build status:** Compiled cleanly inside `/dist` (0 TypeScript / Zod errors). Ready to initialize Git and push.
