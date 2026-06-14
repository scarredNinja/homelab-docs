---
project_id: Homelab-2025
status: Reference
phase: 'Phase 5: Docker Swarm'
tags:
  - reference
  - architecture
---
## Load Balancing Considerations

### Trunk Port 1 (Primary)

- VLAN 60 (Infrastructure) - Heavy NFS/database traffic
- VLAN 85 (Docker Swarm) - Container orchestration
- VLAN 90 (Management) - Lower traffic

### Trunk Port 2 (Secondary)

- VLAN 50 (Media) - Plex streaming (highest bandwidth)
- VLAN 70 (Admin) - Dashboard access
- VLAN 10, 20, 30 (Client VLANs) - User traffic

## Key Docker Swarm Ports

- **2376**: Docker daemon API (TLS)
- **2377**: Swarm cluster management
- **7946**: Container network discovery (TCP & UDP)
- **4789**: Overlay network traffic (UDP)

**08/08/2025**: I did create two bonded pairs on proxmox and extreme and it ran into an issue when trying to connect proxmox and pfsense

**Action:** Go Over network architecture and research on how best to achieve:
	- Good level of bandwidth to the server
	- Docker swarm and implementation
	- VPN and external access
- Considerations found:
	- I remove the bonding on Port 37 in Port at 39 and we're looking to go for dedicated VMS on each network interface card
	- I need to confirm if this is the right or a good option and then test. I cannot ping the 60 or 90 gateways from the switch and need to see why.
