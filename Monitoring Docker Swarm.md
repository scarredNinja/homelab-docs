---
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
tags:
  - Monitoring
  - DockerSwarm
status: Reference
---


**Notes:**

- **Prometheus** on monitoring VM scrapes **all managers + workers**.
    
- **Grafana** visualizes metrics from Prometheus.
    
- **cAdvisor** runs on every manager and worker, exposing container metrics.
    
- **Volumes:** NFS/ZFS mounts are used across all nodes; no local duplication needed.
    
- **Scripts:** Only provisioning/Swarm join scripts remain on managers/workers. Monitoring VM handles all observability.

## Setup

- Done:
	- Monitoring stack created for:
		- Prometheus (swarm)
		- ca_advisor
		- node-exporter
		- influxdb (proxmox)
- Next:
	- Need to test connections and data import
