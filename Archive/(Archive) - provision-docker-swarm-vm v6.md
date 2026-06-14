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
# Version: v9
# ==============================================================================
#
# Supports:
#   - Provisioning new VMs from template
#   - Bootstrapping first swarm manager (--first-manager)
#   - Autojoining workers or managers (--autojoin --role worker|manager)
#   - Creating extra ZFS volumes (--volume <name:size>)
#   - Mounting Synology NFS shares (--nfs <share1,share2,...>)
#   - Displaying saved swarm tokens (--show-tokens)
#
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
NFS_BASE_PATH="/mnt/docker-swarm/volumes"

# --- Flags ---
AUTOSTART=false
AUTOJOIN=false
FIRST_MANAGER=false
ROLE="worker"   # default role
VM_NAME=""
VLAN=$DEFAULT_VLAN
SHOW_TOKENS=false
VOLUMES=()  # format name:size
NFS_SHARES=() # list of shares

# --- Parse Args ---
while [[ $# -gt 0 ]]; do
  case $1 in
    --name) VM_NAME="$2"; shift ;;
    --vlan) VLAN="$2"; shift ;;
    --template-id) TEMPLATE_ID="$2"; shift ;;
    --autojoin) AUTOJOIN=true ;;
    --role) ROLE="$2"; shift ;;   # worker | manager
    --first-manager) FIRST_MANAGER=true ;;
    --start) AUTOSTART=true ;;
    --volume) VOLUMES+=("$2"); shift ;; # format: name:size
    --nfs) IFS=',' read -r -a NFS_SHARES <<< "$2"; shift ;;
    --show-tokens) SHOW_TOKENS=true ;;
    *) echo "[ERROR] Unknown option: $1"; exit 1 ;;
  esac
  shift
done

if [[ -z "$VM_NAME" ]] && ! $SHOW_TOKENS; then
  echo "[ERROR] VM name required (--name <vmname>)"
  exit 1
fi

# --- Show Tokens Only ---
if $SHOW_TOKENS; then
  echo "=== Swarm Tokens & Manager Info ==="
  [[ -f "$SWARM_WORKER_TOKEN_FILE" ]] && echo "Worker Token : $(cat $SWARM_WORKER_TOKEN_FILE)" || echo "Worker Token : [Not Found]"
  [[ -f "$SWARM_MANAGER_TOKEN_FILE" ]] && echo "Manager Token: $(cat $SWARM_MANAGER_TOKEN_FILE)" || echo "Manager Token: [Not Found]"
  [[ -f "$MANAGER_IP_FILE" ]] && echo "Manager IP   : $(cat $MANAGER_IP_FILE)" || echo "Manager IP   : [Not Found]"
  exit 0
fi

# --- Create VM ---
VMID=$((VMID_BASE + RANDOM % 5000))
echo "[INFO] Creating VM $VM_NAME ($VMID) from template $TEMPLATE_ID"

qm clone "$TEMPLATE_ID" "$VMID" --name "$VM_NAME" --full
qm set "$VMID" --net0 virtio,bridge=$BRIDGE_NAME,tag=$VLAN
qm set "$VMID" --sshkey "$SSH_PUBLIC_KEY_PATH"

# Extra ZFS Volumes
for vol in "${VOLUMES[@]}"; do
  NAME=$(echo "$vol" | cut -d: -f1)
  SIZE=$(echo "$vol" | cut -d: -f2)
  echo "[INFO] Adding ZFS volume $NAME ($SIZE) to $VM_NAME"
  qm set "$VMID" --scsi$((RANDOM % 10 + 1)) local-zfs:"$SIZE"
done

if $AUTOSTART; then
  qm start "$VMID"
fi

echo "[INFO] VM created. Remember to assign static IP before starting if needed."

# --- Wait for IP (if autostarted) ---
if $AUTOSTART; then
  echo "[INFO] Waiting for VM to acquire IP..."
  sleep 20
  VM_IP=$(qm guest exec "$VMID" ip -o -4 addr show eth0 | awk '{print $4}' | cut -d/ -f1)
  if [[ -z "$VM_IP" ]]; then
    echo "[ERROR] Unable to detect VM IP"
    exit 1
  fi
  echo "[INFO] VM IP detected: $VM_IP"
fi

# --- Swarm Handling ---
if $FIRST_MANAGER; then
  echo "[INFO] Initializing Docker Swarm on first manager..."
  ssh -o StrictHostKeyChecking=no root@"$VM_IP" "docker swarm init --advertise-addr $VM_IP"

  WORKER_TOKEN=$(ssh root@"$VM_IP" "docker swarm join-token -q worker")
  MANAGER_TOKEN=$(ssh root@"$VM_IP" "docker swarm join-token -q manager")

  echo "$WORKER_TOKEN" > "$SWARM_WORKER_TOKEN_FILE"
  echo "$MANAGER_TOKEN" > "$SWARM_MANAGER_TOKEN_FILE"
  echo "$VM_IP" > "$MANAGER_IP_FILE"

  echo "[INFO] Swarm initialized. Tokens saved."
fi

if $AUTOJOIN && ! $FIRST_MANAGER; then
  MANAGER_IP=$(cat "$MANAGER_IP_FILE")
  if [[ "$ROLE" == "worker" ]]; then
    TOKEN=$(cat "$SWARM_WORKER_TOKEN_FILE")
    echo "[INFO] Joining $VM_NAME to swarm as WORKER..."
    ssh root@"$VM_IP" "docker swarm join --token $TOKEN $MANAGER_IP:2377"
  elif [[ "$ROLE" == "manager" ]]; then
    TOKEN=$(cat "$SWARM_MANAGER_TOKEN_FILE")
    echo "[INFO] Joining $VM_NAME to swarm as MANAGER..."
    ssh root@"$VM_IP" "docker swarm join --token $TOKEN $MANAGER_IP:2377"
  fi
fi

# --- NFS Mounts ---
for share in "${NFS_SHARES[@]}"; do
  echo "[INFO] Mounting NFS share $share on $VM_NAME"
  ssh root@"$VM_IP" "mkdir -p /mnt/media/$share && echo '$NFS_SERVER:$NFS_BASE_PATH/$share /mnt/media/$share nfs defaults 0 0' >> /etc/fstab && mount -a"
done
