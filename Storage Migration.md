---
project_id: Homelab-2025
phase: 'Phase 4: Storage Config'
tags:
  - Storage
status: Reference
---

https://claude.ai/chat/3183aef4-fa64-41ee-88a4-bbdacc0e4808

## **Proxmox Local Storage Strategy**

**Volume 1 (1TB SSD):** OS, docker swarm nodes and high data usage storage
- **Docker images/containers** - Local storage for better performance
- **Temporary/cache data** - Redis, temporary downloads
- **Database storage** - MySQL, InfluxDB (with NAS backup)

- [x] Install 2x SAS SSDs (960GB+) into 2I Box 1 free bays (e.g., Bay 5 + Bay 7). #todoist
- [x] Configure these in RAID‑1 on your P420i for redundancy. #todoist
- [x] Install Proxmox VE onto this RAID‑1. #todoist
- [x] ⏫  Create a VM/LXC storage pool on SSD for: #HomeLabRebuild/Storage 🆔 HomeLab 🛫 2025-08-04 ✅ 2025-10-01
	- Docker Swarm Manager nodes (VLAN 60).
	- Database and stateful services persistent volumes.


# Storage Setup

## Create main Docker Swarm dataset
zfs create rpool/docker-swarm

## Create sub-datasets for organization
zfs create rpool/docker-swarm/volumes      # Persistent volumes
zfs create rpool/docker-swarm/configs      # Stack configs and secrets  
zfs create rpool/docker-swarm/backups      # Local backup staging

### **Set ZFS Properties for Performance**

bash

```bash
# Optimize for Docker workloads
zfs set compression=lz4 rpool/docker-swarm
zfs set atime=off rpool/docker-swarm
zfs set recordsize=64K rpool/docker-swarm/volumes    # Good for databases
zfs set recordsize=1M rpool/docker-swarm/backups     # Good for backup files
```

### **Mount Points**

bash

```bash
zfs set mountpoint=/mnt/docker-swarm rpool/docker-swarm
zfs set mountpoint=/mnt/docker-swarm/volumes rpool/docker-swarm/volumes
zfs set mountpoint=/mnt/docker-swarm/configs rpool/docker-swarm/configs
zfs set mountpoint=/mnt/docker-swarm/backups rpool/docker-swarm/backups
```

## Additional ZFS Configuration

### **Set Quotas (Optional but Recommended)**

bash

```bash
# Prevent any single dataset from consuming all space
zfs set quota=200G rpool/docker-swarm/volumes
zfs set quota=50G rpool/docker-swarm/configs  
zfs set quota=100G rpool/docker-swarm/backups
```

# User Creation

- Created a user for docker swarm - docker - 997
- Password: 997
- Owns the docker-swarm folders
