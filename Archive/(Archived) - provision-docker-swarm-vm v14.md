#!/bin/bash
# ==============================================================================
# Docker Swarm VM Provisioning Script (Proxmox)
# Version: v15.2 (FIXED: Line 134 command not found error)
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
ZFS_POOL="rpool/data"      # Default ZFS pool path
DEFAULT_BOOT_TIMEOUT=45     # Extended default boot timeout

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
VMID=""            # Will be set during execution

usage() {
  echo "Usage: $0 --name <vmname> [--vlan <tag>] [--volume <name:sizeG>] [--nfs <share1,share2>] [--zfs-pool <pool>] [--first-manager] [--autojoin --role worker|manager] [--user <username> --password <plaintext>] [--no-autostart] [--boot-timeout <seconds>] [--no-cleanup]"
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
            qm stop "$VMID" --timeout 10 2>/dev/null || true
            sleep 2
            qm destroy "$VMID" --destroy-unreferenced-disks --purge 2>/dev/null || true
        fi
        
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

trap 'cleanup_on_failure' ERR

# --- Validate required variables ---
[[ -z "$USER_NAME" ]] && { echo "ERROR: USER_NAME cannot be empty"; exit 1; }

# --- Hash password if provided ---
HASHED_PASS=""
if [[ -n "${USER_PASSWORD}" ]]; then
  HASHED_PASS=$(openssl passwd -6 "$USER_PASSWORD")
fi

# --- Allocate VMID ---
VMID="$(pvesh get /cluster/nextid)"
echo ">>> Creating VM $VM_NAME ($VMID) on VLAN $VLAN..."
echo ">>> Boot timeout: ${BOOT_TIMEOUT}s"

# --- Generate MAC ---
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
MAC_ADDR="$(generate_mac)"
echo ">>> Generated MAC address for $VM_NAME: $MAC_ADDR"

# --- Clone Template ---
qm clone "$TEMPLATE_ID" "$VMID" --name "$VM_NAME" --full || { echo "ERROR: qm clone failed"; exit 1; }
qm set "$VMID" --net0 "virtio,bridge=${BRIDGE_NAME},tag=${VLAN},macaddr=${MAC_ADDR}"

# --- Configure Cloud-init User/Password/SSH ---
# Set the user and inject the public key
qm set "$VMID" --ciuser "$USER_NAME" --sshkeys "$SSH_PUBLIC_KEY_PATH"

# Set the password if provided, using Proxmox's ci-latest
if [[ -n "${USER_PASSWORD}" ]]; then
  qm set "$VMID" --cipassword "$USER_PASSWORD"
  echo ">>> Set plaintext password for user ${USER_NAME} via cloud-init"
# NOTE: Proxmox/cloud-init handles the hashing/shadow update securely
fi

# --- ZFS Volumes ---
DISK_COUNTER=1
for VOL_ARG in "${VOLUMES[@]}"; do
    VOLNAME="${VM_NAME}_$(echo "$VOL_ARG" | cut -d: -f1)"
    SIZE="$(echo "$VOL_ARG" | cut -d: -f2)"
    ZVOL_PATH="${ZFS_POOL}/${VOLNAME}"
    if ! zfs list "$ZVOL_PATH" &>/dev/null; then
        echo ">>> Creating ZFS volume: $ZVOL_PATH"
        zfs create -V "$SIZE" "$ZVOL_PATH"
    fi
    SCSI_ID=1
    while qm config "$VMID" | grep -q "^scsi$SCSI_ID:"; do
        SCSI_ID=$((SCSI_ID+1))
    done
    qm set "$VMID" -scsi${SCSI_ID} /dev/zvol/$ZVOL_PATH,discard=on
done

# --- Start VM and detect IP ---
VM_IP=""
if $AUTOSTART; then
  qm start "$VMID" || { echo "ERROR: failed to start VM $VMID"; exit 1; }
  echo ">>> Waiting ${BOOT_TIMEOUT}s for VM to boot..."
  sleep "$BOOT_TIMEOUT"
  for i in {1..24}; do
    VM_IP=$(qm agent $VMID network-get-interfaces \
      | jq -r --arg MAC "$MAC_ADDR" '
           .[] | select(.["hardware-address"]==$MAC) |
           .["ip-addresses"][] | select(.["ip-address-type"]=="ipv4") | .["ip-address"]
      ' 2>/dev/null || true)
    if [[ -n "$VM_IP" ]]; then break; fi
    echo ">>> Waiting for guest-agent to report IP... ($i/24)"; sleep 10
  done
  [[ -z "$VM_IP" ]] && { echo ">>> ERROR: Could not detect VM IP"; exit 1; }
fi

# --- Cloud-init commands ---
# Build the command string properly to avoid variable expansion issues
CLOUD_BOOT_CMDS="set -e
echo '>>> Starting VM provisioning for ${VM_NAME}...'
sudo hostnamectl set-hostname ${VM_NAME}
export DEBIAN_FRONTEND=noninteractive
sudo apt-get update -y
sudo apt-get install -y docker.io nfs-common zfsutils-linux qemu-guest-agent parted
sudo systemctl enable --now docker
sleep 5
id -u ${USER_NAME} &>/dev/null || sudo useradd -m -s /bin/bash ${USER_NAME}
sudo usermod -aG sudo,docker ${USER_NAME} || true
echo '>>> Added ${USER_NAME} to sudo and docker groups'"

# --- Volume mounting ---
if [[ ${#VOLUMES[@]} -gt 0 ]]; then
    for VOL_ARG in "${VOLUMES[@]}"; do
        VOLNAME="$(echo "$VOL_ARG" | cut -d: -f1)"
        MOUNT_POINT="/data/${VOLNAME}"
        DEVICE_PATH="/dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi"
        
        CLOUD_BOOT_CMDS+="
echo '>>> Setting up volume: ${VOLNAME}'
sudo mkdir -p ${MOUNT_POINT}
# Find the correct device (assumes devices are added in order)
for dev in \$(ls ${DEVICE_PATH}* 2>/dev/null | sort); do
    if ! grep -q \"\$dev\" /proc/mounts; then
        echo '>>> Formatting and mounting \$dev to ${MOUNT_POINT}'
        sudo mkfs.ext4 -F \$dev
        echo \"\$dev ${MOUNT_POINT} ext4 defaults 0 2\" | sudo tee -a /etc/fstab
        sudo mount ${MOUNT_POINT}
        sudo chown -R ${USER_NAME}:${USER_NAME} ${MOUNT_POINT}
        break
    fi
done"
    done
fi

# --- NFS mounting ---
if [[ ${#NFS_SHARES[@]} -gt 0 ]]; then
    CLOUD_BOOT_CMDS+="
echo '>>> Setting up NFS mounts...'
sudo mkdir -p /mnt/nfs"
    
    for SHARE in "${NFS_SHARES[@]}"; do
        CLOUD_BOOT_CMDS+="
sudo mkdir -p /mnt/nfs/${SHARE}
echo '${NFS_SERVER}:${NFS_BASE}/${SHARE} /mnt/nfs/${SHARE} nfs defaults,_netdev 0 0' | sudo tee -a /etc/fstab"
    done
    
    CLOUD_BOOT_CMDS+="
sudo mount -a
sudo chown -R ${USER_NAME}:${USER_NAME} /mnt/nfs"
fi

# --- QEMU guest-agent ---
CLOUD_BOOT_CMDS+="
sudo systemctl enable --now qemu-guest-agent.service || true"

# --- Swarm init for first manager ---
if [[ "${FIRST_MANAGER}" == true ]]; then
  CLOUD_BOOT_CMDS+="
for i in {1..3}; do
    if sudo docker swarm init --advertise-addr ${VM_IP}; then
        echo '>>> Docker Swarm initialized'
        break
    else
        echo '>>> Swarm init failed, retrying (\$i/3)'
        sleep 5
    fi
done
sudo docker swarm join-token worker -q > /root/worker_token
sudo docker swarm join-token manager -q > /root/manager_token"
fi

echo ">>> VM provisioning commands ready. Executing..."
ssh -o StrictHostKeyChecking=no -o ConnectTimeout=15 sussh@${VM_IP}" "bash -s" <<EOF
${CLOUD_BOOT_CMDS}
EOF

# --- Autojoin ---
if [[ "${AUTOJOIN}" == true && "${FIRST_MANAGER}" != true ]]; then
    if [[ ! -f "${MANAGER_IP_FILE}" ]]; then
        echo "ERROR: Manager IP file not found. Cannot autojoin."
        exit 1
    fi
    
    MANAGER_IP=$(cat "${MANAGER_IP_FILE}")
    TOKEN_FILE=$([[ "${ROLE}" == "worker" ]] && echo "${SWARM_WORKER_TOKEN_FILE}" || echo "${SWARM_MANAGER_TOKEN_FILE}")
    
    if [[ ! -f "${TOKEN_FILE}" ]]; then
        echo "ERROR: Token file ${TOKEN_FILE} not found. Cannot autojoin."
        exit 1
    fi
    
    TOKEN=$(cat "${TOKEN_FILE}")
    for i in {1..3}; do
        if ssh sussh${VM_IP}" "sudo docker swarm join --token ${TOKEN} ${MANAGER_IP}:2377"; then
            echo ">>> Swarm join successful"; break
        else
            echo ">>> Swarm join failed, retrying ($i/3)"; sleep 5
        fi
    done
fi

# --- Retrieve tokens for first manager ---
if [[ "${FIRST_MANAGER}" == true ]]; then
    sleep 15
    ssh sussh${VM_IP}" 'sudo cat /root/worker_token' > "${SWARM_WORKER_TOKEN_FILE}"
    ssh sussh${VM_IP}" 'sudo cat /root/manager_token' > "${SWARM_MANAGER_TOKEN_FILE}"
    echo "${VM_IP}" > "${MANAGER_IP_FILE}"
fi

trap - ERR

# --- Final Output ---
echo "===================================================================="
echo "PROVISIONING COMPLETED SUCCESSFULLY"
echo "VM Name:     ${VM_NAME}"
echo "VM ID:       ${VMID}"
echo "IP Address:  ${VM_IP}"
echo "MAC Address: ${MAC_ADDR}"
echo "VLAN:        ${VLAN}"
echo "User:        ${USER_NAME}"
[[ ${#VOLUMES[@]} -gt 0 ]] && echo "Volumes:     ${VOLUMES[*]}"
[[ ${#NFS_SHARES[@]} -gt 0 ]] && echo "NFS Shares:  ${NFS_SHARES[*]}"
echo "===================================================================="
echo "Next steps:"
echo "1. Configure static DHCP in pfSense using MAC: ${MAC_ADDR}"
if [[ "${FIRST_MANAGER}" == true ]]; then
  echo "2. Use tokens above to join additional nodes to the swarm"
elif [[ "${AUTOJOIN}" == true ]]; then
  echo "2. Node should now be part of Docker Swarm as ${ROLE}"
else
  echo "2. Manually join this node to Docker Swarm if needed"
fi
echo "3. SSH access: ssh ${USER_NAME}@${VM_IP}"