---
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - Traefik
  - DockerSwarm
---

## To Do

- [ ] Add in details of traefik setup and point to files in git [priority:: 4]
## Details

- Network: proxy


## Directory Structure


./traefik
├── data
│   ├── acme.json
│   ├── dynamic.yml
│   ├── dynamic.yml
│   └── traefik.yml
└── cf_api_token.txt
└── portainer-stack.yml

**Files**: [link to git](https://github.com/scarredNinja/docker-swarm-home)
## Commands

docker network create --driver overlay proxy