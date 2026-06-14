---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
# Docker Swarm Worker Template Setup Guide

> [!info] Overview This guide covers converting a Docker Swarm Manager template into a Worker template, including cleanup, configuration, and verification steps.

## Prerequisites

- [ ] Manager template VM cloned

---

## Phase 1: Cleanup Manager Template

### Stop Manager Services ✅

```bash
# Stop and remove any running monitoring services
docker stack rm monitoring 2>/dev/null || true

# Wait for services to stop
sleep 10

# Remove any running containers
docker container prune -f
docker volume prune -f
docker network prune -f
```

### Leave Swarm ✅

> [!warning] Important This will disconnect the node from any existing swarm

```bash
# Check if in swarm and leave
if docker info | grep -q "Swarm: active"; then
    echo "Leaving swarm..."
    docker swarm leave --force
fi

# Verify swarm is inactive
sudo docker info | grep "Swarm:"  # Should show "inactive"
```

### Remove Manager-Specific Files ✅

```bash
# Remove manager monitoring directory
rm -rf /home/manager/docker-monitoring

# Remove manager-specific deployment script
sudo rm -f /home/manager/scripts/deploy-monitoring.sh

# Keep the sync script as it's useful for workers too
# Keep /home/manager/docker-monitoring-local as reference if needed
```

### Deep Clean Docker System ✅

```bash
# Remove any manager monitoring data
sudo rm -rf /mnt/docker-config/* 2>/dev/null || true

# Remove any prometheus/grafana data directories
sudo rm -rf /home/manager/docker-monitoring-local/data/* 2>/dev/null || true

# Clean docker system
sudo docker system prune -a -f --volumes
```

---

## Phase 2: Add Worker Components

### Create Worker Directory Structure

```bash
# Create worker directory structure
sudo mkdir -p /home/manager/docker-worker-local/{data/cadvisor}
sudo mkdir -p /home/manager/scripts
```

### Worker Setup Script

> [!note] File Location `/home/manager/scripts/setup-worker.sh`

```bash
#!/bin/bash

# Configuration
PC_IP="10.0.4.100"
SHARE_NAME="DockerSwarmWorkerConfigs"
MOUNT_POINT="/mnt/docker-config"
LOCAL_CONFIG="/home/manager/docker-worker-local"
TIMEOUT=5

echo "=== Docker Swarm Worker Setup ==="
echo "Checking PC availability at $PC_IP..."

# Function to check if PC is reachable
check_pc_availability() {
    ping -c 1 -W $TIMEOUT $PC_IP > /dev/null 2>&1
    return $?
}

# Function to mount SMB share
mount_smb_share() {
    echo "Attempting to mount SMB share..."
    
    # Check if already mounted
    if mountpoint -q $MOUNT_POINT; then
        echo "SMB share already mounted"
        return 0
    fi
    
    # Create mount point if it doesn't exist
    sudo mkdir -p $MOUNT_POINT
    
    # Mount the share
    sudo mount -t cifs //${PC_IP}/${SHARE_NAME} $MOUNT_POINT \
        -o credentials=/etc/cifs-credentials,uid=1000,gid=1000,iocharset=utf8,soft,timeo=30
    
    if [ $? -eq 0 ]; then
        echo "✅ SMB share mounted successfully"
        return 0
    else
        echo "❌ Failed to mount SMB share"
        return 1
    fi
}

# Function to show swarm join instructions
show_join_instructions() {
    echo ""
    echo "=== Worker Node Ready ==="
    echo "📋 To join this worker to a swarm, run this command from the MANAGER:"
    echo ""
    echo "   docker swarm join-token worker"
    echo ""
    echo "Then copy and run the resulting command on this worker node."
    echo ""
    echo "🔍 Worker services will be visible at:"
    LOCAL_IP=$(hostname -I | awk '{print $1}')
    echo "   📈 cAdvisor: http://$LOCAL_IP:8080 (when running)"
    echo ""
}

# Function to check swarm status
check_swarm_status() {
    if docker info | grep -q "Swarm: active"; then
        echo "✅ Node is part of a Docker Swarm"
        SWARM_ROLE=$(docker info | grep "Is Manager" | awk '{print $3}')
        if [ "$SWARM_ROLE" = "false" ]; then
            echo "🔧 Role: Worker Node"
        else
            echo "🔧 Role: Manager Node"
        fi
        
        echo ""
        echo "📊 Swarm Status:"
        docker node ls 2>/dev/null || echo "   (Only managers can view node list)"
        
    else
        echo "⚠️  Node is not part of a Docker Swarm yet"
        show_join_instructions
    fi
}

# Function to setup worker-specific monitoring (if not part of swarm)
setup_local_monitoring() {
    if ! docker info | grep -q "Swarm: active"; then
        echo "🔧 Setting up local cAdvisor for standalone monitoring..."
        
        # Stop any existing container
        docker stop cadvisor-local 2>/dev/null || true
        docker rm cadvisor-local 2>/dev/null || true
        
        # Start cAdvisor locally
        docker run -d \
            --name cadvisor-local \
            --restart unless-stopped \
            -p 8080:8080 \
            -v /:/rootfs:ro \
            -v /var/run:/var/run:rw \
            -v /sys:/sys:ro \
            -v /var/lib/docker/:/var/lib/docker:ro \
            gcr.io/cadvisor/cadvisor:latest
        
        if [ $? -eq 0 ]; then
            echo "✅ Local cAdvisor started successfully"
            LOCAL_IP=$(hostname -I | awk '{print $1}')
            echo "📈 cAdvisor available at: http://$LOCAL_IP:8080"
        fi
    else
        echo "ℹ️  Node is in swarm - monitoring handled by manager"
    fi
}

# Main logic
echo "🔧 Setting up worker node..."

# Check SMB availability
if check_pc_availability; then
    echo "✅ PC is online!"
    mount_smb_share
else
    echo "⚠️  PC is offline, worker will use local resources only"
fi

# Check Docker service
if systemctl is-active --quiet docker; then
    echo "✅ Docker service is running"
else
    echo "🔧 Starting Docker service..."
    sudo systemctl start docker
    sudo systemctl enable docker
fi

# Check swarm status
check_swarm_status

# Setup monitoring if needed
setup_local_monitoring

echo ""
echo "=== Worker Node Setup Complete ==="
echo ""
echo "📝 Common commands for workers:"
echo "   Check swarm status:    docker info | grep Swarm"
echo "   View running services: docker service ls"
echo "   View containers:       docker ps"
echo "   Mount SMB manually:    sudo mount -t cifs //${PC_IP}/${SHARE_NAME} ${MOUNT_POINT} -o credentials=/etc/cifs-credentials,uid=1000,gid=1000"
```

### Swarm Join Helper Script

> [!note] File Location `/home/manager/scripts/join-swarm.sh`

```bash
#!/bin/bash

echo "=== Docker Swarm Worker Join Helper ==="
echo ""

# Function to validate join command
validate_join_command() {
    if [[ $1 =~ ^docker\ swarm\ join\ --token\ SWMTKN-.*\ [0-9]+\.[0-9]+\.[0-9]+\.[0-9]+:[0-9]+$ ]]; then
        return 0
    else
        return 1
    fi
}

# Check if already in swarm
if docker info | grep -q "Swarm: active"; then
    echo "⚠️  This node is already part of a Docker Swarm!"
    echo ""
    echo "Current status:"
    docker info | grep -E "Swarm|NodeID|Is Manager"
    echo ""
    echo "To leave current swarm: docker swarm leave"
    echo ""
    exit 1
fi

echo "This worker is ready to join a Docker Swarm."
echo ""
echo "📋 Steps to join:"
echo "1. On your MANAGER node, run: docker swarm join-token worker"
echo "2. Copy the entire 'docker swarm join...' command"
echo "3. Paste it below when prompted"
echo ""

# Get join command from user
read -p "Paste the join command here: " join_command

# Validate the command
if validate_join_command "$join_command"; then
    echo ""
    echo "🔧 Executing join command..."
    echo "$join_command"
    echo ""
    
    # Execute the join command
    eval $join_command
    
    if [ $? -eq 0 ]; then
        echo ""
        echo "✅ Successfully joined the Docker Swarm!"
        echo ""
        echo "📊 Node status:"
        docker info | grep -E "Swarm|NodeID|Is Manager"
        
        # Stop local cAdvisor if it was running
        if docker ps | grep -q cadvisor-local; then
            echo ""
            echo "🔧 Stopping local cAdvisor (swarm will handle monitoring)..."
            docker stop cadvisor-local
            docker rm cadvisor-local
        fi
        
        echo ""
        echo "🎉 Worker node is now part of the swarm and ready to receive tasks!"
    else
        echo ""
        echo "❌ Failed to join swarm. Please check the join command and try again."
    fi
else
    echo ""
    echo "❌ Invalid join command format. Please make sure you copied the complete command."
    echo ""
    echo "Expected format:"
    echo "docker swarm join --token SWMTKN-... <manager-ip>:2377"
fi
```

### Make Scripts Executable

```bash
# Make scripts executable
chmod +x /home/manager/scripts/setup-worker.sh
chmod +x /home/manager/scripts/join-swarm.sh
chmod +x /home/manager/scripts/sync-smb-configs.sh
```

---

## Phase 3: Template Verification

### Pre-Template Checklist

> [!check] Verification Steps Complete these checks before creating the template

#### Docker Configuration

```bash
# Check Docker is installed
docker --version

# Check Docker service status
systemctl status docker

# Verify user is in docker group
groups | grep docker

# Test Docker without sudo
docker run hello-world
```

#### Directory Structure

```bash
# Check all directories exist
ls -la /home/manager/
ls -la /home/manager/scripts/
ls -la /home/manager/docker-worker-local/

# Verify scripts are present and executable
ls -la /home/manager/scripts/
```

Expected output:

```
-rwxr-xr-x setup-worker.sh
-rwxr-xr-x join-swarm.sh
-rwxr-xr-x sync-smb-configs.sh
```

#### SMB Configuration

```bash
# Check CIFS tools are installed
which mount.cifs

# Check credentials file exists
sudo ls -la /etc/cifs-credentials

# Should show: -rw------- root root cifs-credentials

# Test SMB connection (optional - only if PC is on)
smbclient -L //10.0.4.100 -U DockerSMB
```

#### System State

```bash
# Docker should be enabled but no swarm
systemctl is-enabled docker  # Should show "enabled"
docker info | grep "Swarm:"  # Should show "Swarm: inactive"

# No containers should be running
docker ps -a

# Check system resources
df -h  # Check disk space
free -h  # Check memory
```

### Final Cleanup Script

> [!code] Quick Cleanup Run this before creating template

```bash
#!/bin/bash
echo "=== Final Worker Template Cleanup ==="

# Stop all services
docker stack rm monitoring 2>/dev/null || true
sleep 10

# Leave swarm if active
docker swarm leave --force 2>/dev/null || true

# Remove manager-specific files
rm -rf /home/manager/docker-monitoring
rm -f /home/manager/scripts/deploy-monitoring.sh

# Create worker structure
mkdir -p /home/manager/docker-worker-local/{data/cadvisor}

# Clean docker completely
docker system prune -a -f --volumes

# Clear logs and history
sudo journalctl --vacuum-time=1d
history -c

echo "✅ Worker template cleanup complete"
echo ""
echo "Verification:"
docker info | grep "Swarm:"
echo "Containers: $(docker ps -a | wc -l) (should be 1 - header only)"
echo "Images: $(docker images | wc -l) (should be minimal)"
```

### Comprehensive Verification Command

```bash
# Run this comprehensive check
echo "=== Worker Template Verification ==="
echo "Docker version: $(docker --version)"
echo "User in docker group: $(groups | grep -o docker || echo 'NO')"
echo "Scripts executable: $(ls -la /home/manager/scripts/*.sh | wc -l) files"
echo "SMB credentials: $(sudo test -f /etc/cifs-credentials && echo 'Present' || echo 'Missing')"
echo "Swarm status: $(docker info | grep 'Swarm:' | awk '{print $2}')"
echo "Running containers: $(docker ps -q | wc -l)"
echo "=== Verification Complete ==="
```

---

## Phase 4: Template Usage

### After Deploying Worker from Template

1. **Initial Setup**
    
    ```bash
    /home/manager/scripts/setup-worker.sh
    ```
    
2. **Join Swarm**
    
    ```bash
    /home/manager/scripts/join-swarm.sh
    ```
    
3. **Verify Status**
    
    ```bash
    docker info | grep Swarm
    docker node ls  # (from manager)
    ```
    

### Common Worker Operations

|Task|Command|
|---|---|
|Check swarm status|`docker info \| grep Swarm`|
|View running containers|`docker ps`|
|View services|`docker service ls`|
|Leave swarm|`docker swarm leave`|
|Mount SMB manually|`sudo mount -t cifs //10.0.4.100/DockerSwarmWorkerConfigs /mnt/docker-config -o credentials=/etc/cifs-credentials,uid=1000,gid=1000`|

---

## Troubleshooting

### Common Issues

> [!bug] SMB Mount Fails
> 
> - Check PC is online: `ping 10.0.4.100`
> - Verify credentials: `smbclient -L //10.0.4.100 -U DockerSMB`
> - Check firewall settings on PC

> [!bug] Cannot Join Swarm
> 
> - Verify manager is accessible
> - Check firewall ports (2377, 7946, 4789)
> - Ensure join token is valid

> [!bug] Docker Permission Denied
> 
> - Add user to docker group: `sudo usermod -aG docker $USER`
> - Logout and login again
> - Verify: `groups | grep docker`

### Network Requirements

|Port|Protocol|Purpose|
|---|---|---|
|2377|TCP|Swarm management|
|7946|TCP/UDP|Node communication|
|4789|UDP|Overlay network|
|445|TCP|SMB/CIFS|

---

## Tags

#docker #swarm #worker #template #devops #infrastructure
