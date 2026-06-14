---
due_date: '2026-05-30T00:00:00.000Z'
phase: 'Phase 9: External Access'
priority: High
project_id: Homelab-2025
status: Completed
tags:
  - project
  - external-access
  - vpn
  - homelab
---
#  Phase 9: External Access & Admin VPN

Secure remote admin access without exposing services publicly. This homelab is behind ISP CGNAT — inbound port forwarding is impossible. Solution: **Tailscale** on pfSense as a subnet router for `10.0.60.0/25` (VLAN 60 / Infrastructure), with DERP relay for CGNAT traversal.

> [!note] WireGuard/OpenVPN plans from original phase spec are superseded by Tailscale. Cloudflare Tunnel handles the one public-facing service (Plex).

##  Goals

- Off-site admin access to VLAN 60 services (Traefik, Portainer, Grafana, etc.)
- No inbound ports, no public exposure of admin services
- Split-tunnel: only `10.0.60.0/25` routes via VPN, not all device traffic

##  Related Notes

- [[Session Notes  2026-05-07  Tailscale Admin VPN]] — session record, blocked diagnostic steps
- [[Docker Swarm Infrastructure Runbook]] — Phase 9 section
- `docs/admin-vpn.md` in [scarredNinja/docker-swarm-home](https://github.com/scarredNinja/docker-swarm-home) — architecture, checklist, security notes
- [[Service - Termix]] — web-based SSH portal deployment & migration guidelines

## Current Status (2026-05-07)

| Item | Status |
|---|---|
| Tailscale installed on pfSense | |
| Subnet route `10.0.60.0/25` advertised + approved | |
| pfSense firewall rule: Tailscale -> VLAN 60 | |
| Traefik `internal-only` + `100.64.0.0/10` | Deployed |
| Phone -> VLAN 60 routing | Resolved 2026-05-08 |

##  Action Items

```dataviewjs
//  Tasks for this phase only 
// Uses dv.current().phase so this block is identical across every phase hub.
// It will only show tasks from files whose `phase` frontmatter matches this file's.
const PRIORITY_FALLBACK = 999;
const thisPhase = dv.current().phase;

if (!thisPhase) {
    dv.paragraph(" No `phase` field found in this file's frontmatter.");
} else {
    const pages = dv.pages('"10 - Projects"')
        .where(p =>
            p.phase === thisPhase &&
            !p.file.folder.includes("_Archive")
        );

    const tasks = pages
        .flatMap(p => p.file.tasks
            .where(t =>
                !t.completed &&
                !t.tags.includes("#Later") &&
                !t.tags.includes("#MuchLater")
            )
            .map(t => ({ task: t, file: p.file }))
        )
        .array();

    if (tasks.length === 0) {
        dv.paragraph(" No open tasks for this phase.");
    } else {
        tasks.sort((a, b) => {
            const pa = a.task.priority !== undefined ? Number(a.task.priority) : PRIORITY_FALLBACK;
            const pb = b.task.priority !== undefined ? Number(b.task.priority) : PRIORITY_FALLBACK;
            if (pa !== pb) return pa - pb;
            if (a.file.name !== b.file.name) return a.file.name.localeCompare(b.file.name);
            return (a.task.line ?? 0) - (b.task.line ?? 0);
        });

        dv.taskList(tasks.map(t => t.task), false);
    }
}
```

- [x] Install Tailscale on pfSense, advertise `10.0.60.0/25` subnet route — 2026-05-07
- [x] Approve subnet route in Tailscale admin console — 2026-05-07
- [x] pfSense firewall rule: Tailscale interface -> VLAN 60 — 2026-05-07
- [x] Traefik `internal-only` middleware: add `100.64.0.0/10` (Tailscale IP range) — 2026-05-07
- [x] Diagnose phone routing — 2026-05-08
- [x] End-to-end test confirmed — Tailscale deployed and routing to VLAN 60 — 2026-05-08
- [ ] Configure Tailscale ACL: restrict to `10.0.60.0/25` only, no other VLANs [priority:: 2]
- [x] Enrol remaining admin devices (laptop) [priority:: 2]
- [x] Correct access to synology [priority:: 1]
- [x] Deploy Termix stack (`stack-termix.yml`) to Swarm [priority:: 1]
- [x] Export connections from Termius and map to Termix JSON schema [priority:: 2]
- [x] Bulk import connections into Termix and verify access [priority:: 2]

##  Spoke Notes & Documentation
* [[Session Notes  2026-05-07  Tailscale Admin VPN]]
* `docs/admin-vpn.md` in repo

---
##  Next Phase
* [[01 Homelab Rebuild - Phase 10 Documentation Hub]]