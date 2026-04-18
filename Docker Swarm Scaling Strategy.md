---
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Scaling
---

# Docker Swarm Scaling Strategy

> [!info] Purpose This document outlines the strategy for creating VM templates and scaling Docker Swarm clusters in a home lab environment with multiple VLANs.

## Overview

This strategy focuses on creating clean, repeatable deployments using base templates and infrastructure-as-code principles rather than templating working VMs with services pre-installed.

## Architecture Principles

### Single Interface Per VM

- Each Swarm VM has **one network interface** on its designated VLAN
- All inter-VLAN routing handled by pfSense
- Avoids Docker networking conflicts
- Clean, predictable routing table

### External Configuration Storage

- Docker Compose files in version control (git) or NFS
- Traefik configs externally stored
- Easy to replicate across nodes
- No config drift

### Shared Persistent Storage

- Docker volumes on NFS/Ceph/GlusterFS
- Database containers with external volumes
- No data migration needed when scaling
- High availability for stateful services

---

## Strategy: Template + Bootstrap Script

### Phase 1: Create Base Template

> [!warning] Do NOT template your current working VM Your current VM has specific configs, IPs, and potential cruft. Create a clean base instead.

#### Base Template Specifications

**VM Configuration:**

- **OS:** Ubuntu 22.04 LTS or Debian 12
- **Resources:** 2-4 vCPU, 4-8GB RAM, 32GB disk
- **Network:** Single interface (will be configured per VLAN)
- **Hostname:** `swarm-template` (will be changed on deployment)

**Installed Software:**

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Install docker-compose
sudo apt install docker-compose-plugin -y

# Install useful tools
sudo apt install -y \
  vim \
  git \
  htop \
  net-tools \
  nfs-common \
  curl \
  wget \
  traceroute \
  tcpdump
```

**Network Configuration (Placeholder):**

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 10.0.0.10/24  # Placeholder - will be updated
      routes:
        - to: default
          via: 10.0.0.1  # Placeholder - will be updated
      nameservers:
        addresses: [10.0.0.1, 8.8.8.8]
```

**SSH Configuration:**

- Enable SSH: `sudo systemctl enable ssh`
- Add your SSH keys to `~/.ssh/authorized_keys`
- Disable password auth (optional but recommended)

**Clean Up Before Templating:**

```bash
# Clear bash history
cat /dev/null > ~/.bash_history && history -c

# Clear cloud-init (if used)
sudo cloud-init clean

# Clear machine-id (will regenerate on boot)
sudo truncate -s 0 /etc/machine-id
sudo rm /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id

# Clear SSH host keys (will regenerate on boot)
sudo rm /etc/ssh/ssh_host_*

# Clear logs
sudo find /var/log -type f -delete

# Power off and convert to template
sudo poweroff
```

### Phase 2: Configuration Storage Structure

Create a centralized location for all Docker configurations.

#### Option A: Git Repository (Recommended)

```
docker-swarm-configs/
├── README.md
├── traefik/
│   ├── docker-compose.yml
│   ├── traefik.yml
│   └── dynamic/
│       ├── proxmox.yml
│       ├── other-service.yml
│       └── middleware.yml
├── stacks/
│   ├── monitoring/
│   │   └── docker-compose.yml
│   ├── databases/
│   │   └── docker-compose.yml
│   └── apps/
│       └── docker-compose.yml
├── scripts/
│   ├── bootstrap-manager.sh
│   └── deploy-all.sh
└── docs/
    └── deployment-notes.md
```

#### Option B: NFS Share

```bash
# On NAS/File Server
/mnt/docker-configs/
├── traefik/
├── stacks/
└── scripts/

# Mount on Swarm nodes
sudo mkdir -p /mnt/configs
sudo mount -t nfs nas.local:/mnt/docker-configs /mnt/configs

# Add to /etc/fstab for persistence
nas.local:/mnt/docker-configs /mnt/configs nfs defaults 0 0
```

### Phase 3: Bootstrap Script

Create a script to configure new Swarm managers from the template.

**File: `bootstrap-manager.sh`**

```bash
#!/bin/bash
set -e

# Configuration variables
NODE_NAME=$1
VLAN=$2
IP_ADDRESS=$3
GATEWAY=$4
SWARM_TOKEN=$5  # Optional: for joining existing cluster

if [ -z "$NODE_NAME" ] || [ -z "$VLAN" ] || [ -z "$IP_ADDRESS" ] || [ -z "$GATEWAY" ]; then
    echo "Usage: $0 <node-name> <vlan> <ip-address> <gateway> [swarm-token]"
    echo "Example: $0 swarm-mgr1 60 10.0.60.10 10.0.60.1"
    exit 1
fi

echo "=== Configuring Swarm Manager: $NODE_NAME ==="

# 1. Set hostname
echo "Setting hostname to $NODE_NAME..."
sudo hostnamectl set-hostname $NODE_NAME
echo "127.0.0.1 localhost" | sudo tee /etc/hosts
echo "127.0.1.1 $NODE_NAME" | sudo tee -a /etc/hosts

# 2. Configure network
echo "Configuring network: VLAN $VLAN, IP $IP_ADDRESS..."
sudo tee /etc/netplan/00-installer-config.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - $IP_ADDRESS/24
      routes:
        - to: default
          via: $GATEWAY
      nameservers:
        addresses: [$GATEWAY, 8.8.8.8]
EOF

sudo netplan apply

# 3. Wait for network
echo "Waiting for network..."
sleep 5
ping -c 3 $GATEWAY

# 4. Initialize or join Swarm
if [ -z "$SWARM_TOKEN" ]; then
    echo "Initializing new Swarm cluster..."
    docker swarm init --advertise-addr $IP_ADDRESS
    echo "=== Save these tokens for joining workers/managers ==="
    docker swarm join-token manager
    docker swarm join-token worker
else
    echo "Joining existing Swarm cluster..."
    docker swarm join --token $SWARM_TOKEN --advertise-addr $IP_ADDRESS <MANAGER_IP>:2377
fi

# 5. Create directories
echo "Creating directories..."
sudo mkdir -p /opt/docker/{traefik,configs,data}

# 6. Mount NFS configs (optional)
# sudo mkdir -p /mnt/configs
# sudo mount -t nfs nas.local:/mnt/docker-configs /mnt/configs
# echo "nas.local:/mnt/docker-configs /mnt/configs nfs defaults 0 0" | sudo tee -a /etc/fstab

# 7. Clone git repository (optional)
# cd /opt/docker/configs
# git clone https://github.com/yourusername/docker-swarm-configs.git

echo "=== Bootstrap complete! ==="
echo "Node: $NODE_NAME"
echo "IP: $IP_ADDRESS"
echo "Next steps:"
echo "1. Deploy your stacks: docker stack deploy -c docker-compose.yml <stack-name>"
echo "2. Verify services: docker service ls"
echo "3. Check logs: docker service logs <service-name>"
```

**Make it executable:**

```bash
chmod +x bootstrap-manager.sh
```

---

## Deployment Workflows

### Deploying First Manager (VLAN 60 - Services)

1. **Clone template VM in Proxmox:**
    
    - Right-click template → Clone
    - Name: `swarm-mgr1`
    - Full Clone (not linked)
    - Add network device: Bridge `vmbr0`, VLAN Tag `60`
2. **Boot and run bootstrap:**
    
    ```bash
    # SSH into new VM (using template's IP initially)
    ssh user@<template-ip>
    
    # Run bootstrap script
    ./bootstrap-manager.sh swarm-mgr1 60 10.0.60.10 10.0.60.1
    
    # Save the join tokens shown at the end!
    ```
    
3. **Deploy Traefik:**
    
    ```bash
    cd /opt/docker/traefik
    docker stack deploy -c docker-compose.yml traefik
    ```
    

### Adding Second Manager (Different VLAN)

1. **Clone template VM:**
    
    - Name: `swarm-mgr2`
    - Add network device: Bridge `vmbr0`, VLAN Tag `70`
2. **Bootstrap with join token:**
    
    ```bash
    ./bootstrap-manager.sh swarm-mgr2 70 10.0.70.10 10.0.70.1 <MANAGER_JOIN_TOKEN>
    ```
    
3. **Verify cluster:**
    
    ```bash
    docker node ls
    ```
    

### Adding Worker Node

Same process, but use worker join token:

```bash
./bootstrap-worker.sh swarm-worker1 80 10.0.80.10 10.0.80.1 <WORKER_JOIN_TOKEN>
```

---

## Persistent Data Strategy

> [!important] Use Shared Storage from Day 1 Don't store data locally on Swarm nodes. Use NFS/Ceph/GlusterFS for all persistent data.

### NFS Volumes Example

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  traefik:
    image: traefik:latest
    networks:
      - traefik-public
    volumes:
      - traefik-certs:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager

volumes:
  traefik-certs:
    driver: local
    driver_opts:
      type: nfs
      o: addr=nas.local,rw,nolock
      device: ":/mnt/docker/traefik-certs"

networks:
  traefik-public:
    driver: overlay
    attachable: true
```

### Local to NFS Migration

If you already have local data:

```bash
# 1. Create NFS volume
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=nas.local,rw \
  --opt device=:/mnt/docker/traefik-certs \
  traefik-certs-nfs

# 2. Copy data
docker run --rm \
  -v traefik-certs:/from \
  -v traefik-certs-nfs:/to \
  alpine sh -c "cd /from && cp -av . /to"

# 3. Update docker-compose.yml to use new volume
# 4. Redeploy stack
```

---

## VLAN Assignments

Keep track of your VLAN assignments:

|VLAN|Purpose|Subnet|Example Nodes|
|---|---|---|---|
|60|Services/Apps|10.0.60.0/24|swarm-mgr1, traefik|
|70|Additional Services|10.0.70.0/24|swarm-mgr2|
|80|DMZ|10.0.80.0/24|swarm-worker1|
|90|Management|10.0.90.0/24|Proxmox, monitoring|

---

## pfSense Firewall Rules

For each new VLAN, add rules to allow inter-VLAN routing:

**Firewall > Rules > VLAN[X]**

|Action|Source|Destination|Port|Description|
|---|---|---|---|---|
|Pass|VLAN60 net|VLAN90 net|8006|Traefik to Proxmox|
|Pass|VLAN60 net|VLAN70 net|Any|Swarm communication|
|Pass|VLAN70 net|VLAN60 net|Any|Return traffic|

---

## Current Working Configuration

> [!success] Reference: Working Traefik Config (swarm-mgr1) Save your current working configuration for reference.

### Network Config

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 10.0.60.30/24
      routes:
        - to: default
          via: 10.0.60.1
      nameservers:
        addresses: [10.0.60.1, 8.8.8.8]
```

### Traefik Stack

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  traefik:
    image: traefik:latest
    network_mode: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/traefik.yml:ro
      - ./dynamic/:/dynamic/:ro
      - ./letsencrypt:/letsencrypt
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      restart_policy:
        condition: on-failure
```

**dynamic/proxmox.yml:**

```yaml
http:
  routers:
    proxmox-rtr:
      rule: "Host(`pve.home.purvishome.com`)"
      entryPoints:
        - https
      service: proxmox-svc
      tls:
        certResolver: cloudflare
      middlewares:
        - proxmox-headers

  services:
    proxmox-svc:
      loadBalancer:
        servers:
          - url: "https://10.0.90.50:8006"
        serversTransport: pve

  serversTransports:
    pve:
      insecureSkipVerify: true

  middlewares:
    proxmox-headers:
      headers:
        customRequestHeaders:
          X-Forwarded-Proto: "https"
```

---

## Troubleshooting

### VM Can't Reach Other VLANs

**Symptoms:**

- `No route to host` or `Destination Host Unreachable`

**Solutions:**

1. Verify pfSense firewall rules allow traffic
2. Clear pfSense state table: **Diagnostics > States > Reset States**
3. Check VM routing: `ip route`
4. Test gateway: `ping <gateway-ip>`

### Docker Blocking Traffic

**Symptoms:**

- Direct interface pings fail
- Traffic works when routing through default gateway

**Solution:**

- Use single interface per VM (already implemented)
- Let pfSense handle inter-VLAN routing
- Don't use multiple interfaces on Docker Swarm nodes

### Certificate Errors

**Symptoms:**

- `x509: certificate is valid for X, not Y`

**Solution:**

```bash
# On Proxmox host
rm /etc/pve/local/pve-ssl.pem
rm /etc/pve/local/pve-ssl.key
pvecm updatecerts -f
systemctl restart pveproxy
```

### Services Not Deploying

**Check:**

```bash
# View service status
docker service ls

# View service logs
docker service logs <service-name> -f

# View service tasks
docker service ps <service-name>

# Inspect service
docker service inspect <service-name>
```

---

## Best Practices

### ✅ Do This

- Use single network interface per VM
- Store configs in git or NFS
- Use shared storage (NFS) for persistent data
- Create clean base templates
- Document VLAN assignments
- Version control docker-compose files
- Use host network mode for Traefik in Swarm
- Let pfSense handle inter-VLAN routing

### ❌ Avoid This

- Multiple interfaces on Swarm nodes (routing conflicts)
- Storing data locally on nodes (hard to migrate)
- Templating working VMs (carries forward cruft)
- Hardcoding IPs in configs (use variables)
- Running production services on manager nodes (separate workers)
- Disabling Docker iptables (causes issues)

---

## Scaling Checklist

When adding a new Swarm manager:

- [ ] Clone base template VM #Later
- [ ] Configure network device with correct VLAN tag #Later
- [ ] Boot VM and run bootstrap script #Later 
- [ ] Verify network connectivity to other VLANs #Later
- [ ] Join Swarm cluster with manager token #Later
- [ ] Deploy required stacks #Later
- [ ] Add pfSense firewall rules if new VLAN #Later
- [ ] Update documentation with new node info #Later
- [ ] Test service failover #Later

---

## Future Improvements

### Load Balancing

- Add multiple Traefik instances across managers
- Use Keepalived for VIP failover
- DNS round-robin across managers

### Monitoring

- Deploy Prometheus + Grafana stack
- Monitor Swarm nodes and services
- Alert on service failures

### Backup Strategy

- Automated backups of configs to git
- NFS snapshot policies
- Docker volume backups

### CI/CD Integration

- GitOps workflow with Flux or ArgoCD
- Automated deployments on git push
- Rolling updates with zero downtime

---

## Related Documentation

- [[proxmox-network-interfaces-config]]
- [[traefik-proxmox-complete-guide]]

## Tags

#docker #swarm #homelab #infrastructure #scaling #templates #automation #devops

---

## Changelog

| Date       | Change                | Author |
| ---------- | --------------------- | ------ |
| 2026-02-15 | Initial documentation | System |