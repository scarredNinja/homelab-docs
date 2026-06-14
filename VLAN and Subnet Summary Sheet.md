---
project_id: Homelab-2025
phase: 'Phase 3: Network Config'
tags:
  - VLAN
  - Subnet
status: Reference
---

Network setup found: [[(Archive) - Network Correction]]
## VLAN Setup

| VLAN Tag | VLAN Name      | Subnet        | Purpose / Description                                                                                                                                                                                                     | Link                                                       | Services                                                                                       |
| -------- | -------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| 10       | Home           | 10.0.10.0/25  | Vlan for all Home devices                                                                                                                                                                                                 | [[pfSense Firewall Rules#VLAN 10 - Home Network]]          | N/A                                                                                            |
| 20       | IoT            | 10.0.20.0/25  | VLAN for all IoT devices (excluding HomeAssisstant)                                                                                                                                                                       | [[pfSense Firewall Rules#VLAN 20 - IoT]]                   | Hue, SmartThings, ChromeCast                                                                   |
| 30       | Guest          | 10.0.30.0/25  | VLAN for Guest Only Access                                                                                                                                                                                                | [[pfSense Firewall Rules#VLAN 30 - Guest]]                 | NAS                                                                                            |
| 40       | Homelab        | 10.0.40.0/25  | Development and testing environment; hosts Proxmox VMs/containers for lab work and experimentation.                                                                                                                       | [[pfSense Firewall Rules#VLAN 40 - Homelab]]               | TBC                                                                                            |
| 50       | Media          | 10.0.50.0/25  | Media servers like Plex and Tautulli serving content to clients.                                                                                                                                                          | [[pfSense Firewall Rules#VLAN 50 - Media]]                 | Plex, Sonarr, Radarr, Transmission, Prowlarr                                                   |
| 60       | Infrastructure | 10.0.60.0/25  | Core network services (DNS, NTP, backups) and **Docker Swarm Manager nodes**; trusted, stable VLAN. **Unifi**. **HomeAssisstant**                                                                                         | [[pfSense Firewall Rules#VLAN 60 - Infrastructure]]        | InfluxDB. Promethus NAS, Docker Swarm Managers, Portainer, Swarm Workers, Grafana, pihole, dns |
| 70       | Admin          | 10.0.70.0/25  | Workstations, jump boxes, and admin access clients with broad permissions across the network. **Granfana**, **Uptime Kuma**, **Dashy** and VPN clients for remote access, generally granted similar access as Admin VLAN. | [[pfSense Firewall Rules#VLAN 70 - Admin - Is it needed?]] | VPN                                                                                            |
| 80       | DMZ            | 10.0.80.0/25  | Public-facing services isolated from internal VLANs; e.g., reverse proxies or externally accessible hosts.                                                                                                                | [[pfSense Firewall Rules#VLAN 80 - DMZ]]                   | Reverse Proxy                                                                                  |
| 90       | Management     | 10.0.90.0/25  | Management interfaces of network devices (iLO, iDRAC, switches) accessible only by Admin/VPN VLANs.                                                                                                                       | [[pfSense Firewall Rules#VLAN 90 - Management - ✅]]        |                                                                                                |
| 91       | Cluster        | 10.0.91.0/25  | Corosync / Heartbeat - To be created later                                                                                                                                                                                |                                                            | Corosync / Heartbeat                                                                           |
| 100      | Storage        | 10.0.100.0/25 | Handling storage across the network                                                                                                                                                                                       | [[pfSense Firewall Rules#VLAN 100 - Storage]]              | NFS / iSCSI / SMB, Synology                                                                    |

---

## Design Notes

- **IoT VLAN (20)** → only devices; **no controllers** for security.  
- **Infrastructure VLAN (60)** → trusted core for databases, monitoring, storage, and Swarm Managers.  
- **Admin VLAN (70)** → central control plane for dashboards and monitoring tools.  
- **DMZ VLAN (80)** → strictly reverse proxy / public-facing apps. Only controlled ingress to internal VLANs.  
- **Management VLAN (90)** → out-of-band device management only (no internet access).
 
## Switch Configuration
- EXOS Commands
- Trunk & Access Port Setup

## IP Table - Update static when moving over

- [x] ⏫ Combine static ip tables and add check column #HomeLabRebuild/Network ✅ 2025-08-11

| Service Name     | VLAN Tag | VLAN Name      | Static IP Address | Subnet Mask     | Gateway   | Notes                        | Internal Address                              | SSL |
| ---------------- | -------- | -------------- | ----------------- | --------------- | --------- | ---------------------------- | --------------------------------------------- | --- |
| pfSense LAN      | N/A      | LAN            | 10.0.1.1          | 255.255.255.0   | N/A       | pfSense default LAN          |                                               |     |
| Proxmox Host     | 60       | Homelab        | 10.0.60.50        | 255.255.255.128 | 10.0.60.1 | Main hypervisor              | 10.0.90.50:8006                               | NA  |
| Plex             | 50       | Media          | 10.0.50.20        | 255.255.255.128 | 10.0.50.1 | Plex Media Server            | TBC                                           | TBC |
| Tautulli         | 50       | Media          | 10.0.50.10        | 255.255.255.128 | 10.0.50.1 | Plex monitoring              |                                               |     |
| Home Assistant   | 65       | Infrastructure | 10.0.60.11        | 255.255.255.128 | 10.0.60.1 | Smart home automation        |                                               |     |
| UniFi Controller | 65       | Infrastructure | 10.0.60.10        | 255.255.255.128 | 10.0.60.1 | Wi-Fi/AP management          |                                               |     |
| Grafana          | 70       | Admin          | 10.0.70.10        | 255.255.255.128 | 10.0.70.1 | Metrics dashboard            |                                               |     |
| Prometheus       | 60       | Databases      | 10.0.60.11        | 255.255.255.128 | 10.0.60.1 | Data collection              |                                               |     |
| Radarr           | 50       | Media          | 10.0.50.10        | 255.255.255.128 | 10.0.50.1 | Download manager             |                                               |     |
| Sonarr           | 50       | Media          | 10.0.50.11        | 255.255.255.128 | 10.0.50.1 | TV series automation         |                                               |     |
| Transmission     | 50       | Media          | 10.0.50.12        | 255.255.255.128 | 10.0.50.1 | Torrent client               |                                               |     |
| InfluxDB         | 60       | Databases      | 10.0.60.20        | 255.255.255.128 | 10.0.60.1 | Time-series DB               |                                               |     |
| MySQL            | 60       | Databases      | 10.0.60.21        | 255.255.255.128 | 10.0.60.1 | Relational DB                |                                               |     |
| VPN Gateway      | 70       | VPN            | 10.0.75.1         | 255.255.255.128 | 10.0.75.1 | WireGuard/OpenVPN on pfSense |                                               |     |
| Management VLAN  | 90       | Management     | 10.0.90.2         | 255.255.255.128 | 10.0.90.1 | iLO/iDRAC, switches, etc.    |                                               |     |
| Synology         | 60       | Storage        | 10.0.60.80        | 255.255.255.128 | 10.0.60.1 | Synology Nas                 |                                               |     |
| Synology         | 60       | Storage        | 10.0.60.45        | 255.255.255.128 | 10.0.60.1 | Synology Nas Port 2          |                                               |     |
| UpTime Kuma      | 60       | Admin          | 10.0.60.11        | 255.255.255.0   | 10.0.60.1 | Uptime Kuma                  |                                               |     |
| Dashy            | 60       | Admin          | 10.0.60.30        | 255.255.255.0   | 10.0.60.1 | Main Dashboard               |                                               |     |
| Traefik          | 60       | Infrastructure | 10.0.60.30        | 255.255.255.0   | 10.0.60.1 | Reverse Proxy                | https://traefik.home.purvishome.com/          | Y   |
| Portainer        | 60       | Infrastructure | 10.0.60.30        | 255.255.255.0   | 10.0.60.1 | Container Management         | https://portainer.home.purvishome.com/#!/auth | Y   |
| Netbox           | 60       | Infrastructure | 10.0.60.30        | 255.255.255.0   | 10.0.60.1 | Infrastructure               | https://netbox.home.purvishome.com/           | Y   |
