---
type: swarm-service
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Service
service_name: Portainer
vm: manager-01
swarm_constraint: node.role == manager
vlan: 60
service_status: running
stack_file: /mnt/docker-swarm/stacks/portainer/stack.yml
port: 9443
external_access: false
traefik_entrypoint: websecure
url_internal: https://portainer.home.purvishome.com
zfs_dataset: rpool/docker-data/portainer
mount_path: /mnt/docker-data/portainer
last_updated: 2026-04-02
---

# Portainer

Swarm control plane UI. Runs in agent mode — server on manager, global agent on all nodes.
Accessible at port 9443 directly before Traefik is deployed.

## Notes

- portainer_agent deployed as `mode: global` — auto-schedules on new nodes as they join
- UI not yet verified — pending post-boot completion on manager-01
