---

## status: In Progress 
priority: Medium 
due_date: 2026-05-30 
project_id: Homelab-2025 
phase: "Phase 8: Basic Monitoring"

---
# 📊 Phase 8: Basic Monitoring

> [!abstract] Phase dependency Core monitoring infrastructure (Prometheus, Grafana, Loki, node-exporter, cAdvisor) is deployed as part of the Docker Swarm runbook — [[Docker Swarm Infrastructure Runbook#Phase 1.5 — Monitoring VM Provisioning]]. Complete Phase 1.5 first. Phase 8 covers the extended target build-out on top of that foundation.

## Infrastructure

|Component|VM|VLAN|Status|
|---|---|---|---|
|`worker-monitoring-01`|Dedicated monitoring VM|60 (Infrastructure)|Pending provisioning|
|Prometheus|Swarm service on `worker-monitoring-01`|60|Pending|
|Grafana|Swarm service on `worker-monitoring-01`|60|Pending|
|Loki|Swarm service on `worker-monitoring-01`|60|Pending|
|node-exporter|Global — all Swarm nodes|All|Pending|
|cAdvisor|Global — all Swarm nodes|All|Pending|
|Promtail|Global — all Swarm nodes|All|Pending|

**Stack file:** `proxmox-swarm/stacks/monitoring/stack-monitoring.yml` **Config path:** `/mnt/docker-data/prometheus/prometheus.yml` **TSDB path:** `/mnt/docker-tsdb/prometheus`


## Prerequisites (from Phase 1.5)

- [ ] `worker-monitoring-01` provisioned and joined Swarm with `zone=monitoring` label
- [ ] Core monitoring stack deployed and stable
- [ ] Grafana accessible at `https://grafana.home.purvishome.com`
- [ ] Prometheus accessible at `https://prometheus.home.purvishome.com`
- [ ] All current Swarm nodes appearing in Prometheus targets as UP

---

## Phase 8 Extended Targets

Complete these after Phase 2 (all worker VMs provisioned) so all scrape targets exist.

### Proxmox Host Metrics

Add scrape job to `prometheus.yml`:

```yaml
  - job_name: proxmox
    metrics_path: /pve
    params:
      module: [default]
    static_configs:
      - targets: ['10.0.60.50']
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: pve-exporter:9221
```

- [ ] Proxmox API user `prometheus@pve` created with read-only permissions
- [ ] `pve-exporter` added to monitoring stack
- [ ] Proxmox metrics appearing in Prometheus
- [ ] Grafana dashboard imported: ID `10347`

---

### pfSense Metrics

- [ ] pfSense node_exporter package installed (pfSense UI → System → Package Manager)
- [ ] Scrape job added to `prometheus.yml` targeting pfSense IP
- [ ] pfSense metrics appearing in Prometheus

---

### Alertmanager

- [ ] Alertmanager added to monitoring stack
- [ ] `alerting` block added to `prometheus.yml`
- [ ] Alert rules file created at `/mnt/docker-data/prometheus/alerts/`
- [ ] Discord webhook configured
- [ ] Basic alerts: node down, disk > 80%, service restart loop
- [ ] Test alert fired and received

---

### App-Specific Exporters (after Phase 3 service migration)

|Service|Exporter|Status|
|---|---|---|
|Sonarr|`exportarr`|After Phase 3|
|Radarr|`exportarr`|After Phase 3|
|Home Assistant|Built-in Prometheus integration|After Phase 3|
|UniFi|`unifi-poller`|After Phase 3|
|Traefik|Built-in metrics (already scraping)|Done in Phase 1.5|

- [ ] `exportarr` deployed for Sonarr and Radarr
- [ ] Home Assistant Prometheus integration enabled
- [ ] `unifi-poller` deployed

---

### Uptime Kuma

- [ ] Uptime Kuma added to monitoring stack, accessible at `https://uptime.home.purvishome.com`
- [ ] Monitors configured: Traefik, Portainer, Grafana, Plex, Home Assistant, pfSense

---

### Dashboard Build-out

|Dashboard|Grafana ID|When|
|---|---|---|
|Node Exporter Full|1860|Phase 1.5|
|Docker cAdvisor|193|Phase 1.5|
|Docker Swarm|13639|Phase 1.5|
|Traefik v2/v3|15489|Phase 1.5|
|Proxmox|10347|After pve-exporter|
|Loki + Promtail logs|15141|After Loki stable|
|Custom homelab overview|—|After all sources connected|

---

## Stale References — Do Not Follow

- `Docker_Swarm_Monitoring.md` — references VLAN 85/65/90 and a standalone `swarm-monitoring` VM; superseded
- `Monitoring_and_Responsibilities.md` — stale VLAN references; use service list only
- `stack.yaml`, `promethus.yml`, `prometheus + grafana + loki.yml` — old draft stacks; superseded by runbook stack file
- IP table: Grafana `10.0.70.10`, Prometheus `10.0.60.11` — stale; monitoring is a Swarm service, not static IPs

---

## 📝 Links

- [[Docker Swarm Infrastructure Runbook]] → Phase 1.5 for core stack deployment
- [[Monitoring and Responsibilities]] — service checklist (ignore VLAN refs)
- [[z_Archived-Docker Swarm Monitoring]] — archived design doc

---

## ➡️ Next Phase

- [[01 Homelab Rebuild - Phase 9 External Access Hub]]