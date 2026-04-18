---
project_id: Homelab-2025
phase: "Phase 4: Storage Config"
tags:
  - Storage
---
# Current Storage Setup - Information Needed

## Purpose
Gather current storage configuration to:
- Tailor storage advice to existing hardware
- Ensure NFS VM strategy compatibility
- Plan for single-server persistence
- Prepare for future HA expansion

---

## Required Information

[[Server Hardware]]

### 1. Storage Hardware

### Proxmox:

1x Nodes

Drives:
 - 2x 1TB SSDs (RAID 1) - will be used for new server OS
 - 600 GB SAS HDD - Current server OS hard drive
 - 1.2TB SAS HDD (I believe in RAID 5 making it 4 600GB drives) - Used as additional sotrage

The Server: hpe proliant dl360p gen8 e5-2630
### Synology:

1x NAS

Drives:
- 2x 10TB HDDs
- 2x 6TB HDDs

All on Synology RAID and total useable space is 20TB


---
---

### 3. Docker Swarm VM Distribution

**Current VMs**:
- [x] Number of Manager VMs
- [x] Number of Worker VMs
- [x] Total VM count

**VM Specifications**:
| VM Name | Role | vCPU | RAM | Disk Size | Notes |
|---------|------|------|-----|-----------|-------|
| | Manager/Worker | | | | |
| | Manager/Worker | | | | |
| | Manager/Worker | | | | |

---

## Next Steps After Information Gathering

### Immediate Planning
1. Map NFS VM implementation on existing hardware
2. Determine optimal storage allocation
3. Plan VM disk provisioning strategy
4. Identify bottlenecks or limitations

### Future HA Transition
1. Design migration path to multi-server setup
2. Plan for Ceph/GlusterFS integration
3. Document expansion requirements
4. Budget for additional hardware

---

## Template for Response

```
### Storage Hardware
- Drives: [e.g., 2x 500GB NVMe, 6x 2TB HDD]
- Configuration: [e.g., NVMe in ZFS mirror, HDDs in RAIDZ2]

### Proxmox Storage
- Pool 1: local (NVMe) - 400GB free
- Pool 2: vm-storage (HDD) - 8TB free
- Total available for Swarm: XTB

### Current VMs
- 3 Manager VMs: 2vCPU, 4GB RAM each
- 5 Worker VMs: 4vCPU, 8GB RAM each
```

---

## Notes
- Detailed information enables precise recommendations
- Current setup influences NFS VM sizing and placement
- Understanding hardware helps avoid performance bottlenecks
- Planning now simplifies future HA expansion
```

- [ ] Summarize current setup/intended setup and review #HomeLabRebuild/Review ⏫ 

---

```markdown
# Docker Swarm Storage & HA Architecture Plan

## Overview
**Goal**: Achieve HA in Docker Swarm where if a worker VM fails, services can instantly restart on a new worker VM with access to the exact same data.

**Current Setup**: Single HP ProLiant running Proxmox with Docker Swarm VMs

---

## Storage Options Analysis

### Option 1: NFS/SMB Share (Simplest)

**How it works**: Dedicate a separate VM (or physical machine/NAS) to serve NFS/SMB shares. All Docker Swarm VMs mount this share.

**Pros**:
- Easiest to set up
- Minimal overhead on Swarm nodes
- Works well with bind mounts and simple file storage

**Cons**:
- NFS server becomes a Single Point of Failure (SPOF)
- Performance issues for databases due to file locking
- Network latency impacts

**Best For**: Simple file storage (Plex metadata, large media archives)

---

### Option 2: Proxmox Native ZFS Replication

**How it works**: Set up ZFS locally on HP ProLiant. Use Proxmox's ZFS tools to replicate entire VM disks to a standby physical server.

**Pros**:
- Excellent data integrity and performance
- Simple to manage within Proxmox GUI
- Fast local storage

**Cons**:
- **Not True HA for Docker Swarm**
- Minutes of downtime during failover
- Worker VMs cannot simultaneously see same data
- Risk of data corruption for stateful services

**Best For**: Backup and Disaster Recovery for entire VMs, not application-level HA

---

### Option 3: Distributed Storage (Ceph/Longhorn)

**How it works**: Dedicate multiple VMs (minimum 3 recommended) to run distributed storage. Pool local disks to create shared storage for all Swarm VMs.

**Pros**:
- **True HA** - data replicated across nodes
- Instant failover - data accessible from remaining nodes
- Services restart immediately on healthy workers

**Cons**:
- Complex setup and management
- Significant resource requirements (CPU/RAM/Network)
- Ideally needs 10GbE and dedicated network infrastructure
- Better suited for multiple physical servers

**Best For**: Highly critical, stateful services (databases, volume-dependent applications)

---

## Recommended Strategy

### Service-Based Storage Approach

| Service Type | Storage Method | Rationale |
|--------------|----------------|-----------|
| **Stateless Services** (Web frontends, Reverse Proxies) | Swarm Overlay Network | No persistence needs; naturally HA via Swarm replication |
| **Simple Persistent Files** (Media, Logs, Configs) | NFS/SMB Bind Mounts | Fast and simple; SPOF acceptable for non-transactional data |
| **Stateful Databases** (PostgreSQL, MariaDB) | Application-Level Replication | Native database replication across Workers; database handles persistence and failover |
| **Core Storage** (Future) | Proxmox ZFS Replication | Disaster Recovery: replicate entire Docker Swarm VM environment to backup server |

---

## Immediate Next Steps

### Priority: Database Replication Setup

For stateful applications requiring HA:

1. **Deploy 3 replicas** of the database container across Swarm VMs
2. **Configure native replication**:
   - PostgreSQL: streaming replication
   - MongoDB: Replica Sets
   - MariaDB: Galera Cluster
3. **Let the application handle consistency**: When a Worker VM fails, Swarm restarts the service and ensures 3 healthy replicas exist

### Benefits:
- Data volumes can remain local to each worker
- Database software handles data synchronization
- Automatic failover at the application layer
- No complex distributed storage needed initially

---

## Decision Points

**Questions to address**:
1. Focus on HA database setup (PostgreSQL stack)?
2. Split out services architecture first?
3. Budget for future hardware expansion?
4. Timeline for implementation?

---

## Notes

- Single physical server limits true HA options
- Application-level replication is most practical for current setup
- Future expansion to multiple physical servers enables Ceph/Longhorn
- Hybrid approach recommended: combine methods based on service requirements
```