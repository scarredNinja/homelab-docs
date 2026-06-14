---
type: swarm-vm
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
tags:
  - DockerSwarm
  - VM
vmid: 202
hostname: worker-monitoring-01
vcpu: 2
ram_gb: 4
disk_gb: 20
vlan_primary: 60
vlan_secondary: null
ip_primary: 10.0.60.41
ip_secondary: null
swarm_role: worker
node_labels:
  - node.labels.zone=monitoring
vm_status: running
post_boot_run: true
swarm_joined: true
last_updated: '2026-06-08T00:00:00.000Z'
status: Active
---


# worker-monitoring-01

Swarm worker node. Dedicated monitoring and dashboard stack  Prometheus, Grafana, Loki, InfluxDB, Uptime Kuma, NewtonFit, and Homepage.

## Notes

- `zone=monitoring` placement constraint
- Prometheus, Grafana, Loki, InfluxDB all deployed and running  2026-04-12
- Uptime Kuma deployed and running  2026-04-24
- NewtonFit fitness & nutrition dashboard deployed and running  2026-05-21
- Homepage dashboard deployed and running  2026-05-22
- Homepage dashboard updated with Proxmox VE and NAS status widgets integration (Secrets mounted for Synology NAS credentials, custom anime background, and glassmorphic CSS applied) - 2026-06-08
- Prometheus solar history migrated from CT109 (5.4 GB), chown 65534 applied  2026-04-24
- InfluxDB v2 data migrated from CT109  2026-04-24
- Grafana dashboards migrated  **data sources need updating to Swarm service DNS names**
- Proxmox InfluxDB push configured  data arriving in InfluxDB Data Explorer 
- Overlay network: `monitoring_monitoring`  `10.200.1.0/24` (pinned after Bug #22 fix)
- `internal-only` middleware missing from InfluxDB router  security task outstanding (see [[Docker Swarm Infrastructure Runbook]] Appendix F)

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Swarm Topology]]
- [[Service - prometheus]]
- [[Service - grafana]]
- [[Service - influxdb]]
- [[Service - uptime-kuma]]
- [[Service - newtonfit]]
- [[Service - homepage]]
