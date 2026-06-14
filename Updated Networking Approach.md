---
project_id: Homelab-2025
phase: 'Phase 3: Network Config'
tags:
  - Networking
status: Reference
---
# High-Level Network Plan
#HomeLabRebuild/VLAN #HomeLabRebuild/Network 

## 1. Layers Overview
Your network has three main layers:

1. **pfSense Router/Firewall**
   - Handles VLAN creation, inter-VLAN routing, firewall rules, and VPN termination.
2. **Extreme Switch**
   - Manages VLAN tagging/untagging, trunking, and link aggregation (LACP).
3. **Proxmox Host(s)**
   - Uses VLAN-aware network bridges for VMs and containers.

---

## 2. pfSense Setup

### Port Assignments
- **igb0 + igb2** → LAGG (LACP) trunk to Extreme switch (carries all VLANs).
- **igb1** → IoT VLAN 20 (direct SmartThings connection).
- **igb3** → Spare / out-of-band management.

### VLANs
- Create VLANs 10–90 on the LAGG interface.
- Assign each VLAN an interface and gateway IP (e.g., `10.0.X.1`).
- Apply firewall rules per VLAN.

### VPN
- VLAN 75 for VPN clients.
- Treat similar to Admin VLAN 70 but with firewall restrictions.

---

## 3. Extreme Switch

### Trunk Ports
- **Ports 1 + 3** → LACP trunk to pfSense LAGG.
  - Tagged VLANs: 10, 20, 30, 40, 50, 60, 70, 75, 80, 85, 90.

### Proxmox Ports
- **Ports 37, 39, 41, 43** → LACP trunk to Proxmox `bond0`.
  - Tagged VLANs: same set as pfSense trunk (or restricted if desired).

### IoT
- IoT-specific ports untagged VLAN 20, tagged nothing else.

---

## 4. Proxmox Host

### Bonding
- **bond0** with all 4 NICs in LACP (`802.3ad`).
- Recommended settings:
- bond-miimon 100  
- bond-mode 802.3ad  
- bond-lacp-rate fast  
- bond-xmit-hash-policy layer3+4

### Bridging
- Use a single VLAN-aware bridge (`vmbr0`).
- Example:
auto bond0
iface bond0 inet manual
    bond-slaves eno1 eno2 eno3 eno4
    bond-miimon 100
    bond-mode 802.3ad
    bond-lacp-rate fast
    bond-xmit-hash-policy layer3+4

auto vmbr0
iface vmbr0 inet manual
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

