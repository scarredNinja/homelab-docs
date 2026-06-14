---
tags:
  - decisions
  - architecture
  - homelab
last_updated: '2026-05-24T00:00:00.000Z'
project_id: Homelab-2025
status: Reference
phase: 'Phase 5: Docker Swarm'
---

# Architecture Decisions

Running log of significant decisions with rationale. Most recent first.

---

## 2026-05-24 — Git Hosting Strategy

| Repo | Host | Reason |
|---|---|---|
| `docker-swarm-home` | GitHub | Claude Code integration, PR workflow, off-site backup |
| `homelab-docs` | GitHub | Obsidian Git plugin targets GitHub |
| `obsidian-mcp-server` | GitHub | Shareable, public |
| Personal app repos | Gitea | Private, no need for external hosting |
| Docker images | Gitea registry | Local builds stay local |

Gitea's primary value is the **built-in container registry** and **Gitea Actions CI/CD** for personal apps — not replacing GitHub for existing repos.

---

## 2026-05-24 — Documentation Strategy

**Obsidian:** Personal knowledge management, project tracking, session notes, task backlog, phase hubs, decisions, gotchas. Remains unchanged.

**Gitea wiki (per repo):** Stable reference documentation. Architecture overviews, deployment guides, tool references, pfSense rule summaries, runbook quick reference. Audience-facing, not session-oriented.

**Key distinction:** Wiki content describes how the system works now. Obsidian captures how it got there.

**`docker-swarm-home` wiki candidates:** VM topology, VLAN map, stack deployment guide, Traefik routing conventions, pfSense rule reference.

**`obsidian-mcp-server` wiki candidates:** Tool reference, env var guide, Claude Desktop setup, Swarm deployment guide.

---

## 2026-05-24 — Google Antigravity Awareness

Announced at Google I/O 2026 (20 May 2026). Agent-first development platform, positioned as a direct alternative to Claude Code.

**Components:**
- **Antigravity 2.0** — standalone desktop app, parallel agents, scheduled tasks, background subagent workflows
- **Antigravity CLI** — terminal-based, replaces Gemini CLI (sunset: 18 June 2026)
- **Antigravity SDK** — build custom Gemini-powered agents
- **Managed Agents API** — isolated Linux environment per API call, persistent multi-turn state
- **CodeMender** — security agent for AI-generated code vulnerability detection

**MCP relevance:** Uses a shared protocol layer between local development and cloud deployment. The `obsidian-mcp-server` HTTP endpoint is compatible.

**Plan:** Install Antigravity CLI, connect to local MCP HTTP endpoint, compare workflow with Claude Code.

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]]
- [[01 Homelab Rebuild - Phase 10 Documentation Hub]]
- [[Service - obsidian-mcp-server]]
