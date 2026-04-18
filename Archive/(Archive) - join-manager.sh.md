#!/bin/bash
# =========================
# Join additional manager
# =========================

set -e

MANAGER_IP="$1"
MANAGER_TOKEN="$2"

if [[ -z "$MANAGER_IP" || -z "$MANAGER_TOKEN" ]]; then
    echo "Usage: $0 <manager-ip> <manager-token>"
    exit 1
fi

docker swarm join --token "$MANAGER_TOKEN" "$MANAGER_IP":2377
