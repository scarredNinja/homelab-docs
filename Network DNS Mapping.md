---
project_id: Homelab-2025
phase: "Phase 3: Network Config"
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
| Plex                    | Worker-Internal | 50      | Docker   | TBC                            | TBC           | NA      |
| Sonarr / Radarr         | Worker-Internal | 50      | Docker   | TBC                            | NA            | NA      |
| Transmission / Prowlarr | Worker-Internal | 50      | Docker   | TBC                            | NA            | NA      |
| Home Assistant          | Worker-Internal | 60      | Docker   | TBC                            | TBC           | NA      |
| UniFi Controller        | Worker-Internal | 60      | Docker   | TBC                            | NA            | NA      |
| Grafana                 | Manager         | 60      | Docker   | grafana.home.purvishome.com    | NA            | Running |
| Prometheus              | Manager         | 60      | Docker   | prometheus.home.purvishome.com | NA            | Running |
| InfluxDB                | Manager         | 60      | Docker   | influxdb.home.purvishome.com   | NA            | Running |
| Tautulli                | Worker-Internal | 50      | Docker   | TBC                            | NA            | NA      |
| Misc Dev/Test Apps      | Worker-Internal | 40      | Docker   | TBC                            | NA            | NA      |
| Netbox                  | Manager         | 60      | Docker   | netbox.home.purvishome.com     | NA            | Running |
| pfsense                 | NA              | 90      | External | pfsense.home.purvishome.com    | NA            | Running |
| pihole                  |                 |         |          |                                |               |         |
| proxmox                 |                 |         |          |                                |               |         |

