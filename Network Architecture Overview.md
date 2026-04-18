---
project_id: Homelab-2025
phase: "Phase 3: Network Config"
tags:
  - VLAN
  - Network
  - Hardware
---

# 🏠 Home Lab Overview

This document serves as the single source of truth for the Home Lab environment, detailing physical hardware, network segmentation, virtualization, and service orchestration (Docker Swarm).

## I. Physical Hardware Inventory

*This section documents physical assets. Fill in the specific details (Serial numbers, Power draw, Warranty status).*

| Component | Model/Spec | Role | Location | Management IP/Interface |
| :--- | :--- | :--- | :--- | :--- |
| **Firewall/Router** | PC with pfSense | Edge, Routing, DHCP | Rack Slot 1 | N/A (Internal LAN: [Enter IP]) |
| **Main Server** | HP ProLiant [Model] | Proxmox Host, VMs/Swarm | Rack Slot 2 | [Enter IP] (e.g., 10.0.10.5) |
| **Switch** | Extreme 48-port [Model] | L2/L3 Segmentation | Rack Slot 3 | [Enter IP] (e.g., 10.0.10.2) |
| **UPS** | [Model] | Power Backup | Floor | N/A |
| **Witness Node** | [Future Tiny/NUC PC] | Future Swarm Manager | Shelf | TBD |

## II. Network Architecture (IPAM)

*This section defines the logical network map (VLANs/Subnets).*

### A. VLAN Definition

[[VLAN and Subnet Summary Sheet]]

| VLAN ID | VLAN Name | Purpose | Subnet | Gateway (pfSense Interface) | Trunk Ports |
| :---: | :--- | :--- | :--- | :--- | :--- |
| 10 | MGMT | Management (pfSense, Proxmox, iLO) | `[Enter Subnet]` | [Enter IP] | [Switch Port #] (to Proxmox) |
| 20 | LAN | General User/Trusted Devices | `[Enter Subnet]` | [Enter IP] | [Switch Port #] |
| 30 | IoT | Untrusted IoT Devices | `[Enter Subnet]` | [Enter IP] | [Switch Port #] |
| 40 | DMZ | Public Facing Services (Future) | `[Enter Subnet]` | [Enter IP] | [Switch Port #] |
| 90 | DOCKER | Swarm Overlay Network | `[Enter Subnet]` | N/A (Internal) | N/A |

### B. Critical Static IP List

| Device/Service | IP Address | VLAN ID | MAC Address | Notes |
| :--- | :--- | :--- | :--- | :--- |
| pfSense MGMT Interface | [Enter IP] | 10 | | |
| HP ProLiant (Proxmox Host) | [Enter IP] | 10 | | |
| Extreme Switch Management | [Enter IP] | 10 | | |
| Docker Swarm Ingress VIP | [Enter IP] | 20 (or DMZ) | | Future HA Proxy IP |

## III. Virtualization & Orchestration

### A. Proxmox Configuration

| Setting | Value | Notes |
| :--- | :--- | :--- |
| Hostname | [Enter Hostname] | |
| Main Bridge (`vmbr0`) | [Enter Bridge Details] | Must be **VLAN Aware** |
| VM Storage | [Enter Storage Type] | ZFS/LVM/NFS |

### B. Docker Swarm Configuration

*The current strategy uses a Manager-Worker split for resilience and resource control.*

#### Swarm Nodes

| Node Name | Role(s) | CPU/RAM (Allocated) | Host VLAN ID | Node Labels |
| :--- | :--- | :--- | :--- | :--- |
| swarm-manager-1 | Manager | [Enter Spec] | 10 | `storage=ssd`, `ingress=true` |
| swarm-worker-1 | Worker | [Enter Spec] | 10 | `storage=hdd`, `media=true` |
| swarm-worker-2 | Worker | [Enter Spec] | 10 | `storage=ssd`, `db=true` |

#### Key Services (Example)

| Service Name | Description | Placement Constraint (Label) | Ingress/Port |
| :--- | :--- | :--- | :--- |
| `netbox` | IPAM/DCIM Documentation | `storage=ssd` | Port 8080 |
| `traefik` | Reverse Proxy/Load Balancer | `ingress=true` | Ports 80, 443 |
| `plex` | Media Server | `media=true` | Port 32400 |

## IV. High Availability (HA) & Future Roadmap

*This section details planned upgrades and architecture improvements to eliminate single points of failure (SPOFs).*

### A. High Availability Goals

1.  **Eliminate the single Proxmox Host SPOF:** Acquire a second physical node.
2.  **Achieve Swarm Quorum:** Deploy a 3-node Manager setup (currently 1/3).
3.  **Implement Shared Storage:** Move persistent application data to a centralized, fault-tolerant store.

### B. Planned Upgrades

| Upgrade | Priority | Estimated Cost | Notes |
| :--- | :--- | :--- | :--- |
| Dedicated NAS / Shared Storage | High | TBD | Required for persistent HA Swarm services. |
| 10GbE Network Card | Medium | TBD | Upgrade uplink from Proxmox to Switch to reduce bottleneck. |
| Separate Physical Swarm Manager | High | TBD | Use a low-power machine to host the 3rd Swarm Manager for true hardware HA. |
| SNMP Monitoring | Low | N/A | Configure pfSense/Switch/Proxmox for centralized monitoring. |

---

