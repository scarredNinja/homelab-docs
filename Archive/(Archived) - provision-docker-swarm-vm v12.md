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
# Version: v12
# ==============================================================================
#
# Features:
# - VM creation from template
# - VLAN configuration with random MAC for static DHCP
# - SSH key injection
# - Non-root user creation with optional plaintext password
# - Hostname set to VM name
# - Dynamic ZFS volume creation and attachment
# - NFS share mounts
# - Docker Swarm bootstrap (first manager) and token saving
# - Worker/manager autojoin
# - Enhanced cleanup on failure (VM + volumes)
# - Token output display
# - Extended boot timeout option
# - Optional IP fallback if guest agent not installed
# ==============================================================================

set -euo pipefail

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
ZFS_POOL="rpool/data"     # Default ZFS pool path
DEFAULT_BOOT_TIMEOUT=45   # Extended default boot timeout

# --- Flags / Parameters ---
VM_NAME=""
VLAN=$DEFAULT_VLAN
VOLUMES=()        # format: name:sizeG  (e.g. data:50G)
NFS_SHARES=()     # array of share names
ZFS_DISK=""       # optional, not currently used but kept for future
FIRST_MANAGER=false
AUTOJOIN=false
ROLE="worker"
USER_NAME="homelab"
USER_PASSWORD=""
AUTOSTART=true
BOOT_TIMEOUT=$DEFAULT_BOOT_TIMEOUT
CLEANUP_ON_FAILURE=true
VMID=""           # Will be set during execution

usage() {
  echo "Usage: $0 --name <vmname> [--vlan <tag>] [--volume <name:sizeG>] [--nfs <share1,share2>] [--zfs-pool <pool>] [--first-manager] [--autojoin --role worker|manager] [--user <username> --password <plaintext>] [--no-autostart] [--boot-timeout <seconds>] [--no-cleanup]"
  echo ""
  echo "Options:"
  echo "  --name <vmname>           VM name (required)"
  echo "  --vlan <tag>              VLAN tag (default: $DEFAULT_VLAN)"
  echo "  --volume <name:sizeG>     Add ZFS volume (can be used multiple times)"
  echo "  --nfs <share1,share2>     NFS shares to mount (comma-separated)"
  echo "  --zfs-pool <pool>         ZFS pool path (default: $ZFS_POOL)"
  echo "  --first-manager           Initialize as first Docker Swarm manager"
  echo "  --autojoin                Auto-join existing swarm"
  echo "  --role <worker|manager>   Role for autojoin (default: worker)"
  echo "  --user <username>         Non-root user to create (default: $USER_NAME)"
  echo "  --password <plaintext>    Password for user"
  echo "  --no-autostart           Don't start VM after creation"
  echo "  --boot-timeout <seconds>  Boot wait timeout (default: $DEFAULT_BOOT_TIMEOUT)"
  echo "  --no-cleanup             Don't cleanup on failure"
  exit 1
}

# --- Parse Args ---
while [[ $# -gt 0 ]]; do
  case "$1" in
    --name) VM_NAME="$2"; shift 2 ;;
    --vlan) VLAN="$2"; shift 2 ;;
    --volume) VOLUMES+=("$2"); shift 2 ;;
    --nfs) IFS=',' read -ra NFS_SHARES <<< "$2"; shift 2 ;;
    --zfs-pool) ZFS_POOL="$2"; shift 2 ;;
    --zfs-disk) ZFS_DISK="$2"; shift 2 ;;
    --first-manager) FIRST_MANAGER=true; shift ;;
    --autojoin) AUTOJOIN=true; shift ;;
    --role) ROLE="$2"; shift 2 ;;
    --user) USER_NAME="$2"; shift 2 ;;
    --password) USER_PASSWORD="$2"; shift 2 ;;
    --no-autostart) AUTOSTART=false; shift ;;
    --boot-timeout) BOOT_TIMEOUT="$2"; shift 2 ;;
    --no-cleanup) CLEANUP_ON_FAILURE=false; shift ;;
    *) echo "Unknown arg: $1"; usage ;;
  esac
done

[[ -z "$VM_NAME" ]] && usage

# --- Enhanced cleanup function ---
cleanup_on_failure() {
    echo ">>> [ERROR] Script failed. Starting cleanup process..."
    
    if [[ "$CLEANUP_ON_FAILURE" == true ]]; then
        if [[ -n "$VMID" ]]; then
            echo ">>> Stopping and destroying VM $VMID..."
            # Stop VM if running
            qm stop "$VMID" --timeout 10 2>/dev/null || true
            sleep 2
            # Destroy VM completely
            qm destroy "$VMID" --destroy-unreferenced-disks --purge 2>/dev/null || true
        fi
        
        # Clean up ZFS volumes
        if [[ ${#VOLUMES[@]} -gt 0 ]]; then
            echo ">>> Cleaning up created ZFS volumes..."
            for vol in "${VOLUMES[@]}"; do
                NAME=$(echo "$vol" | cut -d: -f1)
                ZVOL="${ZFS_POOL}/${VM_NAME}_${NAME}"
                echo ">>> Destroying ZFS volume: ${ZVOL}"
                zfs destroy -r "${ZVOL}" 2>/dev/null || true
            done
        fi
        
        echo ">>> Cleanup completed."
    else
        echo ">>> Cleanup disabled. VM $VMID and volumes preserved for manual inspection."
    fi
    
    exit 1
}

# Set up enhanced error handling
trap 'cleanup_on_failure' ERR

# --- Hash password if provided (for chpasswd -e) ---
if [[ -n "${USER_PASSWORD}" ]]; then
  # Use SHA512
  HASHED_PASS=$(openssl passwd -6 "$USER_PASSWORD")
else
  HASHED_PASS=""
fi

# --- Allocate VMID ---
VMID="$(pvesh get /cluster/nextid)"
if [[ -z "$VMID" ]]; then
  echo "ERROR: could not get next VMID from Proxmox"
  exit 1
fi
echo ">>> Creating VM $VM_NAME ($VMID) on VLAN $VLAN..."
echo ">>> Boot timeout: ${BOOT_TIMEOUT}s"

# --- Generate Random MAC Address (QEMU OUI 52:54:00 commonly used) ---
generate_mac() {
  hexchars="0123456789abcdef"
  mac="52:54:00"
  for i in {1..3}; do
    a=${hexchars:$((RANDOM%16)):1}
    b=${hexchars:$((RANDOM%16)):1}
    mac="${mac}:$a$b"
  done
  echo "$mac"
}
MAC_ADDR=$(generate_mac)
echo ">>> Generated MAC address for $VM_NAME: $MAC_ADDR"

# --- Clone Template ---
qm clone "$TEMPLATE_ID" "$VMID" --name "$VM_NAME" --full || { echo "ERROR: qm clone failed"; exit 1; }

# Set NIC and MAC and VLAN
qm set "$VMID" --net0 "virtio,bridge=${BRIDGE_NAME},tag=${VLAN},macaddr=${MAC_ADDR}"

# ==============================================================================
# --- Automated ZFS Volumes + Guest-Agent + Auto-IP + Cloud-Init ---
# ==============================================================================
# Requires: $VMID, $VM_NAME, $VOLUMES, $AUTOSTART, $BOOT_TIMEOUT, $ZFS_POOL
# ==============================================================================

VM_IP=""  # Initialize VM_IP

# --- ZFS Data Volumes Creation & Attachment ---
DISK_COUNTER=1  # scsi0 reserved for boot disk
for VOL_ARG in "${VOLUMES[@]}"; do
    VOLNAME="${VM_NAME}_$(echo "$VOL_ARG" | cut -d: -f1)"
    SIZE="$(echo "$VOL_ARG" | cut -d: -f2)"
    ZVOL_PATH="${ZFS_POOL}/${VOLNAME}"

    echo ">>> Processing volume $VOLNAME ($SIZE)"

    # Create ZFS volume if missing
    if zfs list "$ZVOL_PATH" &>/dev/null; then
        echo ">>> ZFS volume $ZVOL_PATH already exists, skipping creation"
    else
        echo ">>> Creating ZFS volume: $ZVOL_PATH"
        zfs create -V "$SIZE" "$ZVOL_PATH"
    fi

    # Attach to next free SCSI ID
    SCSI_ID=1
    while qm config "$VMID" | grep -q "^scsi$SCSI_ID:"; do
        SCSI_ID=$((SCSI_ID+1))
    done
    echo ">>> Attaching $ZVOL_PATH as scsi$SCSI_ID"
    qm set "$VMID" -scsi${SCSI_ID} /dev/zvol/$ZVOL_PATH,discard=on
done

# After starting the VM and waiting initial boot time
if $AUTOSTART; then
  qm start "$VMID" || { echo "ERROR: failed to start VM $VMID"; exit 1; }
  echo ">>> Waiting ${BOOT_TIMEOUT}s for VM to boot..."
  sleep "$BOOT_TIMEOUT"

  echo ">>> Attempting to detect VM IP via guest-agent..."
  for i in {1..24}; do
    # Use correct variable names in jq
    VM_IP=$(qm agent $VMID network-get-interfaces \
        | jq -r --arg MAC "$MAC_ADDR" '
             .[] |
             select(.["hardware-address"] == $MAC) |
             .["ip-addresses"][] |
             select(.["ip-address-type"] == "ipv4") |
             .["ip-address"]
        ' 2>/dev/null || true)

    if [[ -n "$VM_IP" ]]; then
      echo ">>> Detected VM IP: $VM_IP"
      break
    fi
    echo ">>> Waiting for guest-agent to report IP... ($i/24)"
    sleep 10
  done

  if [[ -z "$VM_IP" ]]; then
    echo ">>> ERROR: Could not detect VM IP via guest-agent after 4 minutes."
    echo "Please check that qemu-guest-agent is installed and VM network is up."
    exit 1
  fi
fi

# --- Prepare cloud-init commands for volume mounting and guest-agent ---
CLOUD_BOOT_CMDS=$(cat <<EOF
set -e
echo ">>> Starting VM provisioning for ${VM_NAME}..."

# --- Set hostname ---
sudo hostnamectl set-hostname ${VM_NAME}

export DEBIAN_FRONTEND=noninteractive
echo ">>> Installing required packages..."
sudo apt-get update -y
sudo apt-get install -y docker.io nfs-common zfsutils-linux qemu-guest-agent

# --- Create user if not exists ---
id -u ${USER_NAME} &>/dev/null || sudo useradd -m -s /bin/bash ${USER_NAME}
EOF
)

# Add password if provided
if [[ -n "${HASHED_PASS}" ]]; then
  CLOUD_BOOT_CMDS+="
echo '${USER_NAME}:${HASHED_PASS}' |  sudo chpasswd -e
echo \">>> Set password for user ${USER_NAME}\"
"
fi

CLOUD_BOOT_CMDS+="
sudo usermod -aG sudo,docker ${USER_NAME} || true
echo \">>> Added ${USER_NAME} to sudo and docker groups\"
"

# --- Mount attached volumes inside VM ---
DISK_COUNTER=1
for VOL_ARG in "${VOLUMES[@]}"; do
    NAME=$(echo "$VOL_ARG" | cut -d: -f1)
    CLOUD_BOOT_CMDS+="
sudo mkdir -p /mnt/${NAME} || true
DEV=\$(ls /dev/disk/by-id/ | grep -E 'virtio.*scsi$DISK_COUNTER' | head -n1 || true)
if [[ -z \"\$DEV\" ]]; then
    DEV=\$(lsblk -ndo KNAME,TYPE | awk '\$2==\"disk\"{print \$1}' | sort | tail -n+2 | head -n1)
fi
if [[ -n \"\$DEV\" && -b /dev/\$DEV ]]; then
    if ! blkid /dev/\$DEV >/dev/null 2>&1; then
        mkfs.ext4 -F /dev/\$DEV || true
    fi
    sudo mount /dev/\$DEV /mnt/${NAME} || true
    echo '>>> Mounted /dev/\$DEV to /mnt/${NAME}'
else
    echo '>>> WARNING: Could not find device for volume ${NAME}'
fi
"
    DISK_COUNTER=$((DISK_COUNTER+1))
done

# --- Enable and start QEMU guest-agent ---
CLOUD_BOOT_CMDS+="
echo '>>> Ensuring QEMU guest-agent is enabled and running...'
if systemctl list-unit-files | grep -qw qemu-guest-agent.service; then
    sudo systemctl enable --now qemu-guest-agent.service || true
elif [ -f /lib/systemd/system/qemu-guest-agent.socket ]; then
    sudo systemctl enable --now qemu-guest-agent.socket || true
else
    echo '>>> WARNING: QEMU guest agent unit not found; skipping enable/start'
fi
"

# --- Execute cloud-init commands on the VM ---
echo ">>> Executing bootstrap commands on ${VM_IP}..."
ssh -o StrictHostKeyChecking=no -o ConnectTimeout=15 sussh@"${VM_IP}" "bash -s" <<EOF
${CLOUD_BOOT_CMDS}
EOF


# Swarm init for first manager
if [[ "${FIRST_MANAGER}" == true ]]; then
  CLOUD_BOOT_CMDS+="
echo \">>> Initializing Docker Swarm as first manager...\"
sudo docker swarm init --advertise-addr ${VM_IP} || true
sudo docker swarm join-token worker -q > /root/worker_token || true
sudo docker swarm join-token manager -q > /root/manager_token || true
echo \">>> Docker Swarm initialization completed\"
"
fi

CLOUD_BOOT_CMDS+="
echo \">>> VM provisioning completed successfully\"
"

# Run the cloud commands over SSH (root)
echo ">>> Executing bootstrap commands on ${VM_IP}..."
ssh -o StrictHostKeyChecking=no -o ConnectTimeout=15 sussh@"${VM_IP}" "bash -s" <<EOF
${CLOUD_BOOT_CMDS}
EOF

# --- Autojoin if needed (and not the first manager) ---
if [[ "${AUTOJOIN}" == true && "${FIRST_MANAGER}" != true ]]; then
    if [[ ! -f "${MANAGER_IP_FILE}" || ! -f "${SWARM_WORKER_TOKEN_FILE}" ]]; then
        echo "ERROR: manager info or tokens not present on Proxmox (${MANAGER_IP_FILE}, ${SWARM_WORKER_TOKEN_FILE})"
        exit 1
    fi
    MANAGER_IP=$(cat "${MANAGER_IP_FILE}")
    if [[ "${ROLE}" == "worker" ]]; then
        TOKEN=$(cat "${SWARM_WORKER_TOKEN_FILE}")
    else
        if [[ ! -f "${SWARM_MANAGER_TOKEN_FILE}" ]]; then
            echo "ERROR: manager token not found at ${SWARM_MANAGER_TOKEN_FILE}"
            exit 1
        fi
        TOKEN=$(cat "${SWARM_MANAGER_TOKEN_FILE}")
    fi
    echo ">>> Joining swarm at ${MANAGER_IP}:2377 as ${ROLE}"
    
    # Test connectivity to manager first
    if ! ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 sussh@"${VM_IP}" "nc -zv ${MANAGER_IP} 2377" >/dev/null 2>&1; then
        echo "WARNING: Cannot reach swarm manager at ${MANAGER_IP}:2377"
        echo "Please ensure:"
        echo "  1. Manager node is running and accessible"
        echo "  2. Docker swarm is initialized on manager"
        echo "  3. Network connectivity between nodes"
    fi
    
    ssh -o StrictHostKeyChecking=no -o ConnectTimeout=15 sussh@"${VM_IP}" "docker swarm sudo join --token ${TOKEN} ${MANAGER_IP}:2377" || {
        echo "ERROR: Swarm join failed. Checking swarm status..."
        ssh -o StrictHostKeyChecking=no sussh@"${VM_IP}" "docker info | grep -A5 Swarm" || true
        echo "Please manually join the swarm later with:"
        echo "  docker swarm join --token ${TOKEN} ${MANAGER_IP}:2377"
    }
fi

# --- Retrieve tokens back to Proxmox for first manager ---
if [[ "${FIRST_MANAGER}" == true ]]; then
    echo ">>> Waiting 15s for swarm initialization to complete..."
    sleep 15
    echo ">>> Retrieving swarm tokens from ${VM_NAME} (${VM_IP})..."
     ssh sussh@"${VM_IP}" 'sudo docker swarm join-token worker -q' > "${SWARM_WORKER_TOKEN_FILE}"
     "${SWARM_WORKER_TOKEN_FILE}" || echo "WARNING: couldn't copy worker token"
     ssh sussh@"${VM_IP}" 'sudo docker swarm join-token manager -q' > "${SWARM_MANAGER_TOKEN_FILE}"
      "${SWARM_MANAGER_TOKEN_FILE}" || echo "WARNING: couldn't copy manager token"
    echo "${VM_IP}" > "${MANAGER_IP_FILE}"
    
    # Display tokens
    echo ""
    echo "===================================================================="
    echo "DOCKER SWARM TOKENS"
    echo "===================================================================="
    if [[ -f "${SWARM_WORKER_TOKEN_FILE}" ]]; then
        echo "Worker Token:"
        cat "${SWARM_WORKER_TOKEN_FILE}"
        echo ""
    fi
    if [[ -f "${SWARM_MANAGER_TOKEN_FILE}" ]]; then
        echo "Manager Token:"
        cat "${SWARM_MANAGER_TOKEN_FILE}"
        echo ""
    fi
    echo "Manager IP: ${VM_IP}"
    echo "===================================================================="
    echo ""
    echo "Token files saved on Proxmox:"
    echo "  Worker token:  ${SWARM_WORKER_TOKEN_FILE}"
    echo "  Manager token: ${SWARM_MANAGER_TOKEN_FILE}"
    echo "  Manager IP:    ${MANAGER_IP_FILE}"
fi

# Disable cleanup trap on successful completion
trap - ERR

echo ""
echo "===================================================================="
echo "PROVISIONING COMPLETED SUCCESSFULLY"
echo "===================================================================="
echo "VM Name:     ${VM_NAME}"
echo "VM ID:       ${VMID}"
echo "IP Address:  ${VM_IP}"
echo "MAC Address: ${MAC_ADDR}"
echo "VLAN:        ${VLAN}"
echo "User:        ${USER_NAME}"
if [[ ${#VOLUMES[@]} -gt 0 ]]; then
    echo "Volumes:     ${VOLUMES[*]}"
fi
if [[ ${#NFS_SHARES[@]} -gt 0 ]]; then
    echo "NFS Shares:  ${NFS_SHARES[*]}"
fi
echo "===================================================================="
echo ""
echo "Next steps:"
echo "1. Configure static DHCP reservation in pfSense using MAC: ${MAC_ADDR}"
if [[ "${FIRST_MANAGER}" == true ]]; then
    echo "2. Use the tokens above to join additional nodes to the swarm"
elif [[ "${AUTOJOIN}" == true ]]; then
    echo "2. Node should now be part of the Docker Swarm as a ${ROLE}"
else
    echo "2. Manually join this node to Docker Swarm if needed"
fi
echo "3. SSH access: ssh ${USER_NAME}@${VM_IP} or ssh sussh@${VM_IP}"ch
