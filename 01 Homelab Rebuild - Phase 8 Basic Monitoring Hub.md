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

| Component              | VM                                      | VLAN                | Status                    |
| ---------------------- | --------------------------------------- | ------------------- | ------------------------- |
| `worker-monitoring-01` | Dedicated monitoring VM                 | 60 (Infrastructure) | ✅ Deployed 2026-04-30    |
| Prometheus             | Swarm service on `worker-monitoring-01` | 60                  | ✅ Deployed 2026-04-30    |
| Grafana                | Swarm service on `worker-monitoring-01` | 60                  | ✅ Deployed 2026-04-30    |
| Loki                   | Swarm service on `worker-monitoring-01` | 60                  | ✅ Deployed 2026-04-30    |
| node-exporter          | Global — all Swarm nodes                | All                 | ✅ Deployed 2026-04-30    |
| cAdvisor               | Global — all Swarm nodes                | All                 | ✅ Deployed 2026-04-30    |
| Promtail               | Global — all Swarm nodes                | All                 | ✅ Deployed 2026-04-30    |

**Stack file:** `proxmox-swarm/stacks/monitoring/stack-monitoring.yml` **Config path:** `/mnt/docker-data/prometheus/prometheus.yml` **TSDB path:** `/mnt/docker-tsdb/prometheus`


## Prerequisites (from Phase 1.5)

- [x] `worker-monitoring-01` provisioned and joined Swarm with `zone=monitoring` label ✅ 2026-04-30
- [x] Core monitoring stack deployed and stable ✅ 2026-04-30
- [x] Grafana accessible at `https://grafana.home.purvishome.com` ✅ 2026-04-30
- [x] Prometheus accessible at `https://prometheus.home.purvishome.com` ✅ 2026-04-30
- [x] All current Swarm nodes appearing in Prometheus targets as UP ✅ 2026-04-30

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

- [x] Proxmox API user `prometheus@pve` created with read-only permissions ✅ 2026-04-30
- [x] `pve-exporter` added to monitoring stack ✅ 2026-04-30
- [x] Proxmox metrics appearing in Prometheus ✅ 2026-04-30
- [x] Grafana dashboard imported: ID `10347` ✅ 2026-04-30

---

### pfSense Metrics

- [x] pfSense node_exporter package installed (pfSense UI → System → Package Manager) [priority:: 3] ✅ 2026-05-20
- [x] Scrape job added to `prometheus.yml` targeting pfSense IP [priority:: 3] ✅ 2026-05-20
- [x] pfSense metrics appearing in Prometheus [priority:: 3] ✅ 2026-05-20

---

### Alertmanager

- [x] Alertmanager added to monitoring stack ✅ 2026-05-20 — PR #48; prom/alertmanager:v0.27.0, pinned to zone=monitoring, UI at `alertmanager.home.purvishome.com`
- [x] `alerting` block added to `prometheus.yml` ✅ 2026-05-20 — pointing to `tasks.monitoring_alertmanager:9093`
- [x] Alert rules file created ✅ 2026-05-20 — `config/monitoring/alert-rules.yml`; 7 rules deployed
- [x] Discord webhook configured in Alertmanager ✅ 2026-05-20 — URL stored at `/etc/alertmanager/discord-webhook` on Proxmox host; injected by copy-swarm-config.sh
- [x] Test alert fired and received in Discord ✅ 2026-05-20

#### Hardware Alert Rules (target thresholds)

| Alert | Metric | Threshold | Severity |
|---|---|---|---|
| Node down | `up == 0` | any node offline > 1m | critical |
| CPU high | `node_cpu_seconds_total` | > 85% sustained 5m | warning |
| Memory high | `node_memory_MemAvailable_bytes` | < 10% free sustained 5m | warning |
| Disk high | `node_filesystem_avail_bytes` | < 15% free on any mount | warning |
| Disk critical | `node_filesystem_avail_bytes` | < 5% free on any mount | critical |
| ZFS pool degraded | `node_zfs_pool_state != 0` | any state other than ONLINE | critical |
| Service restart loop | `rate(container_last_seen[5m])` | container restarting repeatedly | warning |
| High load average | `node_load15` | > (num_cores × 0.8) for 15m | warning |
| Swap in use | `node_memory_SwapFree_bytes` | < 50% swap free | warning |
| Proxmox host down | `up{job="proxmox-node"}` | host unreachable > 1m | critical |

- [x] Add hardware alert rules file `config/monitoring/alerts/hardware.yml` [priority:: 2] ✅ 2026-05-22 — Deployed modular alert files on `feature/modular-alerting-rules`
- [x] Add ZFS pool health alert rules `config/monitoring/alerts/zfs.yml` [priority:: 2] ✅ 2026-05-22 — Deployed modular alert files on `feature/modular-alerting-rules`
- [x] Add Swarm service alert rules (node down, service replica count) `config/monitoring/alerts/swarm.yml` [priority:: 2] ✅ 2026-05-22 — Deployed modular alert files on `feature/modular-alerting-rules`

---

### Next Session Tasks

#### PR #48 Cleanup
- [x] Merge PR #48 (feat(monitoring): Alertmanager + Discord webhook alerting) to main ✅ 2026-05-20
- [x] Add DNS A record: `alertmanager.home.purvishome.com` → Traefik/DMZ IP (same as `prometheus.home.purvishome.com`) [priority:: 2] ✅ 2026-05-20

#### Phase 2 — Inter-VLAN SSH + Automated Script Deploy
- [x] Add pfSense firewall rule: VLAN60 (10.0.60.0/25) → VLAN50 (10.0.50.0/24) TCP/22 ✅ 2026-05-20
- [x] Test SSH from manager-01: `ssh docker@worker-media-01` [priority:: 1] ✅ 2026-05-22 — verified from both manager-01 and pve directly
- [x] Test remote deploy: `bash /mnt/docker-swarm/scripts/deploy-worker-scripts.sh --remote worker-media-01 --no-run` [priority:: 1] ✅ 2026-05-22 — working from pve; dual IdentityFile fallback added; both workers deployed successfully

#### Homepage Dashboard
- [ ] Homepage dashboard — verify Traefik routing to dashboard.home.purvishome.com; healthcheck IPv6 issue fixed 2026-05-25 (commit `aa7c4b2`, `127.0.0.1` forced); URL still unresolved — check Pi-hole DNS entry and Traefik router label (`traefik.http.routers.homepage.rule`)

#### Arr Stack Secrets
- [ ] Verify arr stack `_v2` Docker secrets are deployed to Swarm and `stack-arr.yml` has been redeployed with new API key secrets — memory file `project_arr_exportarr_state.md` notes redeploy still pending [priority:: 1]

---

### App-Specific Exporters (after Phase 3 service migration)

|Service|Exporter|Status|
|---|---|---|
|Sonarr|`exportarr`|✅ Active|
|Radarr|`exportarr`|✅ Active|
|Home Assistant|Built-in Prometheus integration|Pending|
|UniFi|`unifi-poller`|✅ Deployed 2026-04-29 — PR #14|
|Traefik|Built-in metrics — `metrics` entrypoint `:8080` + `prometheus` block|✅ PR #24 2026-05-08 — Prometheus was scraping but metrics endpoint was missing|

- [x] `exportarr` deployed for Sonarr and Radarr [priority:: 3] ✅ 2026-05-20 — PR #39 & PR #49; sonarr-exporter port 9707, radarr-exporter port 9708; API keys migrated to prefix-free `API_KEY_FILE` secrets (newline-free `_v2` secrets created via `printf`), scrape target IPs corrected from `.50.30` to `10.0.50.51`, and verified as fully active/UP.
- [ ] Home Assistant Prometheus integration enabled [priority:: 3]
- [ ] Deploy node_exporter on pihole1 (10.0.60.20) and pihole2 (10.0.60.21); add scrape targets to `prometheus.yml` [priority:: 3] #Later
- [x] `unifi-poller` deployed ✅ 2026-04-29 — PR #14; scrape target at `tasks.controller_unpoller:9130`

---

### Uptime Kuma

> [!note] Stack deployed and routing confirmed ✅ 2026-05-21. PR #49 fixed hostname (`uptime-kuma.home.purvishome.com`), added `traefik.swarm.network=traefik-public`, and `internal-only@file` middleware.

- [x] Uptime Kuma deployed via Portainer ✅ 2026-05-08 — `https://uptime-kuma.home.purvishome.com`
- [x] Homepage dashboard — deployed ✅ 2026-05-22 — PR #23 merged; container healthy at localhost:3000; hostname `dashboard.home.purvishome.com`; monitoring network bug fixed; Portainer redeploy pending [priority:: 2]
- [x] Fix Prometheus + Grafana monitors — use internal Swarm DNS (`http://monitoring_prometheus:9090`, `http://monitoring_grafana:3000`) ✅ 2026-05-08
- [x] Fix Transmission monitor — added `arr_arr_default` network to uptime-kuma stack, monitor via `http://gluetun:9091` ✅ 2026-05-08 PR `claude/exciting-brahmagupta-e8f897`
- [x] Fix InfluxDB 502 Bad Gateway — InfluxDB was missing from `traefik-public` network ✅ 2026-05-08 (same PR)
- [x] Merge PR `claude/exciting-brahmagupta-e8f897` and redeploy uptime-kuma stack [priority:: 1] ✅ 2026-05-16
- [x] Add arr service monitors (Sonarr, Radarr, Prowlarr) ✅ 2026-05-08
- [x] Investigate Tautulli — service appears down independently of Uptime Kuma [priority:: 2] ✅ 2026-05-16
- [x] Add remaining monitors: pfSense (`https://10.0.60.1`), Home Assistant (`http://10.0.60.42:8123`) ✅ 2026-05-08
- [x] Create Uptime Kuma public status page and note the slug [priority:: 2] ✅ 2026-05-26 — slug `homelab`; live at `uptime-kuma.home.purvishome.com/status/homelab`
- [x] Update Homepage `widgets.yaml` kuma slug from `default` to the actual status page slug [priority:: 2] ✅ 2026-05-26 — PR #59 merged; slug set to `homelab`

---

### Dashboard Build-out

|Dashboard|Grafana ID|Status|
|---|---|---|
|Node Exporter Full|1860|✅ Phase 1.5|
|Docker cAdvisor|193|✅ Phase 1.5|
|Docker Swarm|13639|✅ Phase 1.5|
|Traefik v2/v3|15489|✅ Phase 1.5|
|Proxmox|10347|✅ 2026-04-30 (pve-exporter)|
|UniFi UAP|11311|✅ 2026-05-07 (imported)|
|UniFi Clients|11315|✅ 2026-05-07 (imported)|
|UniFi Switches|11312|✅ 2026-05-07 (imported)|
|Backup Status|custom|✅ 2026-05-07 — PR #18; `backup-status.json`|
|Loki + Promtail logs|15141|Pending — Loki data source connected, dashboard import not yet done|
|Custom homelab overview|—|Pending — after all sources connected|

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