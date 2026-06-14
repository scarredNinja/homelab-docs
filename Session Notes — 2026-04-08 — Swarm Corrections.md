---
date: '2026-04-08T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
session_type: Implementation
branch: main
status: Completed
tags:
  - SessionNotes
  - Traefik
  - Docker
  - DockerSwarm
  - Debugging
---

# Session Notes — 2026-04-08

Corrected docker swarm provision on first manager and then worker.  Have claude explain it further but issues with handling orphaned networks on docker etc. 

After that got traefik deployed in portainer via git repo integration. I will use this in future once I have a working implementation on the main branch. I will do the rest locally to save time. Had issues with the docker version in relation to traefik. Now using latest image. Also had a swarm connection issue on worker and needed to deploy a proxy on manager to surface the services. 

Had a dns issue and now need to test that I can view rhe traefik dashboard and current internal services. Then test the dmz setup to make sure that is correct. 

Then monitoring deployment which I will need the run book update
