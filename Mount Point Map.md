---
project_id: Homelab-2025
phase: "Phase 4: Storage Config"
tags:
  - Storage
  - Docker Swarm
---

# Mount Points and Connections

## Docker Services

| Service                 | Config / DB Storage | Temp / Cache Storage | Dataset - [[Proxmox storage setup Steps - post migration#🟦 Prepare the Target ZFS Structure]] | Container Mapping                    |
| ----------------------- | ------------------- | -------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------ |
| Portainer               | NFS                 | Local                | rpool/docker-data/portainer                                                              | /mnt/docker-data/portainer/portainer |
| Traefik / Nginx         | NFS                 | Local                | rpool/docker-data/traefik                                                                | /mnt/docker-data/traefik/traefik     |
| Plex                    | Local               | Local                | rpool/docker-data/plex                                                                   |                                      |
| Sonarr / Radarr         | NFS                 | Local                | rpool/docker-data/sonarr<br>rpool/docker-data/radarr                                     |                                      |
| Transmission / Prowlarr | NFS                 | Local                | rpool/docker-data/transmission<br>rpool/docker-data/prowlarr                             |                                      |
| Home Assistant          | NFS                 | Local                | rpool/docker-data/homeassistant                                                          |                                      |
| UniFi Controller        | NFS                 | Local                | rpool/docker-data/uniFi                                                                  |                                      |
| Grafana                 | NFS                 | Local                | rpool/docker-data/grafana                                                                | /mnt/docker-data/grafana             |
| Prometheus              | Local               | Local                | rpool/docker-tsdb/prometheus                                                             | /mnt/docker-tsdb/promethus           |
| InfluxDB                | Local               | Local                | rpool/docker-tsdb/influxDB                                                               | /mnt/docker-tsdb/influxdb            |
| Tautulli                | NFS                 | Local                | rpool/docker-data/tautulli                                                               |                                      |
| Misc Dev/Test Apps      | Local / NFS         | Local                | tbc                                                                                      |                                      |