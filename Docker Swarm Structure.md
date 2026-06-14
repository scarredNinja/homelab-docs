---
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
tags:
  - docker-swarm
  - reference
status: Reference
---

## Template

- Base ISO Image: noble-server-cloudimg-amd64.img
- ssh key location: DJ PC - homelab_ed25519.pub
- Main User: docker
- Services created:
	- Docker Engine
	- Qemu-guest-agent

Main Checks:
 - [x] Can ssh after loading [priority:: 1] ✅ 2026-03-15
 - [x] Qeum Guest Agent Runs [priority:: 1] ✅ 2026-03-19
 - [x] Docker Starts [priority:: 1] ✅ 2026-03-19
## Manager

/home/sussh/
 ├── swarm-monitoring/             # Manager-specific monitoring setup
 │    ├── prometheus.yml     # Prometheus config
 │
 ── /mnt/backups/                # Swarm + manager backup scripts
 │    └── etcd-snapshots/    # Optional etcd or Raft snapshots
 │
 │
 └── cloudinit/              # Cloud-init seed overrides
      ├── user-data.yaml
      └── meta-data.yaml


## Worker

/home/sussh/
 ├── scripts/               # Provisioning helper scripts
 │
 ├── volumes/                # Local Docker volumes or mounted shares
 │    ├── data/              # Default local worker volume
 │    ├── nfs/               # Mounted Synology NFS shares (/mnt/media/<name>)
 │    └── zfs/               # Optional ZFS pools if created
 │
 ├── shared/                 # SMB/NFS mountpoint (if needed)
 │
 └── cloudinit/              # Cloud-init configs for workers
      ├── user-data.yaml
      └── meta-data.yaml
      
      
## Storage map

| **Host ZFS Dataset** | **Host Mount Path** | **Virtio-FS Tag** | **VM Mount Path**   | **Optimization** |
| -------------------- | ------------------- | ----------------- | ------------------- | ---------------- |
| `rpool/docker-data`  | `/mnt/docker-data`  | `docker-data`     | `/mnt/docker-data`  | 128k (Configs)   |
| `rpool/docker-db`    | `/mnt/docker-db`    | `docker-db`       | `/mnt/docker-db`    | 16k (Postgres)   |
| `rpool/docker-tsdb`  | `/mnt/docker-tsdb`  | `docker-tsdb`     | `/mnt/docker-tsdb`  | 16k (Prometheus) |
| `rpool/docker-swarm` | `/mnt/docker-swarm` | `docker-swarm`    | `/mnt/docker-swarm` | 128k (Global)    |
