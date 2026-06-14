---
project_id: Homelab-2025
phase: 'Phase 4: Proxmox Network & Storage Configuration Hub'
tags:
  - Virtio-FS
  - Storage
status: Reference
---

# Virtio-FS (ZFS Datasets from Proxmox Host)
docker-data    /mnt/docker-data    virtiofs    defaults    0    0
docker-db      /mnt/docker-db      virtiofs    defaults    0    0
docker-tsdb    /mnt/docker-tsdb    virtiofs    defaults    0    0
docker-swarm   /mnt/docker-swarm   virtiofs    defaults    0    0

# Synology NFS (Bulk Media - VLAN 100)
10.0.100.20:/volume1/tv         /mnt/synology/tv         nfs    defaults,noatime,nfsvers=4    0    0
10.0.100.20:/volume1/movies     /mnt/synology/movies     nfs    defaults,noatime,nfsvers=4    0    0
10.0.100.20:/volume1/animation  /mnt/synology/animation  nfs    defaults,noatime,nfsvers=4    0    0

## Swarm Storage

|**Resource Type**|**Host Path (Proxmox)**|**Virtio-FS Tag**|**Guest Path (Worker)**|**Purpose**|
|---|---|---|---|---|
|**App Configs**|`/mnt/docker-data`|`docker-data`|`/mnt/docker-data`|YAMLs, Python Scripts, SQLite|
|**Relational DB**|`/mnt/docker-db`|`docker-db`|`/mnt/docker-db`|NetBox (Postgres), MariaDB|
|**Time-Series**|`/mnt/docker-tsdb`|`docker-tsdb`|`/mnt/docker-tsdb`|Prometheus, InfluxDB|
|**Swarm State**|`/mnt/docker-swarm`|`docker-swarm`|`/mnt/docker-swarm`|Global Swarm configs/secrets|
|**Bulk Media**|`NAS:/volume1/...`|N/A (NFS)|`/mnt/synology/...`|TV, Movies, Animation|


| **Folder Path**             | **Purpose**                             | **Backup Priority** |
| --------------------------- | --------------------------------------- | ------------------- |
| `/mnt/docker-swarm/configs` | Global Compose files & Environment vars | **High**            |
| `/mnt/docker-swarm/secrets` | API keys, SSL certs (legacy)            | **Critical**        |
| `/mnt/docker-swarm/backups` | Automated DB exports (Postgres/Influx)  | **Medium**          |
| `/mnt/docker-swarm/volumes` | Shared persistent data (Non-DB)         | **High**            |
