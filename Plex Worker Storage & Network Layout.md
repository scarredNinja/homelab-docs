---
project_id: Homelab-2025
status: Reference
phase: 'Phase 5: Docker Swarm'
tags:
  - reference
  - architecture
---
```text
            Plex Worker VM (VMID: 201)
            ─────────────────────────
                       │
           ┌───────────┴───────────┐
           │                       │
       [ZFS Disk]               [NFS Shares]
   plex-config(20G)        media, extras, trailers, backups
   /mnt/volumes/plex-config /mnt/media
           │                       │
           │                       │
   ┌----------------┐     ┌---------------------------┐
   │ Docker Volumes │     │ Docker Volumes (NFS)      │
   │ plex-config    │     │ media → /mnt/media/media  │
   │ tautulli-config│     │ extras → /mnt/media/extras│
   │                │     │ trailers → /mnt/media/trailers│
   │                │     │ backups → /mnt/media/backups │
   └----------------┘     └---------------------------┘

Network Connections:
-------------------
VM → NFS Server (10.0.60.80) → Reads/Writes directly
VM → Proxmox ZFS pool → Local disk for configs
VM → Docker containers → See volumes via symlinks
