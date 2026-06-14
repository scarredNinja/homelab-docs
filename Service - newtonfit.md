---
type: swarm-service
project_id: FitnessDev-2026
phase: 'Phase 1: Ingestion & Target Sync'
tags:
  - DockerSwarm
  - Service
  - Fitness
  - Nutrition
service_name: NewtonFit
vm: dev-node-01
swarm_constraint: node.labels.zone == dev
vlan: 40
service_status: running
stack_file: proxmox-swarm/stacks/stack-dev-newtonfit.yml
comment: stack_file is repo-relative (Portainer Git deployment)
port: 8080
external_access: false
traefik_entrypoint: websecure
url_internal: 'https://dev-newtonfit.home.purvishome.com'
zfs_dataset: rpool/docker-data/newtonfit-dev
mount_path: /mnt/docker-data/newtonfit-dev
last_updated: '2026-05-27T00:00:00.000Z'
status: Completed
---

# NewtonFit

A premium, full-stack personal fitness, nutrition, and health tracking dashboard. Pinned to `worker-monitoring-01` inside Docker Swarm.

---

## 🛠️ Architecture & Build Details

NewtonFit is built as a highly responsive, dependency-free full-stack application designed specifically for secure local self-hosting in homelabs.

### 1. Backend API & Storage Engine
* **Technology**: Node.js & Express.
* **Database**: Server-side JSON-file database located at `data/fitness_history.json`.
* **State Management**: On first run, it initializes with baseline targets (Bench/OHP/Deadlift 5x5, lipid panel parameters, macro equations) directly derived from Obsidian notes. 
* **State Mutation**: Merges, deduplicates, and chronologically sorts incoming fitness, bodyweight, and nutrition CSV uploads (supporting FitNotes, LoseIt!, and Samsung Health).

### 2. Frontend Design (Nord & Glassmorphism)
* **Styling**: Vanilla CSS utilizing backdrop-blur, custom translucent glass cards (`rgba(36, 41, 51, 0.65)`), dynamic linear gradient backgrounds, and soft animations.
* **Colors**: Curated Nord palette featuring Nordic Peach (`#d08770`), Amber Gold (`#ebcb8b`), Emerald Green (`#a3be8c`), and Frost Blue (`#88c0d0`).
* **Visualizations**: Lightweight, responsive raw SVG graphs for linear bodyweight gains and Resting Heart Rate (RHR) trends, maintaining 100% offline functionality.
* **Calculators**: Integrated dietitian-approved reactive macronutrient calculator using bodyweight scaling formulas.

### 3. Build & Deployment Pipeline
* **Local Build Automation**: Managed via `build-and-push.ps1` in the workspace root. It runs `docker build`, tags the container, and publishes it directly to a private container registry.
* **Registry**: Hosted privately under GitHub Container Registry: `ghcr.io/scarredninja/newtonfit:latest`.
* **GitOps Swarm Deployment**: Integrated directly into Portainer Git sync via the `docker-swarm-home` deployment repository. Stacks are automatically updated on pushes to the `main` branch.

---

## 🚀 Deployment Config

### Docker Swarm Stack (`stack-newtonfit.yml`)
The stack runs on `worker-monitoring-01` (`zone == monitoring`) with Traefik v3 wildcard SSL routing:

```yaml
version: "3.8"

# NewtonFit — worker-monitoring-01
#
# Placement: node.labels.zone == monitoring  (worker-monitoring-01 only)
#
# Pre-requisites (run once on manager):
#   docker node update --label-add zone=monitoring <worker-monitoring-01-node-id>
#
# Provisioning pre-requisites on worker-monitoring-01:
#   - /mnt/docker-data/newtonfit  (virtiofs share; ZFS dataset)
#
# Deploy via Portainer: Stacks → Add stack → Git Repository:
#   URL: https://github.com/scarredNinja/docker-swarm-home
#   Path: proxmox-swarm/stacks/stack-newtonfit.yml
#   Name: "newtonfit"

services:
  newtonfit:
    image: ghcr.io/scarredninja/newtonfit:latest

    volumes:
      - /mnt/docker-data/newtonfit:/app/data

    networks:
      - traefik-public

    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.labels.zone == monitoring
      restart_policy:
        condition: on-failure
        delay: 10s
      update_config:
        parallelism: 1
        delay: 10s
        order: stop-first
        failure_action: rollback
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          cpus: "0.05"
          memory: 64M
      labels:
        - "traefik.enable=true"
        - "traefik.swarm.network=traefik-public"
        - "traefik.http.routers.newtonfit.rule=Host(`newtonfit.home.purvishome.com`)"
        - "traefik.http.routers.newtonfit.entrypoints=websecure"
        - "traefik.http.routers.newtonfit.tls=true"
        - "traefik.http.routers.newtonfit.tls.certresolver=cloudflare"
        - "traefik.http.services.newtonfit.loadbalancer.server.port=8080"

networks:
  traefik-public:
    external: true
```

---

## 🧪 Production Seeding & Recovery
1. Create the persistent ZFS dataset path on `worker-monitoring-01`:
   ```bash
   sudo mkdir -p /mnt/docker-data/newtonfit/data
   sudo chown -R root:root /mnt/docker-data/newtonfit
   ```
2. Copy the active `fitness_history.json` from the local workspace directory to `/mnt/docker-data/newtonfit/data/fitness_history.json`.
3. Force redeploy/restart the task in Portainer.

---

## Related Notes

* [[Docker Swarm Details]] — Host mappings and volume metrics
* [[Docker Swarm Infrastructure Runbook]] — Swarm cluster deployment reference
* [[VM - monitoring-01]] — Destination worker VM
* [[NewtonFit - Data Ingestion & Target Sync Strategy]] — Ingestion and bidirectional sync roadmap
