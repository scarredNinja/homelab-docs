---
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - Traefik
  - DockerSwarm
---

> [!note] Superseded
> The canonical Traefik reference is [[Traefik Routing Architecture]]. The service record (with pre-deploy checklist and deploy status) is [[Service - traefik]].

## Directory Structure

```
/mnt/docker-data/traefik/
├── data/
│   ├── acme.json          (chmod 600 — must be set on Proxmox host)
│   ├── dynamic.yml        (file provider — non-Docker service routes)
│   └── traefik.yml        (static config)
└── cf_api_token.txt       (chmod 600)
```

**Stack file:** `/mnt/docker-swarm/stacks/traefik/stack.yml`
**Git repo:** [docker-swarm-home](https://github.com/scarredNinja/docker-swarm-home)

## Network

```bash
docker network create --driver overlay --attachable traefik-public
```
