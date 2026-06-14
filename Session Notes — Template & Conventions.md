---
type: reference
project_id: Homelab-2025
tags:
  - session-note
  - homelab
last_updated: '2026-04-21T00:00:00.000Z'
status: Completed
phase: 'Phase 5: Docker Swarm'
---

# Session Notes — Template & Conventions

> [!abstract] Purpose
> This note defines the naming standard, frontmatter schema, and required sections for all session notes in this vault. Follow this every time a new session note is created to keep the Working Sessions table in [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]] consistent and the vault searchable.

---

## 📁 File Naming

**Standard:** `Session Notes — YYYY-MM-DD — Short Description.md`

| Rule | Detail |
|------|--------|
| Separator | Always ` — ` (space + em dash + space). Use `—` not `-` |
| Date | Always present, always `YYYY-MM-DD` format |
| Description | 2–5 words, title case, no special characters except `-` and `+` |
| Location | Always in `10 - Projects/` — never in vault root or subfolders |

**Examples:**
```
Session Notes — 2026-04-21 — HTTPS Fix Overlay Subnets.md   ✅
Session Notes — 2026-04-21 — Traefik + Portainer Deployment.md   ✅
Session Notes - 2026-04-21 - HTTPS Fix.md   ❌  (hyphens, not em dashes)
Session Notes 2026-04-21.md   ❌  (no separator, no description)
Session Notes — HTTPS Fix.md   ❌  (no date)
```

**Multiple sessions on the same day:** Add a disambiguating suffix to the description:
```
Session Notes — 2026-03-27 — Provisioning Scripts v2 Complete.md
Session Notes — 2026-03-27 — Provisioning Scripts Live Debug.md
```

---

## 📋 Frontmatter Schema

Copy this block exactly — fill in all fields:

```yaml
---
date: YYYY-MM-DD
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm — Brief Phase Description"
session_type: <see options below>
status: <see options below>
tags:
  - SessionNotes
  - <topic tags — e.g. Traefik, DockerSwarm, Networking, Plex>
---
```

**`session_type` options:**
- `Planning` — architecture decisions, no hands-on work
- `Deploy` — deploying stacks or VMs
- `Diagnose + Fix` — debugging session
- `Code` — Claude Code session writing scripts/config
- `Planning + Deploy` — mixed session

**`status` options:**
- `Complete` — all goals achieved
- `Partial — <reason>` — session ended before goals met, note the blocker
- `Blocked — <reason>` — couldn't proceed, note why

---

## 📝 Required Sections

Every session note must have these sections in this order:

```markdown
## 🎯 Session Goal
One paragraph — what you set out to do.

## ✅ Completed  (or ## 🔍 Diagnostic Path for debug sessions)
What was actually done. Use tables for structured findings.
State only — no shell commands (commands go in the runbook).

## 🐛 Bug #N — Title  (if applicable)
> [!bug] Root cause description
One bug = one section. Number sequentially from the runbook.
> [!warning] if a fix is incomplete or temporary

## 📊 Current State (End of Session)
Tables showing node status and stack status. Always include this.

## ➡️ Next Session Priorities
- [ ] Checkbox list — in priority order

## 🔗 Related Notes
Wikilinks to runbook, relevant VM/service notes, hub
```

---

## ✍️ Style Rules

| Rule | Detail |
|------|--------|
| Commands | **Never in session notes** — add to [[Docker Swarm Infrastructure Runbook]] instead |
| Bugs | Always cross-reference the runbook gotcha number (e.g. "See Gotcha #31") |
| Status | Use ✅ / 🔄 / ⚠️ / 🔲 consistently in tables |
| Callouts | `> [!bug]` for root causes · `> [!warning]` for outstanding fixes · `> [!note]` for gotchas · `> [!important]` for things that must not be forgotten |
| Wikilinks | Use `[[VM - hostname]]` and `[[Service - name]]` to link to individual notes |
| Done items | Mark with checkbox + date — **never delete** |

---

## 🔗 After Creating a Session Note

1. Add a row to the Working Sessions table in [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
2. Update any affected `VM - *.md` or `Service - *.md` frontmatter (`vm_status`, `service_status`, `last_updated`)
3. Add new gotchas to [[Docker Swarm Infrastructure Runbook]] Appendix D
4. Add outstanding tasks to Appendix F

---

## 📌 Instruction — Add to CLAUDE.md

> Copy the block below into the project `CLAUDE.md` under a `## Session Notes` heading so Claude Code follows this convention automatically.

```markdown
## Session Notes

When creating session notes:
- Filename: `Session Notes — YYYY-MM-DD — Short Description.md` (em dash separators, always in `10 - Projects/`)
- Frontmatter: date, project_id, phase, session_type, status, tags (see `Session Notes — Template & Conventions.md`)
- Sections in order: Session Goal → Completed/Diagnostic Path → Bug sections → Current State → Next Session Priorities → Related Notes
- State only — no shell commands in session notes; commands go in the runbook
- After creating: update Phase 5 hub Working Sessions table, update affected VM/service note frontmatter, add gotchas to runbook Appendix D
- Cross-reference runbook gotcha numbers for any bugs found (e.g. "See Gotcha #31")
- Use > [!bug], > [!warning], > [!note], > [!important] callouts consistently
```
