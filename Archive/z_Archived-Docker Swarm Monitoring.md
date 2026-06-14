---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
# 🐳 Docker Swarm Control & Monitoring Plan

## 1. Control Plane

### Portainer
- **Deployment:** As a Swarm service, pinned to **manager nodes only**.
- **VLAN:** **90 (Management)**.
- **Role:** Single pane of glass for managing Swarm workloads, nodes, and stacks.
- **Access:**  
  - Internal: Management VLAN.  
  - External: Optional via VPN or Traefik.  
- **Best Practice:** Backup Portainer configs regularly (bind mount or NFS volume).

---

## 2. Monitoring Plane

### Dedicated VM
- **VM Name:** `swarm-monitoring`
- **VLAN:** **90 (Management)**
- **Role:** Runs monitoring/observability stack (separate from workloads & infrastructure data services).
- **Resources:** 4–8 vCPUs, 8–16GB RAM, ~100GB disk.

### Centralized (on `swarm-monitoring` VM, VLAN 90)
- **Prometheus** → scrapes infra exporters & stores metrics.
- **Grafana** → dashboards (cluster, containers, nodes).
- **Alertmanager** → handles Prometheus alerts (email, Discord, Slack).
- **Loki (optional)** → log aggregation.
- **Uptime Kuma** → service uptime checks.

### Distributed (as global Swarm services)
- **cAdvisor** → container-level metrics.
- **node-exporter** → host metrics (CPU, RAM, disk, network).
- **dockerd-exporter** → Docker engine stats.
- **Traefik exporter (optional)** → ingress/load balancer metrics.

---

## 3. Deployment Strategy
- **Exporters**
  - Run as `--mode global` Swarm services (1 replica per node).
  - Attached to all managers & workers.
  - Metrics exposed locally.

- **Central Stack**
  - Deploy with `docker stack deploy` on **monitoring VM** only.
  - Use constraints (`placement: [manager==true]`) so they don’t land on workers.
  - Runs Prometheus, Grafana, Alertmanager, Loki, Uptime Kuma.

- **Networking**
  - Prometheus scrapes exporters via Swarm overlay (or node IPs).
  - Grafana/Alertmanager/Kuma exposed via Traefik or VPN in VLAN 90.

---

## 4. Retention & Storage
- **Prometheus** → ~15–30 days local, remote storage for long-term.
- **Loki** → configurable retention (disk heavy).
- **Grafana** → backup dashboards/configs (Git or auto-export).
- **Portainer** → persistent configs via volume or NFS share.

---

## 5. Separation of Concerns

- **Control Plane (VLAN 90 — Management)**
  - Portainer (Swarm orchestration UI).

- **Infrastructure Monitoring (VLAN 90 — Management)**
  - Exporters + Prometheus/Grafana/Alertmanager/Uptime Kuma/Loki.

- **Application Telemetry (VLAN 60 — Infrastructure)**
  - Time-series data from apps (e.g., Home Assistant solar data, IoT sensors, Plex stats).
  - Stored in dedicated services (e.g., InfluxDB, MySQL).
  - Visualized in Grafana (which can connect to multiple datasources).

## 6. Database Zones

> **See [[Database Zones]] for full details on DB placement, types, and app-level checklist.**

- **Infrastructure / App-Level Databases:** see linked note.  
- **Control / Observability Databases:** see linked note.

---

## ✅ End Result
- **Portainer (VLAN 90):** Cluster control, manager-only.  
- **Monitoring VM (VLAN 90):** Infra monitoring + visualization.  
- **Infrastructure VLAN (60):** Application data services (InfluxDB, MySQL, etc.).  
- **Grafana (VLAN 90):** Unified dashboards — infra + application data.  
- Clean separation between **control plane**, **observability plane**, and **application data plane**.
