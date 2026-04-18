# Swarm Storage & I/O Considerations

## Current LXC Setup
- Each app service (Plex, Home Assistant, Radarr, Sonarr, Grafana, etc.) runs in its **own LXC container**.
- Configs and persistent data are stored **locally on Proxmox node disks** (ZFS pool).
- **Pros**
  - Fast local access.
  - No network dependency.
- **Cons**
  - Config/data tied to node/container.
  - Harder to migrate or reschedule containers.
  - Backup requires per-node snapshots.

## Proposed Swarm Setup
- Docker Swarm with **NFS-backed volumes** on Synology (`10.0.60.80`).
- **Configs & persistent data** live on NFS shares.
- Volumes mounted into containers as:
  - `/mnt/config/<service>` → config & DBs
  - `/mnt/media/<share>` → media, downloads, or shared datasets
- **Pros**
  - Containers can move between nodes without losing config/data.
  - Centralized backup strategy.
  - Easier scaling of Swarm workers.
- **Cons**
  - Slight latency increase vs local disk.
  - If NFS server goes down, dependent containers lose access.

## I/O Considerations
- **Configs**: small (~MB–GB), occasional reads/writes — negligible impact on Synology.  
- **Databases**: small persistent DBs can live on NFS; for high-write DBs (InfluxDB, MySQL), local node storage is better.  
- **Media & downloads**: continue storing large files on NFS (already high I/O workload).  
- **Scratch space** (temp files, transcodes, incomplete downloads): store locally on node ZFS for high IOPS.

## Recommended Storage Strategy
| Service               | Config / DB | Temp / Cache | Notes |
|-----------------------|------------|--------------|-------|
| Plex                  | NFS        | Local        | Config on NFS, transcodes/cache local |
| Sonarr / Radarr       | NFS        | Local        | Config on NFS, temp downloads on local disk if needed |
| Transmission / Prowlarr | NFS      | Local        | Downloads can go to NFS media shares, incomplete temp on local disk |
| Home Assistant        | NFS        | Local        | Config centralized for HA automation & state |
| UniFi Controller      | NFS        | Local        | Config centralized, keeps controller portable |
| Grafana               | NFS        | Local        | Dashboards & DBs on NFS; temp cache local |
| Prometheus / InfluxDB | Local      | Local        | High-write DBs kept on local node for performance |
| Traefik / Nginx       | NFS        | Local        | Config centralized; logs/cache local |

## Notes
- Using NFS for configs makes containers portable across nodes.
- Heavy I/O workloads (media, DB writes, transcodes) should remain local to maintain performance.
- This hybrid approach balances portability, redundancy, and performance for a home lab setup.