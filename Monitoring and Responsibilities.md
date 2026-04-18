# Swarm Services + Monitoring Responsibilities

| Service / Component      | Node / VLAN                 | Monitored By                  | Metrics / Notes |
|--------------------------|-----------------------------|-------------------------------|----------------|
| Swarm Manager Nodes      | VLAN 60 (Infra)             | Node Exporter → Prometheus    | CPU, Memory, Disk, Network, Docker stats |
| Swarm Worker Nodes       | VLAN 85                     | Node Exporter → Prometheus    | CPU, Memory, Disk, Network, Docker stats |
| Portainer                | Manager Nodes (VLAN 60)     | cAdvisor / Prometheus         | Container health, Swarm overview |
| Traefik / Nginx          | Worker DMZ (VLAN 80)        | Traefik metrics endpoint → Prometheus | Requests/sec, Latency, Backend health |
| Plex                     | Worker Media (VLAN 50/85)   | Node Exporter + Plex Exporter | CPU, Memory, Disk, Transcode stats |
| Sonarr / Radarr          | Worker MediaMgmt (VLAN 85)  | Node Exporter + App Exporter  | CPU, Memory, Queue status |
| Transmission / Prowlarr  | Worker MediaMgmt (VLAN 85)  | Node Exporter + App Exporter  | Active downloads, disk usage |
| Home Assistant           | Controllers VLAN (VLAN 65)  | Node Exporter / HA Exporter   | CPU, Memory, Automation health |
| UniFi Controller         | Controllers VLAN (VLAN 65)  | Node Exporter / UniFi Exporter| Device health, client counts, Wi-Fi stats |
| Tautulli                 | Worker Media (VLAN 50/85)   | Node Exporter + App Exporter  | Plex activity metrics |
| Proxmox Hosts            | Infra VLAN (VLAN 60)        | Prometheus + Proxmox Exporter | VM stats, ZFS pool health, CPU, Memory |
| Synology NAS             | Infra VLAN (VLAN 60)        | Node Exporter / Synology Exporter | Disk usage, I/O, RAID health |
| Network Devices (Switches, Firewalls) | VLANs 10-90 | SNMP Exporters → Prometheus | Port stats, VLAN traffic, uptime |

---

## Notes / Best Practices
- All **metrics are scraped by Prometheus** on the dedicated Monitoring VM (VLAN 70).  
- **Node Exporters** run either on Swarm nodes or host OS, depending on workload.  
- Application-specific exporters (Plex, HA, UniFi, Traefik) provide **service-level insights**.  
- Alerts are handled via **Prometheus Alertmanager** and Grafana dashboards on the monitoring VM.  
- This setup ensures visibility **even if Swarm services fail**, since the monitoring VM is independent.