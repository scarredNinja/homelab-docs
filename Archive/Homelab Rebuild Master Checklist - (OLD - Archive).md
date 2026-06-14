---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
# Homelab Rebuild - Master Checklist

> **Project Status:** Preparation & Planning 
> **Last Updated:** 2025-10-10

---

# General Tasks

- [x] 📅 2025-08-12 update current in progress phase and check for similar tasks #HomeLabRebuild/Critical %%[todoist_id:: 9452796008]%% ✅ 2025-08-22
- [x] ⏫ update dashboard as some groups are either messy or not could be separated further #HomeLabRebuild/Update ✅ 2025-12-13
- [x] ⏫ Split out tasks into sub folders if they become too big and link back #HomeLabRebuild/Update ✅ 2025-12-13
- [x] ⏫ Link recently added documentation #HomeLabRebuild/Update ✅ 2025-12-13
- [x] ⏫ Create in progress/deployed items on canvas #HomeLabRebuild/Update ✅ 2025-12-13

## Phase 1 - Preparation & Planning 🔄 IN PROGRESS

### Backups & Documentation

- [x]  **Backup Everything:** Perform full backups of any existing data and configurations. Verify the integrity of these backups. Copy critical data from your Synology NAS. ✅ 2025-08-04 #HomeLabRebuild/Backup #HomeLabRebuild/Critical #HomeLabRebuild/Phase1  [link](https://app.todoist.com/app/task/9452796066) #todoist %%[todoist_id:: 9452796066]%%
- [x]  Create scripts to back up LXC container 104 data to NAS ✅ 2025-08-04 #HomeLabRebuild/Backup #HomeLabRebuild/Scripts #HomeLabRebuild/Phase1 #todoist
- [x]  Create scripts to back up all LXC container data to NAS ✅ 2025-08-04 #HomeLabRebuild/Backup #HomeLabRebuild/Scripts #HomeLabRebuild/Phase1 #todoist
- [x]  Test backup scripts with one container ✅ 2025-08-04 #HomeLabRebuild/Backup #HomeLabRebuild/Testing #HomeLabRebuild/Phase1  #todoist
- [x]  Set daily backup schedule (replace PBS backups) ✅ 2025-08-04 #HomeLabRebuild/Backup #HomeLabRebuild/Automation #HomeLabRebuild/Phase1 #todoist
- [x]  Confirm backups are running ✅ 2025-08-04 #HomeLabRebuild/Backup #HomeLabRebuild/Verification #HomeLabRebuild/Phase1 #todoist
- [x]  Export all Portainer stacks ✅ 2025-08-04 #HomeLabRebuild/Backup #HomeLabRebuild/Portainer #HomeLabRebuild/Phase1 #todoist
- [x]  **Document Current Configuration:** Note down existing IP addresses, network settings, and software configurations. ✅ 2025-08-04 #HomeLabRebuild/Documentation #HomeLabRebuild/Phase1 #todoist

### Planning & Design

- [x] **Network Diagram:** Draw detailed network diagram with all VLANs (10,20,30,40,50,60,70,75,80,90) and IP subnets #HomeLabRebuild/Network #HomeLabRebuild/Documentation #HomeLabRebuild/Phase1 🔼 ✅ 2025-09-25
- [x] **IP Addressing Scheme:** Define static IPs for infrastructure and DHCP ranges #HomeLabRebuild/Network #HomeLabRebuild/Documentation #HomeLabRebuild/Phase1 #todoist ⏫ ✅ 2025-08-22
- [x]  All pfSense VLAN Rules outlined and documented ✅ 2025-08-04 #HomeLabRebuild/Network #HomeLabRebuild/pfSense #HomeLabRebuild/Documentation #todoist
- [x] Update current networking config for pfSense and switch #HomeLabRebuild/Network #HomeLabRebuild/Documentation #HomeLabRebuild/Phase1 #todoist ✅ 2026-06-02
- [x]  Update IP address scheme as configured ✅ 2025-08-04 #HomeLabRebuild/Network #HomeLabRebuild/Documentation #HomeLabRebuild/Phase1 #todoist
- [x]  Document pfSense rules ✅ 2025-08-04 #HomeLabRebuild/pfSense #HomeLabRebuild/Documentation #HomeLabRebuild/Phase1 #todoist
- [/] Document pfSense rules #HomeLabRebuild/pfSense #HomeLabRebuild/Documentation #HomeLabRebuild/Phase1 ⏫ 🛫 2025-07-01 ⏳ 2025-08-15 📅 2026-06-19

### Infrastructure Preparation

- [x]  **Download ISOs:** Get latest Proxmox VE ISO ✅ 2025-08-04 #HomeLabRebuild/Proxmox #HomeLabRebuild/Phase1 #todoist
- [x]  Setup Bootable Proxmox USB ✅ 2025-08-04 #HomeLabRebuild/Proxmox #HomeLabRebuild/Phase1 #todoist
- [x]  Look at current storage setup and layout new server storage ✅ 2025-08-04 #HomeLabRebuild/Storage #HomeLabRebuild/Planning #HomeLabRebuild/Phase1 #todoist
- [x]  Get current RAID setup on rack server and expansion slots ✅ 2025-08-04 #HomeLabRebuild/Storage #HomeLabRebuild/Hardware #HomeLabRebuild/Phase1 #todoist
- [x]  Update notes with new hard drives and clean up ✅ 2025-08-04 #HomeLabRebuild/Storage #HomeLabRebuild/Documentation #HomeLabRebuild/Phase1 #todoist
- [x]  Steps to setup RAID on server refresh ✅ 2025-08-04 #HomeLabRebuild/Storage #HomeLabRebuild/Documentation #HomeLabRebuild/Phase1 #todoist
- [ ]  Get size of external hard drives  🔽 #HomeLabRebuild/Storage #HomeLabRebuild/Backup #HomeLabRebuild/Phase1

### Docker & Application Prep

- [x]  **Docker Installation Script/Notes:** Prepare commands for Docker Engine installation and Docker Swarm initialization ⏫ ✅ 2025-08-04 #HomeLabRebuild/Docker #HomeLabRebuild/Scripts #HomeLabRebuild/Phase1 #todoist
- [x]  Installation Steps for Docker Swarm (VMs vs LXCs decision) ✅ 2025-08-04 #HomeLabRebuild/Docker #HomeLabRebuild/Documentation #HomeLabRebuild/Phase1 #todoist
- [x]  Information on storage and persistence with Docker Swarm setup ✅ 2025-08-04 #HomeLabRebuild/Docker #HomeLabRebuild/Storage #HomeLabRebuild/Documentation #HomeLabRebuild/Phase1 #todoist

---



---

## Phase 3 - Network Configuration (pfSense) 🔄 IN PROGRESS

### Basic pfSense Setup

- [x]  **Assign Interfaces:** Assign WAN and LAN interfaces in pfSense ✅ 2025-08-04 #HomeLabRebuild/pfSense #HomeLabRebuild/Network #HomeLabRebuild/Phase3 
- [x]  **Create VLAN Interfaces:** Create VLAN interfaces for all VLANs (10,20,30,40,50,55,60,65,70,75,80,90) ✅ 2025-08-04 #HomeLabRebuild/pfSense #HomeLabRebuild/VLAN #HomeLabRebuild/Phase3 
- [x]  **Configure DHCP Servers:** Configure DHCP for each VLAN with specified IP ranges ✅ 2025-08-04 #HomeLabRebuild/pfSense #HomeLabRebuild/DHCP #HomeLabRebuild/Phase3 
- [x]  Apply networking rules ✅ 2025-08-04 #HomeLabRebuild/pfSense  #HomeLabRebuild/Firewall #HomeLabRebuild/Phase3 
- [x]  Re-add WAN configuration ✅ 2025-08-04 #HomeLabRebuild/pfSense #HomeLabRebuild/WAN #HomeLabRebuild/Phase3 
- [x]  Recheck all rules that have been added are correct ✅ 2025-08-04 #HomeLabRebuild/pfSense #HomeLabRebuild/Verification #HomeLabRebuild/Phase3 #todoist

### DNS & Core Services

- [x] 🔼 **Set up DNS:** Configure DNS resolver/forwarder in pfSense (pihole) #HomeLabRebuild/pfSense #HomeLabRebuild/DNS #HomeLabRebuild/Phase3 ✅ 2025-11-03
- [ ] 🔼 Add in the Traeflik work - internal/external #HomeLabRebuild/pfSense #HomeLabRebuild/DNS #HomeLabRebuild/Phase3  #archive 



---


---

## Phase 5 - Docker Swarm and Infrastructure Setup 🔄 IN PROGRESS

### Manager Provisioning Template

[[Manager Provisioning Tasks]]


[[Docker Swarm Tasks]]

### Monitoring Vms Setup

[[Swarm Monitoring VM Tasks]]

### app-db Server Tasks -TBC

[[app-db Server VM Tasks]]
### Testing & Validation

- [x] Test create Portainer deployment on initial swarm manager #HomeLabRebuild/Docker #HomeLabRebuild/Portainer #HomeLabRebuild/Testing #HomeLabRebuild/Phase5 #HomeLabRebuild/SwarmManager ⛔ yhvkcv ✅ 2025-10-10
	- Will live on vlan 90 (set static ip)
    - [x] Take docker compose and try it out #HomeLabRebuild/Docker #HomeLabRebuild/Testing #HomeLabRebuild/Phase5 ✅ 2025-08-27

---

## Phase 6 - Core Service & Workload Deployment 📅 UPCOMING

### Infrastructure Services


- [ ] **Deploy Proxmox Backup Server (PBS):** Create on VLAN 60, install and configure PBS #HomeLabRebuild/Backup #HomeLabRebuild/Infrastructure #HomeLabRebuild/Update #archive 

### Application Services

- [ ] Update list of work services and arwas/VLANS #HomeLabRebuild/Update ⏫ #archive

- [ ]  **Deploy Media Services (VLAN 50):** Plex, qBittorrent, Tdarr with NFS mounts #HomeLabRebuild/Docker #HomeLabRebuild/Media #archive 
- [ ]  **Deploy Media Management (VLAN 50):** Sonarr, Radarr, Prowlarr with NFS persistence #HomeLabRebuild/Docker #HomeLabRebuild/MediaMgmt #archive 
- [ ]  **Deploy Homelab Services (VLAN 40):** Home Assistant, Grafana with NFS persistence #HomeLabRebuild/Docker #HomeLabRebuild/Homelab #todoist #archive 
- [ ]  **Deploy Database Services (VLAN 65):** PostgreSQL, MariaDB, Redis with NFS persistence #HomeLabRebuild/Docker #HomeLabRebuild/Database #todoist #archive 
- [ ]  **Deploy DMZ Services (VLAN 80):** External Access Dashboard (Dashy) with NFS persistence #HomeLabRebuild/Docker #HomeLabRebuild/DMZ #todoist #archive 

---


---


---
