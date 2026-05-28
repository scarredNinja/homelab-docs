---
project_id: Homelab-2025
date: 2026-05-25
tags: [session-note, homelab, docker-swarm, mcp, syncthing, obsidian]
---

# Session Notes — 2026-05-25 — dev-node-01 Provision & MCP Deploy

## Session 1 (earlier today)

### Completed

- **Provisioned dev-node-01** (VMID 206, VLAN 40, `10.0.40.50`) — joined Swarm as worker (7 nodes total). Fixed two sequential provisioning blockers:
  - Cloud-init DNS stall on VLAN 40 — fixed via corrected static netplan config
  - SSH from Proxmox host (`10.0.90.50`) blocked by missing VLAN 90→40 route — resolved by using `manager-01` (`10.0.60.30`) as jump host
- **Created bind mount directories** on dev-node-01:
  - `sudo mkdir -p /mnt/docker-data/vault /mnt/docker-data/syncthing/config`
  - `sudo chown -R 1000:1000` on both paths (Syncthing UID 1000)
- **Fixed obsidian-mcp image** — deployed stack pointed to `scarredninja/obsidian-mcp-server:latest` (Docker Hub, never existed). Updated to `ghcr.io/scarredninja/obsidian-mcp-server:latest` via `docker service update`. Committed as `deploy: update obsidian-mcp-server image to point to ghcr.io`.
- **Fixed `acme.json` permissions** — `/mnt/docker-data/traefik/data/acme.json` was `755`, causing Traefik to silently skip ACME certificate renewal entirely. Fixed with `sudo chmod 600` on `traefik-dmz-01`. Added to Runbook as **Gotcha #50**.
- **Fixed `traefik_docker-proxy` restart policy** — service exits cleanly (code 0) periodically; `on-failure` restart policy does not catch clean exits, leaving Traefik unable to discover Swarm routes. Changed `condition: on-failure` → `condition: any` in `stack-traefik.yml`.
- **Syncthing peered** — Windows Syncthing client connected to `dev-node-01` with static address `tcp://10.0.40.50:22000`. `10 - Projects` folder sync initiated.
- **Pi-hole DNS** — added `obsidian-mcp.home.purvishome.com` → `10.0.60.40` (traefik-dmz-01) entry.

---

## Session 2

### Completed

- **Vault sync completed** — user synced `10 - Projects` folder via Syncthing; files appeared under `/mnt/docker-data/vault/` on `dev-node-01`.
- **Diagnosed `VAULT_ALLOWED_FOLDERS` misconfiguration** — the env var was set to `"10 - Projects,Archive"` expecting files at `/vault/10 - Projects/`. However, Syncthing syncs the shared folder *itself* as the root, so vault markdown files land flat at `/vault/` (no subdirectory). The allowlist caused every file lookup to fail silently.
- **Removed `VAULT_ALLOWED_FOLDERS` from stack** — health endpoint now returns `"allowed_folders":"unrestricted"`. Committed as `fix(obsidian-mcp): remove VAULT_ALLOWED_FOLDERS — vault root is synced folder`.
- **MCP fully verified** — `obsidian_search` tool returned real vault content (3 results for "docker swarm" query). All tools confirmed operational.

### Key Gotchas

> [!warning] Syncthing Folder Sync vs Vault Root Path
> When Syncthing shares a folder (e.g. `10 - Projects`), it syncs the *contents* of that folder into the configured destination path — not a subdirectory named after the folder. If Syncthing is configured to sync to `/mnt/docker-data/vault`, vault markdown files appear at `/vault/` inside the container, **not** at `/vault/10 - Projects/`.
>
> Setting `VAULT_ALLOWED_FOLDERS="10 - Projects"` expects files at `/vault/10 - Projects/*.md` — which never exists. Solution: remove `VAULT_ALLOWED_FOLDERS` entirely (unrestricted) or set it to a subfolder that actually exists inside the synced root.

### MCP Configuration

**Endpoint:** `https://obsidian-mcp.home.purvishome.com/mcp`

**Required headers:**
- `X-API-Key: <secret>`
- `Accept: application/json, text/event-stream`

**Health check:**
```bash
curl -s https://obsidian-mcp.home.purvishome.com/health \
  -H "X-API-Key: <secret>" | jq .
# → {"status":"ok","transport":"http","vault":"/vault","allowed_folders":"unrestricted"}
```

**Test tool call:**
```bash
curl -s https://obsidian-mcp.home.purvishome.com/mcp \
  -H "X-API-Key: <secret>" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"obsidian_search","arguments":{"query":"docker swarm"}}}'
```

**Claude Code MCP config** (`~/.claude/mcp.json`):
```json
{
  "mcpServers": {
    "obsidian": {
      "type": "http",
      "url": "https://obsidian-mcp.home.purvishome.com/mcp",
      "headers": {
        "X-API-Key": "<secret>"
      }
    }
  }
}
```

### Commits This Session

| Commit | Description |
|--------|-------------|
| `5781980` | `deploy: update obsidian-mcp-server image to point to ghcr.io` |
| `db7d1ff` | `feat(mcp): add Swarm deployment stacks for Syncthing and Obsidian MCP` |
| `aa7c4b2` | `fix(homepage): use 127.0.0.1 in healthcheck to prevent IPv6 connection failure` |
| *(Session 2)* | `fix(obsidian-mcp): remove VAULT_ALLOWED_FOLDERS — vault root is synced folder` |
| *(stack-traefik.yml)* | `traefik_docker-proxy restart policy: condition on-failure → any` |
| *(Runbook)* | Gotcha #50: acme.json must be 600 or Traefik skips ACME silently |

---

---

## Session 3

### Completed

- **PR #52 merged** — `fix(traefik): docker-proxy restart_policy → any` and `fix(obsidian-mcp): remove VAULT_ALLOWED_FOLDERS` both merged to `main` on `scarredNinja/docker-swarm-home`.
- **Portainer migration** — both stacks removed from CLI management and re-deployed via Portainer pointing to GitHub repo (`main` branch):
  - `dev-syncthing` → `proxmox-swarm/stacks/stack-syncthing.yml`
  - `dev-obsidian-mcp` → `proxmox-swarm/stacks/stack-obsidian-mcp.yml`
  - Stack names prefixed with `dev-` to indicate dev environment placement
- **Portainer pull-and-redeploy** — `dev-obsidian-mcp` re-synced from `main` after PR #52 merge to align Portainer's stored config with running state.
- **Fixed Homepage healthcheck IPv6 failure** — `stack-homepage.yml` healthcheck was using the container hostname, which resolved to an IPv6 address on dual-stack Docker networks, causing the health probe to fail. Changed to `127.0.0.1` to force IPv4. Commit `aa7c4b2`.

---

## Session 4

### Session Goal

Fix obsidian-mcp-server write tools — `VAULT_WRITE_FOLDERS` was set to a non-existent subfolder path, blocking all write attempts with EACCES. Added wildcard support to the server source and updated the stack to use `VAULT_WRITE_FOLDERS=*`. Also authored a comprehensive homelab roadmap note via the now-working MCP write tools.

### Completed

- **Created homelab roadmap note** — `Homelab Next Phase — Review & Roadmap.md` written to vault via `obsidian_create_note` tool after confirming current project state against all memory files and vault documents.
- **Diagnosed `VAULT_WRITE_FOLDERS` EACCES bug** — env var was set to `10 - Projects` (a Windows Obsidian subfolder name), but inside the container the vault is mounted flat at `/vault/`. The write service resolved `10 - Projects` as `/vault/10 - Projects/` which does not exist — causing all write attempts to fail with `EACCES`. PR #55 set `VAULT_WRITE_FOLDERS` to empty string as interim fix (merged).
- **Added wildcard `*` support to `obsidian-mcp-server` `src/services/write.ts`** — flat vaults with no subfolder structure can now set `VAULT_WRITE_FOLDERS=*` to allow writes anywhere under the vault root. Committed and merged as PR #56 (`feat(obsidian-mcp): enable write tools — remove :ro mount, add VAULT_WRITE_FOLDERS` + subsequent UID fix).
- **Updated `stack-obsidian-mcp.yml`** — set `VAULT_WRITE_FOLDERS=*` and switched container to run as UID 1000 to match Syncthing vault ownership (commits `b5ef37b`, `a3ef137`, `f563a0e`).
- **Built and pushed new image** — `ghcr.io/scarredninja/obsidian-mcp-server:latest` published with wildcard write support via GitHub Actions CI.
- **Confirmed MCP write working end-to-end** — note created in vault, synced to Windows via Syncthing, visible in Obsidian.

### Commits

| Commit | Description |
|--------|-------------|
| `b5ef37b` | `feat(obsidian-mcp): enable write tools — remove :ro mount, add VAULT_WRITE_FOLDERS (#53)` |
| `a3ef137` | `fix(obsidian-mcp): set VAULT_WRITE_FOLDERS to empty for flat vault root (#55)` |
| `f563a0e` | `fix(obsidian-mcp): run container as UID 1000 to match Syncthing vault ownership` |

---

## Open / Next Session

- [ ] Confirm `obsidian_search` and other MCP tools returning results consistently (initial verification done — monitor over time)
- [x] Push `obsidian-mcp-server` source to `scarredNinja/obsidian-mcp-server` GitHub repo (public, standalone) — already done; CI/CD publishing to `ghcr.io`
- [x] Set up GitHub Actions CI for `ghcr.io` image builds — already live (`ci: add GitHub Actions workflow`)
- [ ] Fix Pi-hole wildcard DNS: add `address=/.home.purvishome.com/10.0.60.40` to `/etc/dnsmasq.d/02-homelab-wildcard.conf`
- [ ] Fix inter-VLAN SSH: pfSense TCP/22 rule (VLAN 10→VLAN 60) still pending from Alertmanager session
- [ ] Arr stack Docker secrets creation + redeploy (Docker secrets `_v2` not yet live)

---

---

## Session 5

### Session Goal

Update Claude Code agent definitions (`obsidian-search.md`, `obsidian-editor.md`, `session-note-writer.md`) to use MCP Obsidian tools natively instead of falling back to Glob/Grep/Read/filesystem operations for vault queries and writes.

### Completed

- **Updated `obsidian-search.md` agent** — rewrote vault query instructions to use `mcp__obsidian__obsidian_*` tools (`obsidian_search`, `obsidian_find_by_tag`, `obsidian_get_tasks`, etc.) instead of Glob/Grep/Read for all vault lookups.
- **Updated `obsidian-editor.md` agent** — rewrote write instructions to prefer MCP write tools (`obsidian_create_note`, `obsidian_append_to_note`, `obsidian_update_frontmatter`); filesystem Edit retained only for surgical in-body task checkbox changes.
- **Updated `session-note-writer.md` agent** — migrated each step to use the appropriate MCP tool: `obsidian_find_by_frontmatter` (Step 3 — last session date), `obsidian_get_tasks` (Step 4 — open task scan), `obsidian_list_notes` (Step 5 — existing note check), `obsidian_create_note`/`obsidian_append_to_note` (Step 6 — write/append), with filesystem Edit retained only for task checkbox marking (Step 7).
- **Identified `.claude/agents/` write hard block** — writes to `.claude/agents/` are blocked in auto mode by the harness; user manually pasted `session-note-writer.md` content as a workaround. Noted for future sessions.

### Notes

- The `.claude/agents/` directory is write-protected in auto mode. Agent file updates require either manual paste or an explicit permission grant.
- No phase hub tasks matched this session — agent tooling updates are not currently tracked as hub tasks.

---

## Related Notes

- [[VM - dev-node-01]]
- [[Service - obsidian-mcp-server]]
- [[Docker Swarm Infrastructure Runbook]]
- [[Swarm Topology]]
