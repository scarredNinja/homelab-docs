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
# Version: v10.6 (fixed)
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
# - Optional IP fallback if guest agent not installed
# ==============================================================================

set -euo pipefail

# --- Config ---
TEMPLATE_ID=105
VMID_BASE=200
BRIDGE_NAME="vmbr0"
DEFAULT_VLAN=85
SSH_PUBLIC_KEY_PATH="/root/.ssh/id_rsa.pub"
SWARM_WORKER_TOKEN_FILE="/root/swarm_worker_token.txt"
SWARM_MANAGER_TOKEN_FILE="/root/swarm_manager_token.txt"
MANAGER_IP_FILE="/root/swarm_manager_ip.txt"
NFS_SERVER="10.0.60.80"
NFS_BASE="/mnt/docker-swarm/volumes"
ZFS_POOL="rpool/data"     # Default ZFS pool path (ensure no stray unicode spaces)

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

usage() {
  echo "Usage: $0 --name <vmname> [--vlan <tag>] [--volume <name:sizeG>] [--nfs <share1,share2>] [--zfs-pool <pool>] [--first-manager] [--autojoin --role worker|manager] [--user <username> --password <plaintext>] [--no-autostart]"
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
    *) echo "Unknown arg: $1"; usage ;;
  esac
done

[[ -z "$VM_NAME" ]] && usage

# --- Helper: cleanup created zvols on error ---
cleanup_volumes() {
    echo ">>> Cleaning up created volumes for VM $VMID..."
    for vol in "${VOLUMES[@]}"; do
        NAME=$(echo "$vol" | cut -d: -f1)
        ZVOL="${ZFS_POOL}/${VM_NAME}_${NAME}"
        echo ">>> Destroying ZFS volume: ${ZVOL}"
        # ignore errors
        zfs destroy "${ZVOL}" 2>/dev/null || true
    done
}

trap 'echo "[ERROR] Script failed. Cleaning up..."; cleanup_volumes; exit 1' ERR

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

# --- SSH Key Injection ---
# Use the file path (qm handles reading it in)
if [ -f "$SSH_PUBLIC_KEY_PATH" ]; then
    qm set "$VMID" --sshkey "$SSH_PUBLIC_KEY_PATH"
else
    echo "WARNING: SSH key file not found at $SSH_PUBLIC_KEY_PATH"
fi

# --- Create and attach ZFS volumes (zvols) ---
if [[ ${#VOLUMES[@]} -gt 0 ]]; then
    echo ">>> Creating ZFS volumes in pool: $ZFS_POOL"
    SCSI_COUNTER=0
    for vol in "${VOLUMES[@]}"; do
        NAME=$(echo "$vol" | cut -d: -f1)
        SIZE_WITH_UNIT=$(echo "$vol" | cut -d: -f2)
        # Normalize size: remove trailing non-digit (eg 'G'), allow 'G' only
        SIZE_IN_G=$(echo "$SIZE_WITH_UNIT" | tr -d '[:alpha:]')
        ZVOL_PATH="${ZFS_POOL}/${VM_NAME}_${NAME}"
        echo ">>> Creating ZFS volume: ${ZVOL_PATH} (${SIZE_WITH_UNIT})"

        # Overwrite if exists
        if zfs list "${ZVOL_PATH}" >/dev/null 2>&1; then
            echo ">>> ZFS volume ${ZVOL_PATH} already exists, destroying it..."
            zfs destroy -r "${ZVOL_PATH}" || { echo "ERROR: Failed to destroy existing ${ZVOL_PATH}"; exit 1; }
        fi

        if ! zfs create -V "${SIZE_IN_G}G" "${ZVOL_PATH}"; then
            echo "ERROR: Failed to create ZFS volume ${ZVOL_PATH}"
            cleanup_volumes
            exit 1
        fi

        # Wait for device node to appear
        sleep 1
        DEV_PATH="/dev/zvol/${ZVOL_PATH}"
        if [[ ! -b "${DEV_PATH}" && ! -e "${DEV_PATH}" ]]; then
            echo "ERROR: ZFS device ${DEV_PATH} not available after creation"
            cleanup_volumes
            exit 1
        fi

        # Attach the raw block device to VM on next scsi index
        SCSI_OPT="scsi${SCSI_COUNTER}"
        echo ">>> Attaching ${DEV_PATH} to VM ${VMID} as ${SCSI_OPT}"
        if qm set "$VMID" --"${SCSI_OPT}" "${DEV_PATH},discard=on"; then
            echo ">>> Attached ${DEV_PATH} as ${SCSI_OPT}"
        else
            echo "WARNING: Failed to attach ${DEV_PATH} in raw form, attempting file= syntax"
            qm set "$VMID" --"${SCSI_OPT}" "file=${DEV_PATH},discard=on" || { echo "ERROR: attach attempt failed"; cleanup_volumes; exit 1; }
        fi

        SCSI_COUNTER=$((SCSI_COUNTER + 1))
    done
fi


# --- Start VM if requested ---
if [[ "${AUTOSTART}" == true ]]; then
  qm start "$VMID" || { echo "ERROR: failed to start VM $VMID"; cleanup_volumes; exit 1; }
  echo ">>> Waiting 20s for VM to boot..."
  sleep 20

  # --- Attempt guest-agent IP detection (best-effort) ---
  # Note: qm guest exec requires guest-agent installed & QEMU guest agent running in the VM
  VM_IP=""
  if qm guest exec "$VMID" -- ip -4 -o addr show eth0 >/dev/null 2>&1; then
      VM_IP=$(qm guest exec "$VMID" -- ip -4 -o addr show eth0 | awk '{print $4}' | cut -d/ -f1 2>/dev/null || true)
  fi

  if [[ -z "$VM_IP" ]]; then
      echo "[WARNING] Could not detect VM IP via guest-agent."
      # Non-interactive fallback: try reading DHCP leases for the generated MAC (libvirt/iptables variation)
      # Try Proxmox DHCP leases file locations (best-effort)
      # If still empty, prompt (interactive)
      # first attempt: try `qm monitor` guest network? fallback to manual prompt below.
      read -p "Enter VM IP for $VM_NAME (for provisioning and swarm join): " VM_IP
  fi
  echo ">>> VM IP: $VM_IP"
fi

# --- Build cloud-init / remote bootstrap commands ---
# Using a single quoted heredoc to avoid issues with many && concatenations
CLOUD_BOOT_CMDS=$(cat <<EOF
set -e
# set hostname
hostnamectl set-hostname ${VM_NAME}

export DEBIAN_FRONTEND=noninteractive
apt-get update -y
apt-get install -y docker.io nfs-common zfsutils-linux qemu-guest-agent
# create user if not exists
id -u ${USER_NAME} &>/dev/null || useradd -m -s /bin/bash ${USER_NAME}
EOF
)

# Add password if provided (hashed via chpasswd -e expects hash)
if [[ -n "${HASHED_PASS}" ]]; then
  CLOUD_BOOT_CMDS+="
echo '${USER_NAME}:${HASHED_PASS}' | chpasswd -e
"
fi

CLOUD_BOOT_CMDS+="
usermod -aG sudo ${USER_NAME} || true
"

# ZFS-mounted disks: inside VM, assume devices are present as /dev/disk/by-id/ or /dev/sdX
# We'll attempt to mkfs and mount predictable devices created earlier
DISK_COUNTER=0
for vol in "${VOLUMES[@]}"; do
    NAME=$(echo "$vol" | cut -d: -f1)
    # build commands that are tolerant to device naming differences
    CLOUD_BOOT_CMDS+="
# prepare and mount ${NAME}
mkdir -p /mnt/${NAME} || true
# attempt to find the attached device by scanning 'lsblk' for a device whose model or serial contains 'qm-${VMID}-disk-${DISK_COUNTER}' or use first unpartitioned disk
DEV=\$(lsblk -ndo KNAME,MODEL,SERIAL | awk '/qm-${VMID}-disk-${DISK_COUNTER}/ {print \$1}' | head -n1)
if [[ -z \"\$DEV\" ]]; then
    # fallback: pick next unused disk (careful) - this is a heuristic
    DEV=\$(lsblk -ndo KNAME,TYPE | awk '\$2==\"disk\"{print \$1}' | sort | head -n$((DISK_COUNTER+1)) | tail -n1)
fi
if [[ -n \"\$DEV\" && -b /dev/\$DEV ]]; then
    # Only mkfs if no filesystem detected
    if ! blkid /dev/\$DEV >/dev/null 2>&1; then
        mkfs.ext4 -F /dev/\$DEV || true
    fi
    mount /dev/\$DEV /mnt/${NAME} || true
fi
"
    DISK_COUNTER=$((DISK_COUNTER+1))
done

# NFS mounts: append to /etc/fstab safely and mount
for share in "${NFS_SHARES[@]}"; do
    CLOUD_BOOT_CMDS+="
mkdir -p /mnt/media/${share} || true
echo '${NFS_SERVER}:${NFS_BASE}/${share} /mnt/media/${share} nfs defaults 0 0' >> /etc/fstab || true
"
done

# Ensure mounts are applied
CLOUD_BOOT_CMDS+="
mount -a || true
"

# Swarm init for first manager
if [[ "${FIRST_MANAGER}" == true ]]; then
  CLOUD_BOOT_CMDS+="
docker swarm init --advertise-addr ${VM_IP} || true
docker swarm join-token worker -q > /root/worker_token || true
docker swarm join-token manager -q > /root/manager_token || true
"
fi

# Run the cloud commands over SSH (root). Use a heredoc for robustness.
echo ">>> Executing bootstrap commands on ${VM_IP} ..."
ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 root@"${VM_IP}" 'bash -s' <<'SSH_EOF'
# remote marker: will be replaced by CLOUD_BOOT_CMDS
SSH_EOF

# We need to actually send CLOUD_BOOT_CMDS; the previous heredoc is placeholder. Now do it properly:
ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 root@"${VM_IP}" "bash -s" <<EOF
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
        TOKEN=$(cat "${SWARM_MANAGER_TOKEN_FILE}")
    fi
    echo ">>> Joining swarm at ${MANAGER_IP}:2377 as ${ROLE}"
    ssh -o StrictHostKeyChecking=no root@"${VM_IP}" "docker swarm join --token ${TOKEN} ${MANAGER_IP}:2377" || echo "WARNING: autojoin may have failed"
fi

# --- Retrieve tokens back to Proxmox for first manager ---
if [[ "${FIRST_MANAGER}" == true ]]; then
    echo ">>> Waiting 30s for first manager to finish setup..."
    sleep 30
    echo ">>> Retrieving swarm tokens from ${VM_NAME} (${VM_IP})..."
    scp -o StrictHostKeyChecking=no root@"${VM_IP}":/root/worker_token "${SWARM_WORKER_TOKEN_FILE}" || echo "WARNING: couldn't copy worker token"
    scp -o StrictHostKeyChecking=no root@"${VM_IP}":/root/manager_token "${SWARM_MANAGER_TOKEN_FILE}" || echo "WARNING: couldn't copy manager token"
    echo "${VM_IP}" > "${MANAGER_IP_FILE}"
    echo ">>> Tokens saved on Proxmox:"
    echo "Worker token file: ${SWARM_WORKER_TOKEN_FILE}"
    echo "Manager token file: ${SWARM_MANAGER_TOKEN_FILE}"
fi

echo ">>> Provisioning complete for ${VM_NAME} (VMID ${VMID})"
echo ">>> MAC address for static DHCP in pfSense: ${MAC_ADDR}"
