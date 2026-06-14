---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
#!/bin/bash
# =========================
# Join worker node
# =========================

set -e

MANAGER_IP="$1"
WORKER_TOKEN="$2"

if [[ -z "$MANAGER_IP" || -z "$WORKER_TOKEN" ]]; then
    echo "Usage: $0 <manager-ip> <worker-token>"
    exit 1
fi

docker swarm join --token "$WORKER_TOKEN" "$MANAGER_IP":2377
