#HomeLabRebuild/Storage 

## General Notes

- All local VM disks live on your Proxmox storage pool (`rspool` or similar).
    
- NFS shares from `10.0.60.80` can be mounted under `/mnt/media/<share>` for media or shared volumes.
    
- VM disks are labeled as `vm-<ID>-disk-<n>` in Proxmox.
    
- Docker volumes map directly to mount points inside the VM.
    

---

## VM: `plex-worker` (Media Consumption)

**Proxmox Disks:**

- `vm-105-disk-1` → 100GB → Configs
    
- `vm-105-media` → 2TB → Media (or NFS mount `/mnt/media/plex`)
    

**Mount Points:**

- `/mnt/configs` → Plex, Tautulli
    
- `/mnt/media` → Media library
    

**Docker Volumes:**

- `plex-config` → `/mnt/configs/plex`
    
- `tautulli-config` → `/mnt/configs/tautulli`
    
- `media` → `/mnt/media`
    

---

## VM: `media-mgmt-worker` (Radarr, Sonarr, Transmission)

**Proxmox Disks:**

- `vm-110-disk-1` → 50GB → Configs
    
- `vm-110-downloads` → 500GB → Download staging (local or NFS)
    

**Mount Points:**

- `/mnt/configs` → Radarr, Sonarr, Transmission configs
    
- `/mnt/downloads` → Temporary download storage
    

**Docker Volumes:**

- `radarr-config` → `/mnt/configs/radarr`
    
- `sonarr-config` → `/mnt/configs/sonarr`
    
- `transmission-config` → `/mnt/configs/transmission`
    
- `downloads` → `/mnt/downloads`
    

---

## VM: `home-assistant-worker`

**Proxmox Disks:**

- `vm-102-disk-1` → 50GB → Configs
    

**Mount Points:**

- `/mnt/configs` → HA & UniFi configs
    

**Docker Volumes:**

- `ha-config` → `/mnt/configs/home-assistant`
    
- `unifi-config` → `/mnt/configs/unifi`
    

---

## VM: `metrics-worker` (Grafana, Prometheus, Graylog)

**Proxmox Disks:**

- `vm-105-disk-1` → 50GB → Configs
    
- `vm-105-data` → 500GB → Databases/metrics
    

**Mount Points:**

- `/mnt/configs` → Service configs
    
- `/mnt/data` → DB & metrics storage
    

**Docker Volumes:**

- `grafana-config` → `/mnt/configs/grafana`
    
- `prometheus-data` → `/mnt/data/prometheus`
    
- `graylog-data` → `/mnt/data/graylog`
    

---

## VM: `db-worker` (InfluxDB & MySQL)

**Proxmox Disks:**

- `vm-109-disk-1` → 50GB → Configs
    
- `vm-109-data` → 1TB → DB storage
    

**Mount Points:**

- `/mnt/configs` → DB configs
    
- `/mnt/data` → DB data
    

**Docker Volumes:**

- `influxdb-data` → `/mnt/data/influxdb`
    
- `mysql-data` → `/mnt/data/mysql`