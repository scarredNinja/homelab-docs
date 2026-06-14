---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
#HomeLabRebuild/Virtualisation #HomeLabRebuild/SwarmManager #HomeLabRebuild/SwarmWorker #HomeLabRebuild/Tem

#!/bin/bash
# ==============================================================================
# Docker Swarm VM Provisioning Script (Proxmox)
# Version: v7
# ==============================================================================
#
# Features:
# - VM creation from template
# - VLAN configuration, SSH key injection, user info
# - ZFS volume creation and auto-mount
# - NFS mount setup
# - Docker Swarm bootstrap (first manager) and token saving
# - Worker/manager autojoin
# - Automatic token retrieval back to Proxmox
# ==============================================================================

# --- Config ---
TEMPLATE_ID=101
VMID_BASE=200
BRIDGE_NAME="vmbr0"
DEFAULT_VLAN=85
SSH_PUBLIC_KEY_PATH="/root/.ssh/id_rsa.pub"
SWARM_WORKER_TOKEN_FILE="/root/swarm_worker_token.txt"
SWARM_MANAGER_TOKEN_FILE="/root/swarm_manager_token.txt"
MANAGER_IP_FILE="/root/swarm_manager_ip.txt"
NFS_SERVER="10.0.60.80"
NFS_BASE="/mnt/docker-swarm/volumes"
DISK_STORAGE="local-zfs"

# --- Flags / Parameters ---
VM_NAME=""
VLAN=$DEFAULT_VLAN
VOLUMES=()         # format name:size
NFS_SHARES=()
FIRST_MANAGER=false
AUTOJOIN=false
ROLE="worker"
USER_NAME="homelab"
AUTOSTART=true

usage() {
  echo "Usage: $0 --name <vmname> [--vlan <tag>] [--volume <name:size>] [--nfs <share1,share2>] [--first-manager] [--autojoin --role worker|manager] [--user <username>]"
  exit 1
}

# --- Parse Args ---
while [[ $# -gt 0 ]]; do
  case "$1" in
    --name) VM_NAME="$2"; shift 2 ;;
    --vlan) VLAN="$2"; shift 2 ;;
    --volume) VOLUMES+=("$2"); shift 2 ;;
    --nfs) IFS=',' read -ra NFS_SHARES <<< "$2"; shift 2 ;;
    --first-manager) FIRST_MANAGER=true; shift ;;
    --autojoin) AUTOJOIN=true; shift ;;
    --role) ROLE="$2"; shift 2 ;;
    --user) USER_NAME="$2"; shift 2 ;;
    *) echo "Unknown arg: $1"; usage ;;
  esac
done

[[ -z "$VM_NAME" ]] && usage

# --- Allocate VMID ---
VMID=$((VMID_BASE + RANDOM % 5000))
echo ">>> Creating VM $VM_NAME ($VMID) on VLAN $VLAN..."

# --- Clone Template ---
qm clone $TEMPLATE_ID $VMID --name $VM_NAME --full
qm set $VMID --net0 virtio,bridge=$BRIDGE_NAME,tag=$VLAN
qm set $VMID --sshkey $SSH_PUBLIC_KEY_PATH
qm set $VMID --ciuser $USER_NAME

# --- Add ZFS Volumes ---
for vol in "${VOLUMES[@]}"; do
  NAME=$(echo $vol | cut -d: -f1)
  SIZE=$(echo $vol | cut -d: -f2)
  echo ">>> Adding ZFS volume $NAME ($SIZE)..."
  qm set $VMID --scsi$(ls /dev/zvol/$DISK_STORAGE | wc -l) $DISK_STORAGE:$SIZE,discard=on
done

# --- Start VM ---
if $AUTOSTART; then
  qm start $VMID
  echo ">>> Waiting 20s for VM to boot..."
  sleep 20
  VM_IP=$(qm guest exec "$VMID" ip -o -4 addr show eth0 | awk '{print $4}' | cut -d/ -f1)
  [[ -z "$VM_IP" ]] && { echo "[ERROR] Could not detect VM IP"; exit 1; }
  echo ">>> VM IP detected: $VM_IP"
fi

# --- Setup cloud-init commands ---
CLOUD_CMD="apt-get update && apt-get install -y docker.io nfs-common zfsutils-linux && systemctl enable docker"

# --- ZFS Mounts inside VM ---
for vol in "${VOLUMES[@]}"; do
  NAME=$(echo $vol | cut -d: -f1)
  CLOUD_CMD+=" && mkdir -p /mnt/$NAME"
  CLOUD_CMD+=" && mkfs.ext4 /dev/disk/by-label/$NAME || true"
  CLOUD_CMD+=" && mount /dev/disk/by-label/$NAME /mnt/$NAME"
done

# --- NFS Mounts inside VM ---
for share in "${NFS_SHARES[@]}"; do
  CLOUD_CMD+=" && mkdir -p /mnt/media/$share"
  CLOUD_CMD+=" && echo '$NFS_SERVER:$NFS_BASE/$share /mnt/media/$share nfs defaults 0 0' >> /etc/fstab"
  CLOUD_CMD+=" && mount -a"
done

# --- Swarm Setup ---
if $FIRST_MANAGER; then
  CLOUD_CMD+=" && docker swarm init --advertise-addr $VM_IP"
  CLOUD_CMD+=" && docker swarm join-token worker -q > /root/worker_token"
  CLOUD_CMD+=" && docker swarm join-token manager -q > /root/manager_token"
fi

# --- Execute cloud-init commands ---
ssh -o StrictHostKeyChecking=no root@"$VM_IP" "$CLOUD_CMD"

# --- Autojoin if needed ---
if $AUTOJOIN && ! $FIRST_MANAGER; then
  MANAGER_IP=$(cat $MANAGER_IP_FILE)
  if [[ "$ROLE" == "worker" ]]; then
    TOKEN=$(cat $SWARM_WORKER_TOKEN_FILE)
    ssh root@"$VM_IP" "docker swarm join --token $TOKEN $MANAGER_IP:2377"
  else
    TOKEN=$(cat $SWARM_MANAGER_TOKEN_FILE)
    ssh root@"$VM_IP" "docker swarm join --token $TOKEN $MANAGER_IP:2377"
  fi
fi

# --- Retrieve tokens back to Proxmox for first manager ---
if $FIRST_MANAGER; then
  echo ">>> Waiting 30s for first manager to finish setup..."
  sleep 30
  echo ">>> Retrieving swarm tokens from $VM_NAME..."
  scp -o StrictHostKeyChecking=no root@"$VM_IP":/root/worker_token $SWARM_WORKER_TOKEN_FILE
  scp -o StrictHostKeyChecking=no root@"$VM_IP":/root/manager_token $SWARM_MANAGER_TOKEN_FILE
  echo ">>> Tokens saved on Proxmox:"
  echo "Worker token: $SWARM_WORKER_TOKEN_FILE"
  echo "Manager token: $SWARM_MANAGER_TOKEN_FILE"
  echo "$VM_IP" > $MANAGER_IP_FILE
fi

echo ">>> Provisioning complete for $VM_NAME ($VMID)"
