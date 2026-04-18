
---

# 🖥️ Homelab VM Architecture

This note outlines the VM setups for the homelab, separating **application databases** and **Swarm monitoring/observability services**, with persistence strategy and future scaling considerations.

---

## **1️⃣ App Database VM**

**Purpose:** Hosts all application databases and acts as the central DB server for app services.

|Feature|Recommendation|
|---|---|
|**VM Name**|`app-db`|
|**VLAN**|60 (Infrastructure)|
|**CPU**|4 vCPUs (1 socket × 4 cores)|
|**RAM**|8–12 GB|
|**OS**|Ubuntu Server LTS 24.04|
|**Network**|Static IP on VLAN 60; accessible by relevant app containers|
|**Volumes**||

- **Primary DB volume:** `/mnt/app-db` → Home Assistant, Sonarr, Radarr, Ombi, MySQL/Postgres, InfluxDB
    
- **Dedicated Plex volume:** `/mnt/plex-db` → Plex metadata DB |  
    | **Backup** | Nightly database dumps + VM snapshot; Plex DB can have separate backup schedule if needed |
    

**Notes:**

- Databases currently run directly on the VM, optionally in Docker for portability.
    
- Acts as the **central DB server** for app services.
    
- **Future:** Databases could be containerized and managed via Docker Swarm, either on this VM or across multiple nodes.
    

**Obsidian Links:**

- DB placement & checklist:  [[Database Zones]]
    
- Monitoring dashboards:  [[Monitoring Docker Swarm]]
    

---

## **2️⃣ Swarm Monitoring VM**

**Purpose:** Runs all monitoring and observability services for the homelab.

|Feature|Recommendation|
|---|---|
|**VM Name**|`swarm-monitoring`|
|**VLAN**|90 (Management)|
|**CPU**|4–8 vCPUs (1 socket × 4–8 cores)|
|**RAM**|8–16 GB|
|**OS**|Ubuntu Server LTS 24.04|
|**Network**|Static IP on VLAN 90; accessible from admin devices|
|**Volumes**|`/mnt/monitoring/{prometheus,grafana,loki,kuma,portainer}`|
|**Backup**|VM snapshot + volume backups for TSDB, logs, configs|

**Services:**

- Prometheus, Grafana, Loki, Uptime Kuma, Portainer
    
- Collects metrics from app databases and other services
    
- Application metrics are visualized here; databases remain on the App DB VM
    

**Notes:**

- Single-node setup uses host-mounted volumes; easy to backup and restore.
    
- **Future multi-node scaling:** volumes can move to NFS or another networked storage solution.
    

**Obsidian Links:**

- Swarm monitoring plan:  [[Monitoring Docker Swarm]]
    
- DB info for infra metrics: [[Database Zones]]
    

---

## **3️⃣ Key Points / Best Practices**

- **Separation of Concerns:**
    
    - Monitoring services live on a separate VM from actual application databases.
        
    - App DB VM hosts all databases, providing a single point for backups and high I/O workloads.
        
- **Volume Strategy:**
    
    - Start with **one primary volume for most DBs**, plus a **dedicated volume for Plex**.
        
    - Monitoring VM uses dedicated host-mounted volumes per service.
        
- **Future Scaling:**
    
    - Swarm services can later scale across multiple nodes.
        
    - Networked storage (NFS/Gluster/Ceph) will be required for DBs if they move into Swarm containers.
        
- **Backups:**
    
    - App DB VM: nightly DB dumps + VM snapshot.
        
    - Monitoring VM: snapshot `/mnt/monitoring`; optionally backup Grafana dashboards or Prometheus/Loki data.