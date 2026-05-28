---
date: 2026-05-08
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
session_type: Diagnose + Fix
status: Complete
tags:
  - SessionNotes
  - DockerSwarm
  - compose-vpn
  - gluetun
  - Transmission
  - systemd
  - Provisioning
---

# Session Notes — 2026-05-08 — compose-vpn Auto-Start Fix

## Session Goal

Fix `compose-vpn.yml` (gluetun + transmission, plain Docker Compose on `worker-mediamanagement-01`) failing to come back up after VM reboot, and generalize the provisioning path so future plain-compose stacks attaching Swarm overlays can be installed the same way.

---

## Diagnostic Path

| Observation | Conclusion |
|---|---|
| Containers `Exited` after VM reboot, despite `restart: unless-stopped` | Docker daemon gave up restarting before the network was ready |
| `docker logs gluetun` showed `failed to attach to network arr_arr_default: context deadline exceeded` | Overlay attach was racing Swarm gossip |
| `docker network inspect arr_arr_default` succeeded *before* an actual container could attach | Inspect-existence is not a reliable readiness signal — first attach can still fail |
| Manual `docker compose up -d` minutes later succeeded every time | Issue is purely a boot-time race, not config |

> [!bug] Two-layer race on boot
> 1. `docker network inspect <overlay>` reports the network locally before Swarm has finished gossiping it from the manager.
> 2. Even after `inspect` succeeds, the *first* attach still fails until manager-node gossip fully completes.
>
> The only reliable readiness signal is a transient probe container actually attaching to each overlay successfully.

---

## Bug #58 — compose-vpn fails to auto-start after reboot

> [!bug] Plain-compose stack with `restart: unless-stopped` does not survive VM reboot when attaching Swarm overlays
> Docker's restart cap is hit before Swarm gossip completes. Adding more retries to compose alone does not help — the daemon-level restart logic is what gives up.

> [!warning] Rule for future stacks
> Any plain-compose stack on a Swarm node that attaches a Swarm overlay must use a systemd oneshot unit with probe-based overlay readiness. `restart: unless-stopped` alone is insufficient.

See Gotcha #58 in [[Docker Swarm Infrastructure Runbook]] Appendix D.

---

## Completed

### 1. New systemd unit — `compose-vpn.service`

| Property | Value |
|---|---|
| Type | `oneshot`, `RemainAfterExit=yes` |
| Order | `Requires=docker.service`, `After=docker.service network-online.target` |
| `ExecStartPre` | Wait for `Swarm.LocalNodeState=active` → both overlays inspectable → transient `alpine` probe attaches to each overlay |
| `ExecStart` | `docker compose -p compose-vpn -f /opt/compose-vpn/compose-vpn.yml up -d`, retried up to 5× with `down --remove-orphans` between attempts |
| Project name | Pinned to `compose-vpn` to prevent container-name conflicts when stack started from a different cwd |

### 2. Generalized provisioning path

So future plain-compose stacks slot in the same way without touching post-boot manually:

| File | Change |
|---|---|
| `02-provision-vm.sh` | New repeatable `--compose-unit <name>` flag → writes `compose_units` array to VM registry JSON |
| `03-post-boot.sh` | New `install_compose_units` step copies `/mnt/docker-swarm/stacks/<name>.yml` → `/opt/<name>/<name>.yml` and `/mnt/docker-swarm/systemd/<name>.service` → `/etc/systemd/system/<name>.service`, then `systemctl enable` (does NOT auto-start — env files / secrets must land first) |
| `copy-swarm-config.sh` | Now syncs `proxmox-swarm/systemd/*.service` → `/mnt/docker-swarm/systemd/` so unit files reach all VMs via virtiofs |

### 3. Gluetun config touch-up

| File | Change |
|---|---|
| `compose-vpn.yml` | Added `UPDATER_PERIOD: 24h` — gluetun's hardcoded NordVPN server list goes stale and surfaces as TLS handshake / EHOSTUNREACH errors |

### 4. NordVPN secret format documented

`/etc/gluetun/vpn-secrets.env` requires *service credentials* (not account login) — get from nordvpn.com → Services → NordVPN → Set up manually → Service credentials tab.

```
OPENVPN_USER=<service username>
OPENVPN_PASSWORD=<service password>
```

### 5. PR + commits

PR [#25](https://github.com/scarredNinja/docker-swarm-home/pull/25), branch `claude/mystifying-engelbart-998246`.

| Commit | Message |
|---|---|
| `c8f4b2c` | fix(compose-vpn): auto-start at boot via systemd unit |
| `eb9e5cf` | fix(compose-vpn): wait for swarm gossip + retry compose up at boot |
| `5ecce9b` | fix(compose-vpn): add UPDATER_PERIOD=24h for fresh NordVPN server list |

---

## Verified

- VM reboot → `compose-vpn.service` reaches `active (exited)`
- Both containers `Up`
- gluetun + transmission both exit via `180.149.231.200` (Auckland NordVPN)
- Transmission's traffic correctly bound to gluetun's network namespace
- `docker exec gluetun wget -qO- https://ipinfo.io/ip` returns NordVPN IP, not home WAN

> [!note] Transient `EHOSTUNREACH` / `TLS handshake failed` against NordVPN endpoints during connect is normal — gluetun rotates servers automatically.

---

## Current State (End of Session)

| Node | Status |
|---|---|
| `worker-mediamanagement-01` | ✅ `compose-vpn.service` enabled + active |

| Stack | Status |
|---|---|
| `compose-vpn` (gluetun + transmission) | ✅ Auto-starts at boot via systemd; survives reboot test |

---

## Next Session Priorities

- [ ] Apply same systemd-unit pattern if any future plain-compose stack is added on a Swarm node
- [x] Merge PR #25 to `main`
- [x] Replay on existing VM if recovering from a wipe: `sudo install -m 644 /mnt/docker-swarm/systemd/compose-vpn.service /etc/systemd/system/compose-vpn.service && sudo systemctl daemon-reload && sudo systemctl enable --now compose-vpn.service`

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]] — Gotcha #58 (Swarm overlay gossip race)
- [[Service - transmission]]
- [[VM - worker-mediamanagement-01]]
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
