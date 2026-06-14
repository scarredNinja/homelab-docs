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
# Version: v10.2
# ==============================================================================
#
# Features:
# - VM creation from template
# - VLAN configuration with random MAC for static DHCP
# - SSH key injection
# - Non-root user creation with optional plaintext password
# - Hostname set to VM name
# - Dynamic ZFS volume creation
# - NFS share mounts
# - Docker Swarm bootstrap (first manager) and token saving
# - Worker/manager autojoin
# - Optional IP fallback if guest agent not installed
# ==============================================================================

# --- Config ---
TEMPLATE_ID=104
VMID_BASE=200
BRIDGE_NAME="vmbr0"
DEFAULT_VLAN=85
SSH_PUBLIC_KEY_PATH="/root/.ssh/id_rsa.pub"
SWARM_WORKER_TOKEN_FILE="/root/swarm_worker_token.txt"
SWARM_MANAGER_TOKEN_FILE="/root/swarm_manager_token.txt"
MANAGER_IP_FILE="/root/swarm_manager_ip.txt"
NFS_SERVER="10.0.60.80"
NFS_BASE="/mnt/docker-swarm/volumes"
DISK_STORAGE="local-zfs"  # This is the Proxmox storage name
ZFS_DISK="/dev/sdb"       # This is only needed for initial pool creation

# --- Flags / Parameters ---
VM_NAME=""
VLAN=$DEFAULT_VLAN
VOLUMES=()
NFS_SHARES=()
FIRST_MANAGER=false
AUTOJOIN=false
ROLE="worker"
USER_NAME="homelab"
USER_PASSWORD=""
AUTOSTART=true

usage() {
  echo "Usage: $0 --name <vmname> [--vlan <tag>] [--volume <name:size>] [--nfs <share1,share2>] [--first-manager] [--autojoin --role worker|manager] [--user <username> --password <plaintext>]"
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
    --password) USER_PASSWORD="$2"; shift 2 ;;
    *) echo "Unknown arg: $1"; usage ;;
  esac
done

[[ -z "$VM_NAME" ]] && usage

# --- Hash password if provided ---
if [[ -n "$USER_PASSWORD" ]]; then
  HASHED_PASS=$(openssl passwd -6 "$USER_PASSWORD")
else
  HASHED_PASS=""
fi

# --- Allocate VMID ---
# Find the next free VMID starting from the base
VMID=$(pvesh get /cluster/nextid)
echo ">>> Creating VM $VM_NAME ($VMID) on VLAN $VLAN..."

# --- Generate Random MAC Address ---
generate_mac() {
  hexchars="0123456789ABCDEF"
  echo "52:54:00$(for i in {1..3}; do echo -n :${hexchars:$((RANDOM%16)):1}${hexchars:$((RANDOM%16)):1}; done)"
}
MAC_ADDR=$(generate_mac)
echo ">>> Generated MAC address for $VM_NAME: $MAC_ADDR"

# --- Clone Template ---
qm clone $TEMPLATE_ID $VMID --name $VM_NAME --full
qm set $VMID --net0 virtio,bridge=$BRIDGE_NAME,tag=$VLAN,macaddr=$MAC_ADDR
qm set $VMID --sshkey $SSH_PUBLIC_KEY_PATH

# --- Add ZFS Volumes dynamically using Proxmox storage alias ---
SCSI_COUNTER=1
for vol in "${VOLUMES[@]}"; do
    NAME=$(echo "$vol" | cut -d: -f1)
    SIZE_WITH_UNIT=$(echo "$vol" | cut -d: -f2)
    SIZE=$(echo "$SIZE_WITH_UNIT" | tr -d 'G') # Remove the 'G'
    echo ">>> Attaching volume $NAME (${SIZE_WITH_UNIT}) from $DISK_STORAGE..."
    qm set $VMID --scsi${SCSI_COUNTER} "${DISK_STORAGE}:${SIZE}G,discard=on"
    SCSI_COUNTER=$((SCSI_COUNTER + 1))
done

# --- Start VM ---
if $AUTOSTART; then
  qm start $VMID
  echo ">>> Waiting 20s for VM to boot..."
  sleep 20

  # --- Attempt guest-agent IP detection ---
  VM_IP=$(qm guest exec $VMID -- ip -4 -o addr show eth0 | awk '{print $4}' | cut -d/ -f1 2>/dev/null)
  if [[ -z "$VM_IP" ]]; then
      echo "[WARNING] Could not detect VM IP via guest-agent."
      read -p "Enter VM IP for $VM_NAME (for provisioning and swarm join): " VM_IP
  fi
  echo ">>> VM IP: $VM_IP"
fi

# --- Setup Cloud-init Commands ---
# Prefix all commands requiring root with 'sudo'
CLOUD_CMD="sudo hostnamectl set-hostname $VM_NAME && sudo apt-get update && sudo apt-get install -y docker.io nfs-common zfsutils-linux qemu-guest-agent"

# --- Create Non-root User ---
CLOUD_CMD+=" && sudo id -u $USER_NAME &>/dev/null || sudo useradd -m -s /bin/bash $USER_NAME"
if [[ -n "$HASHED_PASS" ]]; then
  CLOUD_CMD+=" && echo '$USER_NAME:$HASHED_PASS' | sudo chpasswd -e"
fi
CLOUD_CMD+=" && sudo usermod -aG sudo $USER_NAME"

# --- ZFS Mounts inside VM ---
DISK_COUNTER=1
for vol in "${VOLUMES[@]}"; do
    NAME=$(echo $vol | cut -d: -f1)
    # The disk will be named by the LVM or ZFS storage system, so use a simple, predictable name.
    # We will use the disk's by-id path for reliability.
    CLOUD_CMD+=" && sudo mkdir -p /mnt/$NAME"
    CLOUD_CMD+=" && sudo mkfs.ext4 /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_qm-${VMID}-disk-${DISK_COUNTER} || true"
    CLOUD_CMD+=" && sudo mount /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_qm-${VMID}-disk-${DISK_COUNTER} /mnt/$NAME"
    DISK_COUNTER=$((DISK_COUNTER+1))
done

# --- NFS Mounts inside VM ---
for share in "${NFS_SHARES[@]}"; do
    CLOUD_CMD+=" && sudo mkdir -p /mnt/media/$share"
    CLOUD_CMD+=" && sudo echo '$NFS_SERVER:$NFS_BASE/$share /mnt/media/$share nfs defaults 0 0' | sudo tee -a /etc/fstab"
    CLOUD_CMD+=" && sudo mount -a"
done

# --- Swarm Setup ---
if $FIRST_MANAGER; then
    CLOUD_CMD+=" && sudo docker swarm init --advertise-addr $VM_IP"
    CLOUD_CMD+=" && sudo docker swarm join-token worker -q > /root/worker_token"
    CLOUD_CMD+=" && sudo docker swarm join-token manager -q > /root/manager_token"
fi

# --- Execute Cloud-init Commands ---
ssh -o StrictHostKeyChecking=no root@"$VM_IP" "$CLOUD_CMD"

# --- Autojoin if needed ---
if $AUTOJOIN && ! $FIRST_MANAGER; then
    MANAGER_IP=$(cat $MANAGER_IP_FILE)
    if [[ "$ROLE" == "worker" ]]; then
        TOKEN=$(cat $SWARM_WORKER_TOKEN_FILE)
        ssh root@"$VM_IP" "sudo docker swarm join --token $TOKEN $MANAGER_IP:2377"
    else
        TOKEN=$(cat $SWARM_MANAGER_TOKEN_FILE)
        ssh root@"$VM_IP" "sudo docker swarm join --token $TOKEN $MANAGER_IP:2377"
    fi
fi

# --- Retrieve tokens back to Proxmox for first manager ---
if $FIRST_MANAGER; then
    echo ">>> Waiting 30s for first manager to finish setup..."
    sleep 30
    echo ">>> Retrieving swarm tokens from $VM_NAME..."
    scp -o StrictHostKeyChecking=no root@"$VM_IP":/root/worker_token $SWARM_WORKER_TOKEN_FILE
    scp -o StrictHostKeyChecking=no root@"$VM_IP":/root/manager_token $SWARM_MANAGER_TOKEN_FILE
    echo "$VM_IP" > $MANAGER_IP_FILE
    echo ">>> Tokens saved on Proxmox:"
    echo "Worker token: $SWARM_WORKER_TOKEN_FILE"
    echo "Manager token: $SWARM_MANAGER_TOKEN_FILE"
fi

echo ">>> Provisioning complete for $VM_NAME ($VMID)"
echo ">>> MAC address for static DHCP in pfSense: $MAC_ADDR"
