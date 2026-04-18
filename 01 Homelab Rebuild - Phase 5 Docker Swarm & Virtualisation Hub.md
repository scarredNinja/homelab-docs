---
status: Active
priority: High
due_date: 2026-04-15
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
---
# 🔄 Phase 5: Docker Swarm & Virtualisation

Setting up the container orchestration platform to host your services, focusing on stability and scale.

## 🎯 Goals

* Deploy all 5 Swarm nodes (1 manager, 4 workers)
* Deploy Traefik to DMZ node and verify external Plex access
* Migrate legacy services from old Compose server via ZFS send/receive
* Verify cluster stability before Phase 6

---

## 🖥️ VM Status

```dataviewjs
const pages = dv.pages('"10 - Projects"')
  .where(p => p.type === "swarm-vm")
  .sort(p => p.vmid ?? 999, "asc");

const statusIcon = s => ({
  "not-created": "⬜",
  "provisioned": "🟡",
  "post-boot-complete": "🟠",
  "running": "🟢",
}[s] ?? "❓");

dv.table(
  ["VM", "VMID", "VLAN(s)", "IP", "Status", "Post-Boot", "Swarm"],
  pages.map(p => [
    p.file.link,
    p.vmid ?? "—",
    [p.vlan_primary, p.vlan_secondary].filter(Boolean).join(" + "),
    p.ip_primary ?? "—",
    statusIcon(p.vm_status) + " " + (p.vm_status ?? "—"),
    p.post_boot_run ? "✅" : "🔲",
    p.swarm_joined ? "✅" : "🔲",
  ])
);
```

---

## 🐳 Service Status

```dataviewjs
const pages = dv.pages('"10 - Projects"')
  .where(p => p.type === "swarm-service")
  .sort(p => p.service_name, "asc");

const statusIcon = s => ({
  "pending":   "⬜",
  "deploying": "🟡",
  "running":   "🟢",
  "degraded":  "🔴",
}[s] ?? "❓");

dv.table(
  ["Service", "VM", "Status", "Port", "External", "Dataset"],
  pages.map(p => [
    p.file.link,
    p.vm ?? "—",
    statusIcon(p.service_status) + " " + (p.service_status ?? "—"),
    p.port ?? "—",
    p.external_access ? "🌐 Yes" : "🔒 No",
    p.zfs_dataset ?? "—",
  ])
);
```

---

## 🔗 Open Tasks



```dataviewjs
const PRIORITY_FALLBACK = 999;
const thisPhase = dv.current().phase;

const pages = dv.pages('"10 - Projects"')
  .where(p =>
    p.phase === thisPhase &&
    !p.file.folder.includes("_Archive") &&
    p.type !== "swarm-vm" &&
    p.type !== "swarm-service"
  );

const tasks = pages
  .flatMap(p => p.file.tasks
    .where(t => !t.completed)
    .map(t => ({ task: t, file: p.file }))
  )
  .array();

if (tasks.length === 0) {
  dv.paragraph("✅ No open tasks for this phase.");
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
```

> [!note] Ordered work lives in [[Docker Swarm Infrastructure Runbook]] phase checklists.
> Current blockers and next actions: [[Swarm Topology]]

---


- [ ] Fix Obsidian Bases files — Swarm VM Board and Swarm Service Table parsing errors [priority:: 2]
## 📝 Reference Docs

### Architecture
- [[Swarm Topology]] ← living dashboard, current blockers and next actions
- [[Docker Swarm Infrastructure Runbook]] ← ordered phase checklists
- [[Traefik Routing Architecture]] ← entrypoints, service split, VPN access model
- [[Docker Swarm Details]] ← service placement and storage matrix
- [[ZFS Configuration and Setup]]
- [[VLAN and Subnet Summary Sheet]]

### Monitoring
- [[Monitoring Docker Swarm]]
- [[z_Archived-Docker Swarm Monitoring]]

### Networking
- [[Netbox Setup]]
- [[Docker Swarm Scaling Strategy]]

---

## 📋 Working Sessions (chronological)

| Session                                                               | Date       | Key outcome                                                                                                                                                                                       |
| --------------------------------------------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [[Session Notes - VM Template Debugging & Fixes]]                     | 2026-03-19 | SSH key setup, template bugs, overlay2→fuse-overlayfs                                                                                                                                             |
| [[Session Notes - DataSourceNoCloud Deep Dive Dive Template Fix]]     | 2026-03-23 | NBD fix for 99-pve.cfg, DataSourceNoCloud root cause                                                                                                                                              |
| [[Session Notes - Mix of Claude and and GPT]]                         | 2026-03-24 | Interface naming, multi-NIC, netplan timing                                                                                                                                                       |
| [[Session Notes - Mix of Claude and and GPT]]                         | 2026-03-27 | virtiofs mapping ID fix, multi-NIC support, pfSense DHCP prompt                                                                                                                                   |
| [[Session Notes - Provisioning Scripts v2 v2 Complete]]               | 2026-03-27 | Scripts v2 complete, ZFS zvol backup gap identified                                                                                                                                               |
| [[Session Notes - Provisioning Scripts Live Debug Fix]]               | 2026-03-27 | virtiofs syntax, idx++ bug, qm agent jq fix, VLAN 60 routing                                                                                                                                      |
| [[Session Notes - Post-Boot Improvements Zfs Tuning Traefik_Stack]]   | 2026-03-30 | 03-post-boot v2, ZFS tuning scripts, Traefik v3 stack                                                                                                                                             |
| [[Session Notes - DockerSwarm Pipeline Fixes 1]]                      | 2026-03-31 | Pipeline bugs fixed, Portainer Swarm mode, manager-01 + traefik-dmz-01 provisioned                                                                                                                |
| [[Session Notes - virtiofs Fix Permissions Provisioning Script Bugs]] | 2026-04-02 | daemon.json path bug, fstab missing, ZFS permissions, DHCP blocker                                                                                                                                |
| [[Session Notes — Phase 0 Complete, Swarm Nodes & Portainer]]         | 2026-04-05 | Finished most of Phase 0 (beside PBS), deployed manager and traefik vms                                                                                                                           |
| [[Session Notes- Traefik Config Docker Fixes]]                        | 2026-04-07 | Worked on and fixed traefik configuration files with Claude code.                                                                                                                                 |
| [[Session Notes - Traefik Deployment]]                                | 2026-04-08 | Worked on docker swarm issues and then traefik deployment with Claude code                                                                                                                        |
| [[Session Notes — 2026-04-10 — Traefik + Portainer Deployment]]       | 2026-04-10 | Long session covering Swarm manager recovery, virtiofs permission fixes, and full<br>deployment of Traefik + Portainer with working HTTPS routing via Cloudflare DNS<br>challenge wildcard certs. |
| [[Session Notes 2026-04-12]]                                          | 12/04/2026 | Worked on and fixed a swarm init error. Redployed portainer and traefik and diagnosed some traefik issues                                                                                         |
| [[Session Notes - 2026-04-12 Monitoring Deploy]]                        | 12/04/2026 | Corrected and migrated how docker information and setup is installed. Updated the scripts to reflect this. Deployed monitoring                                                                    |

---

## ➡️ Next Phase
* [[01 Homelab Rebuild - Phase 6 Service Deployment Hub]]
