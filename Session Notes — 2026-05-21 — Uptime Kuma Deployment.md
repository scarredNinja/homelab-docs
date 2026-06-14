---
project_id: Homelab-2025
date: 2026-05-21T00:00:00.000Z
tags:
  - session-note
  - homelab
phase: 'Phase 8: Basic Monitoring'
status: Completed
---

# Session Notes — 2026-05-21 — Uptime Kuma Deployment

## Completed

- Triage review — surveyed all in-flight projects and identified next-session priorities across arr/monitoring/homepage branches
- **PR #49 merged** — `fix(uptime-kuma): correct hostname, add swarm network label, add internal-only middleware`
  - Fixed wrong hostname in Traefik router label: `uptime.home.purvishome.com` → `uptime-kuma.home.purvishome.com`
  - Added `traefik.swarm.network=traefik-public` label to resolve multi-network container routing (ERR_CONNECTION_TIMED_OUT)
  - Added `internal-only@file` middleware to restrict external access via Tailscale/VLAN 60 only
- **Uptime Kuma Traefik routing confirmed working** — stack redeployed with PR #49 fixes, verified accessible at `https://uptime-kuma.home.purvishome.com`
- **Homepage dashboard reviewed** — branch `claude/elegant-lehmann-00bda3` inspected; deployment deferred to next session (Traefik route verification needed, `widgets.yaml` kuma slug TBD)

## Next Session

- Create Uptime Kuma public status page, note slug
- Update Homepage `widgets.yaml` kuma slug (`default` → actual slug)
- Deploy Homepage stack (`claude/elegant-lehmann-00bda3` branch)
- Open Homepage PR once routing verified

## Commits

- `19ada24` fix(uptime-kuma): correct hostname, add swarm network label, add internal-only middleware (#49)
- `5157067` feat(stacks): add stack-newtonfit.yml for personal fitness dashboard

## Related Notes

- [[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]]
- [[Service - uptime-kuma]]
