---
phase: 'Phase 3: Network Config'
project_id: Homelab-2025
status: Reference
tags:
  - VLAN
  - Network
  - DNS
---


# Network DNS Mapping



| Service                 | Node Type       | VLAN(s) | Platform | Internal FQDN                  | External FQDN | Status  |
| ----------------------- | --------------- | ------- | -------- | ------------------------------ | ------------- | ------- |
| Portainer               | Manager         | 60      | Docker   | portainer.home.purvishome.com  | NA            | Running |
| Traefik                 | Worker-Public   | 80      | Docker   | traefik.home.purvishome.com    | NA            | Running |
| Plex                    | Worker-Media    | 50      | Docker   | plex.home.purvishome.com       | plex.purvishome.com | Running |
| Sonarr / Radarr         | Worker-MediaMgmt| 50      | Docker   | sonarr.home.purvishome.com<br>radarr.home.purvishome.com | NA            | Running |
| Transmission / Prowlarr | Worker-MediaMgmt| 50      | Docker   | transmission.home.purvishome.com<br>prowlarr.home.purvishome.com | NA            | Running |
| Seerr                   | Worker-MediaMgmt| 50      | Docker   | seerr.home.purvishome.com      | seerr.purvishome.com | Running |
| Home Assistant          | Worker-Internal | 60      | Docker   | homeassistant.home.purvishome.com | NA            | Running |
| UniFi Controller        | Worker-Internal | 60      | Docker   | unifi.home.purvishome.com      | NA            | Running |
| Grafana                 | Manager         | 60      | Docker   | grafana.home.purvishome.com    | NA            | Running |
| Prometheus              | Manager         | 60      | Docker   | prometheus.home.purvishome.com | NA            | Running |
| InfluxDB                | Manager         | 60      | Docker   | influxdb.home.purvishome.com   | NA            | Running |
| Tautulli                | Worker-Internal | 50      | Docker   | tautulli.home.purvishome.com   | NA            | Running |
| Misc Dev/Test Apps      | Worker-Internal | 40      | Docker   | TBC                            | NA            | NA      |
| Netbox                  | Manager         | 60      | Docker   | netbox.home.purvishome.com     | NA            | Running |
| pfsense                 | NA              | 90      | External | pfsense.home.purvishome.com    | NA            | Running |
| pihole                  | NA              | 60      | External | pihole.home.purvishome.com     | NA            | Running |
| pihole-2                | NA              | 60      | External | pihole2.home.purvishome.com    | NA            | Running |
| proxmox                 | NA              | 90      | External | pve.home.purvishome.com        | NA            | Running |
| synology                | NA              | 100     | External | synology.home.purvishome.com   | NA            | Running |
