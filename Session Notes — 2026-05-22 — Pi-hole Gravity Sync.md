---
project_id: Homelab-2025
date: '2026-05-22T00:00:00.000Z'
phase: 'Phase 3: Network Config'
tags:
  - session-note
  - homelab
  - pihole
  - gravity-sync
status: Completed
---

# Session Notes — 2026-05-22 — Pi-hole Gravity Sync

## Summary

Implemented Gravity Sync between the two Pi-hole instances (`pihole1` at `10.0.60.20`, `pihole2` at `10.0.60.21`). Worked through several Gravity Sync 4.x install quirks before getting a clean working setup. Corrected the IP documentation (was incorrectly recorded as `.10`/`.11`). Two follow-up tasks identified: Traefik routing for the Pi-holes and node_exporter for Prometheus scraping.

---

## Completed

- **Corrected Pi-hole IPs** — both Pis confirmed on VLAN 60: pihole1 = `10.0.60.20`, pihole2 = `10.0.60.21` (previous docs had `.10`/`.11`)
- **Gravity Sync 4.x installed on pihole1 and pihole2** — cloned from GitHub, copied the `gravity-sync` binary (not `gravity-sync.sh` which is the v3→v4 migration utility), copied templates to `/etc/gravity-sync/.gs/templates/`
- **Gravity Sync configured on pihole2** — ran `gravity-sync config` interactive wizard; SSH key registered to pihole1, local and remote Pi-hole installs auto-detected
- **Initial sync completed** — `gravity-sync push` from pihole2 pushed pihole1's full config across; `gravity-sync logs` confirmed success
- **Auto-sync enabled** — `gravity-sync auto` set up systemd timer for ongoing replication
- **Vault docs updated** — `Pi-Hole Installation Steps.md` fully rewritten with correct IPs, accurate v4 install steps, and v4 gotchas

---

## Gotchas / Lessons Learned

- `https://gravity.vmstan.com` shortlink is dead — install must be done via `git clone https://github.com/vmstan/gravity-sync`
- `gravity-sync.sh` in the repo root is the **v3→v4 migration utility**, not the installer — the actual binary is `gravity-sync` (no extension, 89KB)
- Templates must be manually copied to `/etc/gravity-sync/.gs/templates/` before `gravity-sync config` will run
- Initial sync is `gravity-sync push` (from secondary pointing at primary), not `pull`
- Auto-sync uses **systemd timers**, not crontab

---

## Open Tasks

- [ ] Traefik — add routing for pihole1 (`10.0.60.20`) and pihole2 (`10.0.60.21`)
- [ ] Investigate node_exporter deployment on pihole1 and pihole2 for Prometheus scraping

---

## Related Notes

- [[Pi-Hole Installation Steps]]
- [[01 Homelab Rebuild - Phase 3 Network Config Hub]]
- [[VLAN and Subnet Summary Sheet]]
