---
project_id: Homelab-2025
phase: 'Phase 4: Storage Config'
tags:
  - ZFS
  - Storage
status: Reference
---

# ZFS Configuration and Setup

## ZFS Layout

- `rpool/docker-data` (General Files - `recordsize=128k`) - Mapped
    
    - `portainer/`
        
    - `traefik/`
        
    - `netbox/`
        
- `rpool/docker-tsdb` (Database Files - **`recordsize=16k`**) - Mapped
    
    - `netbox-postgres/`
    - 
- `rpool/media` - Mapped
	- `recordsize=1M`



### 1. The Metrics Tier (Prometheus / InfluxDB)

- **Dataset:** `rpool/docker-tsdb`
    
- **Optimization:** `recordsize=16k` or `32k` (check your specific Influx version docs).
    
- **Why:** TSDBs write in small chunks. A smaller recordsize prevents ZFS from rewriting 128k of data for every 4k metric update.
    

### 2. The Media Tier (Plex)

- **Dataset:** `rpool/media`
    
- **Optimization:** `recordsize=1M`
    
- **Why:** Movies are huge. A 1MB recordsize is much more efficient for streaming and significantly reduces the metadata overhead on your ZFS pool.
    
- **Note:** Turn **Atime=off** here so Plex doesn't write to the disk every time it "reads" a file to check for metadata.


### 3. The App Data Tier (Configs / Traefik)

- **Dataset:** `rpool/docker-data`
    
- **Optimization:** Default (128k).
    
- **Why:** Great for small text files, YAMLs, and Portainer’s database.
