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
  - cloudflared
  - Portainer
  - ExternalAccess
service_status: deployed
---

# Session Notes — 2026-05-08 — cloudflared Tunnel Token Persistence

## Session Goal

Fix `plex_cloudflared` service stuck at 0/1 replicas with `Update paused due to failure`, and persist the tunnel token so the stack survives reboots / fresh ssh sessions / arbitrary redeploy paths.

---

## Diagnostic Path

| Observation | Conclusion |
|---|---|
| `plex_cloudflared` at 0/1, "Update paused due to failure" | Container exiting fast on every replica restart |
| `docker service inspect plex_cloudflared` → `Args: tunnel --no-autoupdate run --token` | Token argument interpolated to **empty string** at deploy time |
| Container exit code 255 | cloudflared exits with 255 when given empty token |
| `${CLOUDFLARE_TUNNEL_TOKEN}` not set in operator shell | Compose was reading from the deploying shell — found the source-of-truth gap |

> [!bug] Token was never persisted on the manager
> `command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}` interpolates from the **deploying operator's shell**, not from any persistent store. Fresh ssh sessions, reboots, or Portainer redeploys without the env var would silently produce a broken stack.

---

## Bug #59 — cloudflared distroless image breaks entrypoint shim approach

> [!bug] `cloudflare/cloudflared:latest` is distroless — no `/bin/sh`, no busybox, nothing but the binary
> First attempted fix used a Docker secret + entrypoint shim:
> ```yaml
> entrypoint: ["sh", "-c", 'exec cloudflared tunnel --no-autoupdate run --token "$$(cat /run/secrets/cf_tunnel_token)"']
> ```
> Container failed with `exec: "/bin/sh": stat /bin/sh: no such file or directory`. Any shell-based wrapper around cloudflared in the official image is impossible.

See Gotcha #59 in [[Docker Swarm Infrastructure Runbook]] Appendix D.

---

## Completed

### 1. Failed approach — Docker secret + entrypoint shim (reverted)

| PR | Commit | Status |
|---|---|---|
| [#26](https://github.com/scarredNinja/docker-swarm-home/pull/26) | `8917230` — fix(plex): pass cloudflare tunnel token via Docker secret | Superseded |
| [#27](https://github.com/scarredNinja/docker-swarm-home/pull/27) | `5b87840` — fix(plex): escape $ in cloudflared entrypoint | Superseded |
| [#28](https://github.com/scarredNinja/docker-swarm-home/pull/28) | `c6f6318` — revert: cloudflared Docker-secret approach | **Landed** |

### 2. Final fix — Portainer stack environment variable

Kept original `--token ${CLOUDFLARE_TUNNEL_TOKEN}` form. Persisted the variable in **Portainer's stack environment variables** (Stacks → plex → Environment variables).

| Property | Value |
|---|---|
| Storage | Portainer's encrypted DB |
| Interpolation | Always at Portainer deploy time, regardless of operator shell |
| Survives | Manager reboots, fresh ssh sessions, Git webhook redeploys, UI redeploys |

Portainer is already the deploy authority for the plex stack (Git-managed) — this matches the existing operational model from [[Session Notes — 2026-05-08 — Monitoring Fixes, unifi-poller, Git Integration]].

### 3. Stale Docker secret cleanup

`cf_tunnel_token` Docker secret no longer referenced — safe to remove:

```
docker secret rm cf_tunnel_token
```

> [!warning] Two ways this pattern can still fail
> 1. Stack deployed via `docker stack deploy` from a shell that lacks the var → silent empty interpolation.
> 2. Portainer stack env var deleted → same silent failure.
>
> **Mitigation (not yet implemented):** healthcheck that fails fast on missing/empty token. Worth considering as a follow-up if this bites again.

---

## Verified

- `plex_cloudflared` 1/1 replicas
- 4× "Registered tunnel connection" in container logs
- Plex external access functional through Cloudflare Tunnel

---

## Current State (End of Session)

| Stack | Status |
|---|---|
| `plex` | ✅ All services 1/1; cloudflared tunnel up |

| Asset | State |
|---|---|
| `CLOUDFLARE_TUNNEL_TOKEN` (Portainer plex stack env var) | ✅ Set |
| `cf_tunnel_token` Docker secret | 🗑️ Stale, safe to remove |

---

## Key Takeaways

- **`cloudflare/cloudflared:latest` is distroless** — no shell, no debug tools. Cannot wrap with `sh -c '...'` entrypoints. Only path for shell-based wrappers is to build a thin custom image (e.g. alpine + copied cloudflared binary). Not pursued — Portainer env var is sufficient.
- **For env-var-interpolation in compose stacks, the source-of-truth must be persistent.** Operator shell is not a source of truth. Acceptable persistent stores in this homelab: Portainer stack env vars, virtiofs-mounted env files, Docker secrets (with caveat: only consumable by non-distroless images via shim, or by services that natively support `_FILE` env var conventions — see also Gotcha #57).

---

## Next Session Priorities

- [x] Remove stale `cf_tunnel_token` Docker secret on manager: `docker secret rm cf_tunnel_token` [priority::2] ✅ 2026-05-19
- [ ] Consider healthcheck on cloudflared service that fails fast on empty/missing token (follow-up if this bites again) [priority::2]
- [ ] Audit other stacks for the same `${VAR}` interpolation pattern with shell-only source-of-truth

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]] — Gotcha #59 (cloudflared distroless + token persistence), External Access (Cloudflare Tunnel) section
- [[Service - plex]]
- [[Session Notes — 2026-05-08 — Monitoring Fixes, unifi-poller, Git Integration]] — Portainer Git integration rollout
- [[01 Homelab Rebuild - Phase 9 External Access Hub]]
