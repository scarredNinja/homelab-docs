---
due_date: '2026-04-30'
phase: 'Phase 3: Network Config'
priority: Low
project_id: Homelab-2025
status: Active
---
#  Phase 3: Network Config

The foundation phase: deploying the firewall, setting up routing, VLANs, and core network services.

##  Goals

* Deploy the firewall (pfSense/OPNsense) and configure all WAN/LAN interfaces.
* Implement all necessary VLANs on the switch and firewall.
* Deploy DHCP and DNS services (e.g., Pi-Hole).

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

### Basic pfSense Setup

- [x] **Assign Interfaces:**Assign WAN and LAN interfaces in pfSense  2025-08-04 #pfSense #Network 
- [x] **Create VLAN Interfaces:**Create VLAN interfaces for all VLANs (10,20,30,40,50,55,60,65,70,75,80,90)  2025-08-04 #pfSense #VLAN
- [x] **Configure DHCP Servers:**Configure DHCP for each VLAN with specified IP ranges  2025-08-04 #pfSense #DHCP
- [x] Apply networking rules  2025-08-04 #pfSense  #Firewall
- [x] Re-add WAN configuration  2025-08-04 #pfSense #WAN
- [x] Recheck all rules that have been added are correct  2025-08-04 #pfSense #Verification 
- [x] next steps [ ] [[Pi-Hole Installation Steps#Tasks]] -  [priority:: 1]  2026-02-24
- [x] Clean out test rules and set main ip to VLAN 90 [priority:: 1]  2026-02-24
- [ ] Update Firewall rules mappings [priority:: 3]

### DNS & Core Services

- [x]  **Set up DNS:**Configure DNS resolver/forwarder in pfSense (pihole) #pfSense #DNS   2025-11-03
- [x] Add in the Traeflik work - internal/external [priority:: 2] #pfSense #DNS  2026-02-14
- [x] Review and setup [[Network Phased Configuration#Phase 2 The Manager VM "Virtual Hardware"]] [priority:: 1]  2026-02-14

### Pi-hole DNS & Traefik Routing

- [x] Add Pi-hole wildcard DNS: `address=/.home.purvishome.com/10.0.60.40` in `/etc/dnsmasq.d/02-homelab-wildcard.conf` on both pihole1 and pihole2  resolves all `.home.purvishome.com` subdomains internally without individual A records [priority:: 1]  2026-05-26  added via Pi-hole UI local DNS record (`*.home.purvishome.com  10.0.60.40`)
- [ ] Add Traefik routing for pihole1 (`pihole1.home.purvishome.com`  `10.0.60.20`) and pihole2 (`pihole2.home.purvishome.com`  `10.0.60.21`) [priority:: 2] #Later
- [x] Add pfSense firewall rule: VLAN 10 ➔ VLAN 60 TCP/22 to allow inter-VLAN SSH admin access to management infrastructure [priority:: 2]

### VLAN Migration & Cleanup

- [x] Migrate IoT devices to new VLAN  2025-08-04 #Migration #IoT 
- [x] Migrate Infrastructure (NAS) to new VLAN  2025-08-04 #Migration #Infrastructure 
- [x] Migrate OldNetwork - AP to new VLAN #Migration #Network #AccessPoint  kekb8h  qt6du4  [priority:: 1]
- [x] Remove unused Alisis [priority::1] #pfSense #Cleanup  2026-01-15
- [x] Clean up and check off other tasks #Cleanup #Verification  kekb8h  2025-12-24
- [x] Clean up existing firewall rules [priority:: 1] #pfSense #Firewall #Cleanup  2025-12-24

### Advanced Firewall Configuration

- [x] Review and update firewall rules [[pfSense Firewall Rules]]  [priority:: 1]#Firewall  2025-12-24
	- [x] Update VLAN 85 rules with provising script #pfSense #Firewall   2025-11-03
- [x] Remove OldNetwork Vlan and rules once migrated #pfSense #Cleanup  [priority:: 1]

### Switch Configuration

- [x] **Configure Network Switch:**Ensure switch supports VLANs and configure trunk ports #Network #Switch #Critical   2025-08-13  2025-08-16 
- [x] Check server ports are on correct VLAN information #Network #Verification 



- [x] Fix Dataview connection for sub tasks -  [priority:: 2]
```dataview 
TASK 
FROM "10 - Projects" 
WHERE !completed 
AND contains(phase, "Phase 3: Network Config")
AND contains(project_id, "Homelab-2025")
AND file.name != this.file.name 
SORT file.name ASC 
```

##  Spoke Notes & Documentation
* [[Pfsense Configuration Details]]
* [[pfSense Firewall Rules]]
* [[Proxmox Network Setup]]
* [[Extreme Switch Setup]]
* [[Extreme Switch Config Save and Rollback]]
* [[VLAN and Subnet Summary Sheet]]
* [[Networking Notes]]
* [[Updated Networking Approach]]
* [[Pi-Hole Setup]]
* [[Pi-Hole Installation Steps]]
* [[Router to Switch Layout.png]]
* [[Port Aliases - Services]]
* [[pfSense VPN Policy Routing & Kill Switch (NordVPN)]]
* [[Network Phased Configuration]]

---
##  Next Phase
* [[01 Homelab Rebuild - Phase 4 Proxmox Network & Storage Configuration Hub]]
