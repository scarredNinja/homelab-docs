#!/bin/bash
# =========================
# Initialize Docker Swarm (first manager)
# =========================

set -e

MANAGER_IP="$1"

if [[ -z "$MANAGER_IP" ]]; then
    echo "Usage: $0 <manager-ip>"
    exit 1
fi

# Initialize swarm
docker swarm init --advertise-addr "$MANAGER_IP"

# Output tokens for additional managers and workers
echo "Manager join token:"
docker swarm join-token manager -q

echo "Worker join token:"
docker swarm join-token worker -q
