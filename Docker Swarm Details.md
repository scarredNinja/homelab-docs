---
phase: 'Phase 5: Docker Swarm'
project_id: Homelab-2025
tags:
  - docker-swarm
  - reference
status: Reference
---
# Swarm Services Placement & Storage Matrix

## Service to Worker Mapping

Reference: [[VLAN and Subnet Summary Sheet]]

- [x] Remove two manager vms and stick to the one [priority:: 1]  2026-01-28

[[Mount Point Map#Docker Services]]

| Service                 | Node Type    | VLAN(s) | Config / DB Storage | Temp / Cache Storage | Notes                                                      | Node Labels                | Dataset - [[Proxmox storage setup Steps - post migration# Prepare the Target ZFS Structure]] | Status  |
| ----------------------- | ------------ | ------- | ------------------- | -------------------- | ---------------------------------------------------------- | -------------------------- | ---------------------------------------------------------------------------------------------- | ------- |
| Portainer               | Manager      | 60      | NFS                 | Local                | Control plane, HA mode                                     | node.role == manager       | rpool/docker-data/portainer                                                                    | Running |
| Traefik                 | Manager      | 60      | NFS                 | Local                | Reverse proxy for external access                          | node.labels.zone == public | rpool/docker-data/traefik                                                                      | Running |
| Plex                    | Worker-Media | 50      | Local               | Local                | Config on local; transcodes/cache local                    | node.labels.type == media  | rpool/docker-data/plex                                                                         | Running |
| Sonarr / Radarr         | Worker-MediaMgmt | 50   | Local               | Local                | SQLite DBs must stay on local storage (virtiofs)           | node.labels.zone == mediamanagement | rpool/docker-data/sonarr<br>rpool/docker-data/radarr                                     | Running |
| Transmission / Prowlarr | Worker-MediaMgmt | 50   | Local               | Local                | Transmission + Gluetun run via Compose, Prowlarr in Swarm | node.labels.zone == mediamanagement | rpool/docker-data/transmission<br>rpool/docker-data/prowlarr                                   | Running |
| Seerr                   | Worker-MediaMgmt | 50   | Local               | Local                | Request management UI, rebranded successor of Overseerr   | node.labels.zone == mediamanagement | rpool/docker-data/seerr                                                                       | Running |
| Homepage                | Worker-Monitoring | 60      | Local               | Local                | Self-hosted dashboard; https://dashboard.home.purvishome.com | node.labels.zone == monitoring | rpool/docker-data/homepage                                                                    | Running |
| Home Assistant          | Controller   | 60      | Local               | Local                | Config on local storage (virtiofs) for SQLite integrity    | node.labels.zone == controller | rpool/docker-data/homeassistant                                                                | Running |
| UniFi Controller        | Controller   | 60      | Local               | Local                | Network controller, MongoDB local storage                 | node.labels.zone == controller | rpool/docker-data/uniFi                                                                        | Running |
| Grafana                 | Manager      | 60      | NFS                 | Local                | Dashboards, metrics                                        | node.role == manager       | rpool/docker-data/grafana                                                                      | Running |
| Prometheus              | Manager      | 60      | Local               | Local                | High-write metrics DB, local for performance               | node.labels.storage == ssd | rpool/docker-tsdb/prometheus                                                                   | Running |
| Alertmanager            | Worker-Monitoring | 60 | Local          | Local                | Alert routing + Discord notifications; data dir owned by nobody (uid 65534) | node.labels.zone == monitoring | rpool/docker-tsdb/alertmanager                                                             | Running |
| InfluxDB                | Manager      | 60      | Local               | Local                | Time-series DB, high-write, local node storage             | node.labels.storage == ssd | rpool/docker-tsdb/influxDB                                                                     | Running |
| Uptime Kuma             | Worker-Monitoring | 60 | Local          | Local                | Service uptime monitoring; https://uptime-kuma.home.purvishome.com | node.labels.zone == monitoring | rpool/docker-data/uptime-kuma                                                          | Running |
| NewtonFit               | Worker-Monitoring | 60 | Local          | Local                | Personal fitness dashboard; https://newtonfit.home.purvishome.com | node.labels.zone == monitoring | rpool/docker-data/newtonfit                                                            | Running |
| Tautulli                | Worker-Media | 50      | NFS                 | Local                | Optional monitoring for Plex                               | node.labels.type == media  | rpool/docker-data/tautulli                                                                     | Running |
| Misc Dev/Test Apps      | HomeLab      | 40      | Local / NFS         | Local                | Depends on workload; temporary apps can use local          | node.labels.nas == nfs     | tbc                                                                                            | NA      |

---
## Storage

| Service             | Stated Persistence         | Refined Best Practice       | Critical Data Location | Bulk Data Location            |
|---------------------|----------------------------|-----------------------------|------------------------|-------------------------------|
| Plex                | Config on NFS              | Database/Metadata: **LOCAL**| Local SSD/NVMe         | NFS (Media Files)             |
| Sonarr / Radarr     | Config/DB on NFS (Implied) | Database/Config: **LOCAL**  | Local SSD/NVMe         | NFS (Downloads, Media)        |
| Home Assistant      | Config/DB on NFS (Implied) | Database/Config: **LOCAL**  | Local SSD/NVMe         | NFS (Backups  Optional)      |
| UniFi Controller    | Config/DB on NFS (Implied) | Database/Config: **LOCAL**  | Local SSD/NVMe         | NFS (Backups  Optional)      |
| Tautulli            | Optional Monitoring        | Database/Config: **LOCAL**  | Local SSD/NVMe         | NFS (Logs/Backups  Optional) |
| Traefik / Nginx     | Reverse Proxy              | Certificates/Config: **LOCAL** | Local SSD/NVMe      | N/A                           |
| Transmission/Prowlarr| Downloads on NFS           | Config/Metadata: **LOCAL**  | Local SSD/NVMe         | NFS (Downloads)               |
| Seerr               | Config on NFS (Implied)    | Database/Config: **LOCAL**  | Local SSD/NVMe         | N/A                           |
| Homepage            | Local                      | Database/Config: **LOCAL**  | Local SSD/NVMe         | N/A                           |
| Prometheus          | Local                      | Database: **LOCAL** (Confirmed) | Local SSD/NVMe     | N/A                           |
| InfluxDB            | Local                      | Database: **LOCAL** (Confirmed) | Local SSD/NVMe     | N/A                           |
| NewtonFit           | Local                      | Database: **LOCAL** (Confirmed) | Local SSD/NVMe     | N/A                           |

---

### Swarm Details:

docker swarm join --token SWMTKN-1-0vau8ooxvkknug97zfd1h1ayuah7e1um73mo86rvx6igf6750l-b2zkeftuz53588rjwruipb0d4 10.0.60.30:2377

## Action Plan Notes

- **Map Local Volumes**  
  Ensure the Docker volume mount for the main configuration path (e.g., `/config` or `/var/lib/plexmedia`) of all the **LOCAL** services above is pointed to a persistent volume on the worker's local storage.  

- **NFS for Bulk**  
  Ensure the Docker volume mounts for large content (e.g., `/data`, `/movies`, `/tv`) for Plex and the download clients are correctly mapped to the NFS share.  

- **Database/NFS Warning **  
  Never store high-IO databases like **SQLite (Plex, Sonarr, Radarr)**, **MongoDB (UniFi)**, or the **HA database** on an NFS share due to the risk of corruption from poor file locking and high latency.


## Notes / Best Practices
- **Managers** (VLAN 60) should run critical orchestration & DB services. Avoid heavy workloads.  
- **Workers** (VLAN 85) handle scaled services and high-I/O apps.  
- **Controllers VLAN 65** isolates HA and UniFi from IoT VLAN 20 for security, but allows controlled access.  
- **Local storage for temp/scratch workloads** preserves performance for heavy I/O (transcoding, high-write DBs).  Also config
- **Overlay networks** should be used for cross-VLAN communication within Swarm.


**Notes**

Managers (VLAN 60)  Portainer, Grafana, Prometheus, INFLUXDB. Traefik

Workers (VLAN 85)  Scalable app workloads: Plex, Radarr/Sonarr, Transmission.

Controllers VLAN (65)  Home Assistant & UniFi Controller, connected to IoT VLAN (20) for device access.

DMZ VLAN (80)  Traefik exposes services externally; only allowed paths to internal VLANs.

Admin VLAN (70)  Dashboards (Grafana, Portainer).

Overlay networks handle cross-node container communication.


---

## Links

- [[Old - Docker Swarm Worker Template Setup Guide]]
- [[Docker Swarm Provisioning Template]]
- [[Docker Swarm Infrastructure Runbook]]
