---
date: '2026-05-08T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
session_type: Diagnose + Fix
status: Completed
tags:
  - SessionNotes
  - DockerSwarm
  - Plex
  - NFS
  - autofs
  - BindMounts
---

# Session Notes — 2026-05-08 — Plex Media Mount Fix

## Session Goal

Fix Plex container seeing empty `/media/Movies`, `/media/TVShows`, `/media/Animation` despite NFS shares being mounted on `worker-media-01`.

---

## Diagnostic Path

| Observation | Conclusion |
|---|---|
| Plex libraries showed zero content for all three categories | Container can't see media data |
| `ls /mnt/media/Movies` on the host returned the full library | NFS mounts on the host are healthy |
| `docker exec plex ls /media/Movies` returned empty | Bind mount propagation isn't reaching the container |
| Host mount inspection: `/mnt/media/{Movies,TVShows,Animation}` are autofs direct-mount triggers, not statically-mounted NFS | Each share is mounted only when accessed via its trigger path |
| Stack bind: `/mnt/media:/media:ro` (parent path) | Docker captured the autofs trigger directories but not the underlying NFS mounts |

> [!bug] Docker default bind propagation (`rprivate`) doesn't follow autofs triggers
> When Docker resolves a bind mount with `rprivate` propagation, it captures the source path's namespace at mount time. autofs triggers ARE captured (they're directories), but the dynamic NFS submounts they fire on access are NOT.

---

## Bug #60 — Bind-mounting parent of autofs triggers yields empty containers

> [!bug] Stack bind-mounted `/mnt/media` parent; autofs sub-triggers were invisible to the container

> [!warning] Rule for any future bind mount
> Whenever a source path contains dynamically-triggered mounts (autofs, systemd `.automount` units, etc.), bind each trigger **directly** — never its parent. Docker's default propagation does not follow triggers.

See Gotcha #60 in [[Docker Swarm Infrastructure Runbook]] Appendix D.

---

## Completed

### 1. Stack change — `stack-plex.yml`

Replaced single parent bind with one bind per NFS share:

```yaml
volumes:
  - /mnt/media/Movies:/media/Movies:ro
  - /mnt/media/TVShows:/media/TVShows:ro
  - /mnt/media/Animation:/media/Animation:ro
```

Container-side paths under `/media/<share>` are unchanged → no Plex library reconfiguration needed. Same pattern already in `stack-arr.yml` — plex stack predated it.

### 2. Verified

- All three libraries populate inside the container
- Plex DB sees the existing files at the original paths — no library rescan needed

### 3. Considered alternative — `bind.propagation: rshared`

Would make the container observe new submounts as they appear, but requires the source to be a shared mount on the host and is more invasive than per-share binds. Not pursued.

### 4. PR + commit

PR [#30](https://github.com/scarredNinja/docker-swarm-home/pull/30) merged.

| Commit | Message |
|---|---|
| `ffd7d90` | fix(plex): bind each NFS share directly under /media |

---

## Follow-up parked

- [ ] Remove stale `/mnt/media/tvshows` (lowercase) autofs trigger on `worker-media-01` — no backing NFS export, not blocking anything

---

## Current State (End of Session)

| Stack | Status |
|---|---|
| `plex` | ✅ All three libraries visible; cloudflared tunnel still up from earlier session |

---

## Key Takeaways

- **Bind-mounting a parent of autofs triggers gives empty containers.** Always bind-source the actual mount point, not its parent.
- The arr stack got this right out of the gate; the plex stack predated the pattern. Worth a one-time audit of any other stacks that bind under `/mnt/media` or similar autofs-driven paths.

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]] — Gotcha #60 (autofs + Docker bind propagation)
- [[Service - plex]]
- [[VM - worker-media-01]]
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
