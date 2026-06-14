---
date: '2026-04-21'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm — HTTPS Fix & Plex External Access'
session_type: Diagnose + Fix
status: Completed
tags:
  - SessionNotes
  - DockerSwarm
  - Traefik
  - Networking
  - Plex
  - HTTPS
---
# Session Notes — 2026-04-21

## 🎯 Session Goal

Diagnose why HTTPS access through Traefik was failing intermittently, resolve the root causes, and progress Plex external access to a point where a browser end-to-end test is possible.

---

## 🔍 Diagnostic Path

| Step | Finding | Outcome |
|------|---------|---------|
| Check ACME certs in acme.json | 15 valid certs present — cert issuance not the problem | ✅ Certs OK |
| DNS resolution via Cloudflare | `plex.purvishome.com` resolving correctly to WAN IP | ✅ DNS OK |
| Test HTTPS from multiple clients | ~50% of connections succeeded; failures were consistent TLS handshake never completing | 🔴 Routing asymmetry suspected |
| Check routing table on traefik-dmz-01 | Two default routes present at equal metric 100 — `eth0` VLAN 60 and `enp6s19` VLAN 80 | 🔴 Bug #21 found |
| Delete VLAN 80 default route, re-test | HTTPS now 100% reliable | ✅ Routing fix confirmed |
| Test service-to-service connectivity | Containers on different nodes couldn't reach each other despite overlay networks existing | 🔴 Overlay subnet collision suspected |
| Inspect overlay subnet assignments | `monitoring_monitoring` → `10.0.4.0/24` (collision with workstation LAN); `traefik-public` → `10.0.2.0/24` | 🔴 Bug #22 found |
| Redeploy all stacks with pinned subnets | Service-to-service connectivity restored | ✅ Overlay fix confirmed |

---

## 🐛 Bug #21 — Asymmetric Routing (traefik-dmz-01)

> [!bug] Root cause — dual default routes at equal metric
> `traefik-dmz-01` is dual-homed: `eth0` on VLAN 60 (10.0.60.40) and `enp6s19` on VLAN 80 (10.0.80.20). Both interfaces received a default gateway via DHCP at metric 100. Linux ECMP (Equal-Cost Multi-Path) split outbound traffic across both routes — meaning ~50% of SYN-ACK replies left via the DMZ interface (`enp6s19`, VLAN 80) instead of returning via VLAN 60 through pfSense. The client never received the handshake completion and the TLS connection timed out.

**Temporary fix applied (2026-04-21):** Deleted the VLAN 80 default route — does not survive a reboot.

> [!warning] Permanent fix outstanding
> The VLAN 80 default route will return on next reboot or DHCP renewal. Permanent fix: add `dhcp4-overrides: use-routes: false` to the `enp6s19` interface stanza in netplan on `traefik-dmz-01`. This is the same pattern already applied to media VMs on VLAN 100 (see [[Docker Swarm Infrastructure Runbook]] Gotcha #25).

---

## 🐛 Bug #22 — Overlay Network Subnet Collision

> [!bug] Root cause — Docker auto-assigned overlay subnets colliding with physical VLANs
> Docker assigns overlay network subnets automatically from the `10.0.0.0/8` address space. Since physical VLANs also use `10.0.x.x`, Docker's auto-assigned subnets collided with live network ranges. Packets destined for physical hosts were being routed into Docker's virtual overlay network instead.

**Affected networks before fix:**

| Network | Old Subnet (auto) | Conflicted With | New Subnet (pinned) |
|---------|-------------------|-----------------|---------------------|
| `monitoring_monitoring` | `10.0.4.0/24` | Workstation LAN | `10.200.1.0/24` |
| `traefik-public` | `10.0.2.0/24` | Physical range | `10.200.2.0/24` |
| `traefik_traefik-backend` | (auto) | Potential conflict | `10.200.3.0/24` |

**Fix:** All stacks redeployed with subnets pinned in stack YAML under `networks.<name>.ipam.config.subnet`. All overlay networks now use the `10.200.0.0/16` range which does not overlap any physical VLAN.

Changes committed as `eafa48b` on branch `claude/happy-cartwright-f38701`, merged to main.

---

## 🔐 Plex External Access — In Progress

| Item | Status | Notes |
|------|--------|-------|
| ACME certs | ✅ Valid | 15 certs confirmed in acme.json |
| Cloudflare DNS | ✅ Configured | `plex CNAME → ddns.purvishome.com` (DNS only mode) |
| Public DNS resolution | ✅ Resolving | `plex.purvishome.com` resolves via 1.1.1.1 |
| pfSense port forward | ⚠️ Needs verification | External 443 → `traefik-dmz-01` — not yet confirmed |
| Local PiHole | ⚠️ Cached negative | Flush or wait for TTL expiry |
| Browser end-to-end test | 🔲 Not done | **Next session** |

> [!note] PiHole cached negative response
> Local DNS may still be returning a cached NXDOMAIN for `plex.purvishome.com` from before the Cloudflare record was added. Flush PiHole cache or wait for TTL expiry before testing internally. External test (mobile data) is unaffected.

---

## 📊 Current State (End of Session)

### Swarm Nodes

| Node | Role | Status | VLAN |
|------|------|--------|------|
| `manager-01` (10.0.60.30) | Leader | Ready | 60 |
| `traefik-dmz-01` (10.0.60.40) | Worker | Ready | 80 + 60 |
| `worker-monitoring-01` (10.0.60.41) | Worker | Ready | 60 |
| `worker-media-01` | Worker | Ready | 50 + 100 |
| `worker-mediamanagement-01` | Worker | Ready | 50 + 100 |

### Stacks

| Stack | Status | Notes |
|-------|--------|-------|
| `portainer` | ✅ Running | |
| `traefik` | ✅ Running | HTTPS confirmed working; overlay subnets pinned |
| `monitoring` | ✅ Running | Overlay subnet pinned `10.200.1.0/24` |
| `plex` | 🔄 In progress | Stack running; external access DNS configured, browser test next session |
| `arr` | ⚠️ Partial | Sonarr port bind issue carry-over from 2026-04-19 |

---

## Next Session Priorities

- [x] **Asymmetric routing netplan fix consolidated** — tracked on master VM page [[VM - traefik-dmz-01]] ✅ 2026-06-01
- [x] **Verify pfSense port forward** — external 443 → `traefik-dmz-01` — confirm rule is active and NAT is correct ✅ 2026-05-01
- [x] **Flush PiHole cache** — or verify TTL has expired, then test `plex.purvishome.com` locally ✅ 2026-04-30
- [x] **Plex browser test end-to-end** — `https://plex.purvishome.com` through Cloudflare → pfSense → Traefik → Plex ✅ 2026-04-30
- [x] **Fix Sonarr port bind (carry-over)** — check `AuthenticationMethod` in config.xml, change to `Forms` if Sonarr v4 ✅ 2026-04-30
- [x] **Fix Sonarr → Transmission connection (carry-over)** — update download client host to Gluetun service name ✅ 2026-04-30

---

## 🔗 Related Notes

- [[Docker Swarm Infrastructure Runbook]]
- [[VM - traefik-dmz-01]]
- [[Service - plex]]
- [[Service - traefik]]
- [[VLAN and Subnet Summary Sheet]]
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
