---
project_id: Homelab-2025
phase: "Phase 4: Storage Config"
tags:
  - Storage
  - Network
  - Docker
---
[[01 Homelab Rebuild - Phase 4 Proxmox Network & Storage Configuration Hub]]
## 🏗️ Homelab Storage & Performance Architecture

### 1. Storage Tiering Strategy

To maximize stability, we are separating the "Fast Path" (operating systems and databases) from the "Bulk Path" (media and backups).

|**Tier**|**Hardware**|**Capacity**|**Role**|**Performance Notes**|
|---|---|---|---|---|
|**Tier 0: Flash**|2x 1TB SSDs (RAID 1)|~1TB|Proxmox OS, Swarm VM Boot Disks, Databases|**Primary Speed Layer.** 500MB/s+ R/W. Essential for `etcd` and DB latency.|
|**Tier 1: SAS**|4x 600GB (RAID 5)|~1.8TB|Torrent Incompletes, ISOs, VM Snapshots|**Scratch Layer.** Fast sequential speeds, higher latency than SSD. Saves SSD wear.|
|**Tier 2: Cold**|1x 600GB SAS|600GB|Local Backup / Emergency Boot|Single point of failure. Use for non-critical logs or local templates.|
|**Tier 3: Bulk**|Synology (Hybrid RAID)|20TB|Media (Plex), Backups, Docker Configs|**Mass Storage.** Limited by 1Gbps network speed (~115MB/s max).|

---

### 2. Network & Connectivity Constraints

Your 1Gbps Extreme Switch is the main bottleneck. We must manage "east-west" traffic (traffic between the server and the NAS) carefully.

- **Speed Limit:** 1Gbps = **125MB/s theoretical max**. Real-world throughput is usually **~110MB/s**.
    
- **VLAN Isolation:** * **VLAN 60 (Infrastructure/Storage):** High-priority. Ensure the Synology Port 2 and the Proxmox Storage Bridge are on this VLAN.
    
    - **VLAN 50 (Media):** High-bandwidth. Traffic from Synology to Plex will live here.
        
- **Optimization:** Use **LACP (Bonding)** on the Synology ports if your Extreme Switch supports it. This won't make a single file transfer faster than 1Gbps, but it allows _two_ 1Gbps streams to happen simultaneously to different clients.
    

---

### 3. Docker Swarm Data Placement

In a Swarm, where a container "lives" determines how it should access data.

- **Stateful Services (Databases):** * _Examples:_ InfluxDB, MySQL, Prometheus.
    
    - _Storage:_ **Local Bind Mounts** on the SSD Tier.
        
    - _Constraint:_ Use **Placement Constraints** in your Docker Compose to lock these containers to specific Swarm nodes that have the SSD backing.
        
- **Stateless Services (App Logic):** * _Examples:_ Dashy, Uptime Kuma, Nginx.
    
    - _Storage:_ Can live anywhere; configs should be mirrored to Synology via NFS for easy recovery (backups) and local config with SSD Bind mounts

|**Service Category**|**Examples**|**Placement / Storage**|**VLAN**|
|---|---|---|---|
|**Media Managers**|Sonarr, Radarr, Prowlarr|**Compute:** SSD VMs. **Config:** Synology NFS.|50 (Media)|
|**Dashboards**|Dashy, HomeAssistant|**Compute:** SSD VMs. **Config:** Local SSD Bind Mount.|70 (Admin) / 60 (Infra)|
|**The "Big" App**|Plex|**Compute:** SSD VMs. **Config/Metadata:** SSD Bind Mount.|50 (Media)|
|**Network Tools**|Uptime Kuma, UniFi|**Compute:** SSD VMs. **Config:** Local SSD Bind Mount.|60 (Infra)|
        
- **Media Services (IO Intensive):**
    
    - _Examples:_ Plex, Radarr.
        
    - _Storage:_ **NFS Mounts** to Synology.
        
    - _Performance Tip:_ Map the "Transcode" directory for Plex to `/dev/shm` (RAM) or the SSDs to prevent 1Gbps network lag from buffering your video.
        

---

### 4. Critical Stability Checklists

- **The "Gen8" Fan Issue:** Since you are using non-HPE SSDs, monitor your iLO. If the fans stay at 70%+, you may need to use a modified iLO firmware or specific fan-control scripts to keep the noise down.
    
- **Write Cache:** Ensure the P420i controller has a working battery (FBWC). **Without it, RAID 5 write speeds will drop to ~20MB/s**, which will bottleneck your downloads.
    
- **IP Conflicts:** As noted previously, ensure `10.0.60.11` is unique to one device/service to prevent the Swarm Ingress mesh from collapsing.
    

---

### 5. Growth & Size Limitations

- **Expandability:** The DL360p Gen8 is SFF (2.5"). 1TB is currently your SSD ceiling unless you buy expensive high-capacity enterprise drives.
    
- **Swarm Scaling:** With one Proxmox node, you are "Scaling for Management," not "Scaling for Redundancy." If the Proxmox host fails, all Swarm nodes fail.


----

### 📂 Proposed Storage Configuration Map

#### **Tier 1: High-Speed Local (2x 1TB SSD RAID 1)**

- **Proxmox Host:** OS, ISO Templates, and Container Images.
    
- **Swarm VMs:** Virtual disks ($vDisks$) for the Swarm nodes.
    
- **App Data (The "Snappy" Stuff):** * **Plex Metadata:** Millions of tiny files (posters/database) that need low latency.
    
    - **Databases:** InfluxDB, MySQL, and Prometheus WAL (Write-Ahead Logs).
        
    - **Dashboards:** Configs for Dashy and Home Assistant.
        

#### **Tier 2: Intermediate Local (4x 600GB SAS RAID 5)**

- **The "Worker" Space:** Dedicated virtual disks mounted to the Swarm VMs at `/mnt/scratch`.
    
- **Heavy IO:** Transmission/qBittorrent "Incomplete" folders. This prevents "torrent thrashing" from wearing out your SSDs.
    
- **Backups:** Local Proxmox snapshots of your Swarm VMs for quick rollbacks.
    

#### **Tier 3: Bulk Network Storage (Synology 20TB)**

- **The "Vault":** Mounted via **NFS** to the Swarm Workers over **VLAN 60/50**.
    
- **Media Library:** Final "Movies" and "TV" folders for Plex.
    
- **Secondary Backups:** The final destination for all critical app configs (automated nightly rsync).
    

---

### 🚦 Performance & Size Review

|**Metric**|**SSD Tier (Local)**|**SAS Tier (Local)**|**Synology Tier (NFS)**|
|---|---|---|---|
|**Max Throughput**|~550 MB/s (SATA3)|~400 MB/s (RAID 5)|**~110 MB/s (1GbE Limit)**|
|**Latency**|< 1ms|~5–10ms|~15–30ms+|
|**Size Limit**|1TB (Fixed)|1.8TB (Fixed)|**20TB (Expandable)**|
|**Best For**|Booting, DBs, UI|Downloads, Logs|Media, Cold Backups|

### ⚠️ Critical Configuration Rule

**Avoid "Double Networking":** Do not run a Swarm Manager's internal state on an NFS share. If your Extreme Switch reboots or a cable nudges loose, the Swarm will lose quorum and collapse. Keeping the Swarm state on the **Tier 1 SSDs** ensures that even if the NAS goes offline, your management plane and internal dashboards remain accessible.

![[Pasted image 20260116170820.png]]