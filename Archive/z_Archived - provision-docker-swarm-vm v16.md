---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
#!/bin/bash

  

# ==============================================================================

# Docker Swarm VM Provisioning Script (Proxmox)

# Version: v16.0 (Complete rewrite with enhanced error handling)

# ==============================================================================

  

set -euo pipefail

  

# --- Configuration ---

TEMPLATE_ID=100

BRIDGE_NAME="vmbr0"

DEFAULT_VLAN=85

SSH_PUBLIC_KEY_PATH="/root/.ssh/id_rsa.pub"

SWARM_WORKER_TOKEN_FILE="/root/swarm_worker_token.txt"

SWARM_MANAGER_TOKEN_FILE="/root/swarm_manager_token.txt"

MANAGER_IP_FILE="/root/swarm_manager_ip.txt"

DEFAULT_NFS_SERVER="10.0.60.80"

DEFAULT_NFS_BASE="/mnt/docker-swarm/volumes"

ZFS_POOL="rpool/data"

DEFAULT_BOOT_TIMEOUT=45

SSH_USER="sussh"

  

# --- Script Parameters ---

VM_NAME=""

VLAN=$DEFAULT_VLAN

VOLUMES=()

NFS_SHARES=()

FIRST_MANAGER=false

AUTOJOIN=false

ROLE="worker"

CLOUD_INIT_USER="homelab"

CLOUD_INIT_PASSWORD=""

AUTOSTART=true

BOOT_TIMEOUT=$DEFAULT_BOOT_TIMEOUT

CLEANUP_ON_FAILURE=true

VMID=""

  

# --- Usage ---

usage() {

    cat <<EOF

Usage: $0 --name <vmname> [OPTIONS]

  

Required:

  --name <vmname>               Name for the VM

  

Optional:

  --vlan <tag>                  VLAN tag (default: $DEFAULT_VLAN)

  --volume <name:sizeG>         Add ZFS volume (can specify multiple times)

  --nfs <spec>                  Mount NFS share (can specify multiple times)

                                Format: [server:]path[:mountname]

                                  server defaults to $DEFAULT_NFS_SERVER

                                  mountname defaults to basename of path

                                Examples:

                                  --nfs /volume/docker-swarm/grafana

                                  --nfs 10.0.60.80:/volume/docker-swarm/portainer:portainer

                                  --nfs 192.168.1.5:/exports/data:shared-data

  --zfs-pool <pool>             ZFS pool path (default: $ZFS_POOL)

  --first-manager               Initialize as first Docker Swarm manager

  --autojoin                    Auto-join existing swarm

  --role <worker|manager>       Role when auto-joining (default: worker)

  --user <username>             Cloud-init username (default: $CLOUD_INIT_USER)

  --password <plaintext>        Cloud-init password

  --no-autostart                Don't start VM automatically

  --boot-timeout <seconds>      Boot timeout (default: $DEFAULT_BOOT_TIMEOUT)

  --no-cleanup                  Preserve VM on failure for debugging

  

Examples:

  # Create first manager

  $0 --name swarm-manager-01 --first-manager --password mypass123

  

  # Create worker with volume

  $0 --name swarm-worker-01 --volume data:50G --autojoin --role worker

  

  # Create with NFS shares (multiple formats)

  $0 --name swarm-worker-02 --nfs /volume/docker-swarm/grafana --autojoin

  $0 --name swarm-worker-03 --nfs 10.0.60.80:/exports/portainer:portainer --autojoin

EOF

    exit 1

}

  

# --- Parse Arguments ---

while [[ $# -gt 0 ]]; do

    case "$1" in

        --name) VM_NAME="$2"; shift 2 ;;

        --name) VM_NAME="$2"; shift 2 ;;

        --vlan) VLAN="$2"; shift 2 ;;

        --volume) VOLUMES+=("$2"); shift 2 ;;

        --nfs) NFS_SHARES+=("$2"); shift 2 ;;

        --zfs-pool) ZFS_POOL="$2"; shift 2 ;;

        --first-manager) FIRST_MANAGER=true; shift ;;

        --autojoin) AUTOJOIN=true; shift ;;

        --role) ROLE="$2"; shift 2 ;;

        --user) CLOUD_INIT_USER="$2"; shift 2 ;;

        --password) CLOUD_INIT_PASSWORD="$2"; shift 2 ;;

        --no-autostart) AUTOSTART=false; shift ;;

        --boot-timeout) BOOT_TIMEOUT="$2"; shift 2 ;;

        --no-cleanup) CLEANUP_ON_FAILURE=false; shift ;;

        -h|--help) usage ;;

        *) echo "ERROR: Unknown argument: $1"; usage ;;

    esac

done

  

# --- Validation ---

[[ -z "$VM_NAME" ]] && { echo "ERROR: VM name is required"; usage; }

[[ -z "$CLOUD_INIT_USER" ]] && { echo "ERROR: Cloud-init user cannot be empty"; exit 1; }

  

if [[ ! -f "$SSH_PUBLIC_KEY_PATH" ]]; then

    echo "ERROR: SSH public key not found at $SSH_PUBLIC_KEY_PATH"

    exit 1

fi

  

# --- Cleanup Function ---

cleanup_on_failure() {

    local exit_code=$?

    echo ""

    echo "=========================================================================="

    echo "[ERROR] Script failed with exit code: $exit_code"

    echo "=========================================================================="

    if [[ "$CLEANUP_ON_FAILURE" == true ]]; then

        if [[ -n "$VMID" ]] && qm status "$VMID" &>/dev/null; then

            echo ">>> Cleaning up VM $VMID..."

            qm stop "$VMID" --timeout 10 2>/dev/null || true

            sleep 2

            qm destroy "$VMID" --destroy-unreferenced-disks --purge 2>/dev/null || true

            echo ">>> VM $VMID destroyed"

        fi

        if [[ ${#VOLUMES[@]} -gt 0 ]]; then

            echo ">>> Cleaning up ZFS volumes..."

            for vol in "${VOLUMES[@]}"; do

                NAME=$(echo "$vol" | cut -d: -f1)

                ZVOL="${ZFS_POOL}/${VM_NAME}_${NAME}"

                if zfs list "$ZVOL" &>/dev/null; then

                    echo ">>> Destroying ZFS volume: ${ZVOL}"

                    zfs destroy -r "${ZVOL}" 2>/dev/null || true

                fi

            done

        fi

        echo ">>> Cleanup completed"

    else

        echo ">>> Cleanup disabled - VM $VMID preserved for inspection"

        echo ">>> To manually clean up: qm destroy $VMID --purge"

    fi

    exit "$exit_code"

}

  

trap cleanup_on_failure ERR INT TERM

  

# --- Helper Functions ---

generate_mac() {

    local hexchars="0123456789abcdef"

    local mac="52:54:00"

    for i in {1..3}; do

        local a=${hexchars:$((RANDOM%16)):1}

        local b=${hexchars:$((RANDOM%16)):1}

        mac="${mac}:$a$b"

    done

    echo "$mac"

}

  

wait_for_ssh() {

    local ip="$1"

    local max_attempts=30

    local attempt=0

    echo ">>> Waiting for SSH connectivity to ${SSH_USER}@${ip}..."

    while [[ $attempt -lt $max_attempts ]]; do

        if ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 -o BatchMode=yes \

            "${SSH_USER}@${ip}" "exit 0" &>/dev/null; then

            echo ">>> SSH connection successful"

            return 0

        fi

        attempt=$((attempt + 1))

        echo ">>> Waiting for SSH... ($attempt/$max_attempts)"

        sleep 5

    done

    echo "ERROR: SSH connection failed after $max_attempts attempts"

    return 1

}

  

# --- Start Provisioning ---

echo "=========================================================================="

echo "Proxmox Docker Swarm VM Provisioning"

echo "=========================================================================="

echo "VM Name:        $VM_NAME"

echo "VLAN:           $VLAN"

echo "Cloud-init User: $CLOUD_INIT_USER"

[[ ${#VOLUMES[@]} -gt 0 ]] && echo "Volumes:        ${VOLUMES[*]}"

[[ ${#NFS_SHARES[@]} -gt 0 ]] && echo "NFS Shares:     ${NFS_SHARES[*]}"

[[ "$FIRST_MANAGER" == true ]] && echo "Role:           First Swarm Manager"

[[ "$AUTOJOIN" == true ]] && echo "Auto-join:      Yes (as $ROLE)"

echo "=========================================================================="

  

# --- Allocate VM ID ---

if [[ "$FIRST_MANAGER" == true ]] || [[ "$ROLE" == "manager" ]]; then

    BASE_VMID=100

    echo ">>> Role: Manager (using VMID base 100)"

else

    BASE_VMID=200

    echo ">>> Role: Worker (using VMID base 200)"

fi

  

VMID=$BASE_VMID

while qm status "$VMID" &>/dev/null; do

    VMID=$((VMID + 1))

done

echo ">>> Allocated VM ID: $VMID"

  

# --- Generate MAC Address ---

MAC_ADDR="$(generate_mac)"

echo ">>> Generated MAC address: $MAC_ADDR"

  

# --- Clone Template ---

echo ">>> Cloning template $TEMPLATE_ID..."

if ! qm clone "$TEMPLATE_ID" "$VMID" --name "$VM_NAME" --full; then

    echo "ERROR: Failed to clone template"

    exit 1

fi

  

# --- Configure Network ---

echo ">>> Configuring network..."

qm set "$VMID" --net0 "virtio,bridge=${BRIDGE_NAME},tag=${VLAN},macaddr=${MAC_ADDR}"

  

# --- Configure Cloud-init ---

echo ">>> Configuring cloud-init..."

qm set "$VMID" --ciuser "$CLOUD_INIT_USER"

  

if ! qm set "$VMID" --sshkeys "$SSH_PUBLIC_KEY_PATH"; then

    echo "ERROR: Failed to set SSH keys"

    exit 1

fi

  

if [[ -n "$CLOUD_INIT_PASSWORD" ]]; then

    if ! qm set "$VMID" --cipassword "$CLOUD_INIT_PASSWORD"; then

        echo "ERROR: Failed to set cloud-init password"

        exit 1

    fi

    echo ">>> Cloud-init password configured"

fi

  

# --- Create ZFS Volumes ---

if [[ ${#VOLUMES[@]} -gt 0 ]]; then

    echo ">>> Creating ZFS volumes..."

    for VOL_ARG in "${VOLUMES[@]}"; do

        VOLNAME="${VM_NAME}_$(echo "$VOL_ARG" | cut -d: -f1)"

        SIZE="$(echo "$VOL_ARG" | cut -d: -f2)"

        ZVOL_PATH="${ZFS_POOL}/${VOLNAME}"

        if zfs list "$ZVOL_PATH" &>/dev/null; then

            echo ">>> ZFS volume already exists: $ZVOL_PATH"

        else

            echo ">>> Creating ZFS volume: $ZVOL_PATH ($SIZE)"

            zfs create -V "$SIZE" "$ZVOL_PATH"

        fi

        # Find next available SCSI ID

        SCSI_ID=1

        while qm config "$VMID" | grep -q "^scsi$SCSI_ID:"; do

            SCSI_ID=$((SCSI_ID + 1))

        done

        echo ">>> Attaching volume to VM as scsi${SCSI_ID}"

        qm set "$VMID" --scsi${SCSI_ID} "/dev/zvol/${ZVOL_PATH},discard=on"

    done

fi

  

# --- Start VM ---

VM_IP=""

if [[ "$AUTOSTART" == true ]]; then

    echo ">>> Starting VM..."

    if ! qm start "$VMID"; then

        echo "ERROR: Failed to start VM"

        exit 1

    fi

    echo ">>> Waiting ${BOOT_TIMEOUT}s for VM to boot..."

    sleep "$BOOT_TIMEOUT"

    # Detect IP address

    echo ">>> Detecting VM IP address..."

    for i in {1..24}; do

        VM_IP=$(qm agent "$VMID" network-get-interfaces 2>/dev/null \

            | jq -r --arg MAC "$MAC_ADDR" '

                .[] | select(.["hardware-address"]==$MAC) |

                .["ip-addresses"][] | select(.["ip-address-type"]=="ipv4") |

                .["ip-address"]

            ' 2>/dev/null || true)

        if [[ -n "$VM_IP" && "$VM_IP" != "null" ]]; then

            echo ">>> Detected IP address: $VM_IP"

            break

        fi

        echo ">>> Waiting for guest agent to report IP... ($i/24)"

        sleep 10

    done

    if [[ -z "$VM_IP" || "$VM_IP" == "null" ]]; then

        echo "ERROR: Could not detect VM IP address"

        echo ">>> Troubleshooting:"

        echo "    1. Check VM console: qm terminal $VMID"

        echo "    2. Verify guest agent is installed in template"

        echo "    3. Check DHCP server logs"

        exit 1

    fi

    wait_for_ssh "$VM_IP"

else

    echo ">>> Skipping VM start (--no-autostart specified)"

fi

  

# --- Build Provisioning Commands ---

PROVISION_SCRIPT="#!/bin/bash

set -e

  

echo '========================================='

echo 'VM Provisioning for ${VM_NAME}'

echo '========================================='

  

# Set hostname

echo '>>> Setting hostname...'

sudo hostnamectl set-hostname ${VM_NAME}

  

# Update and install packages

echo '>>> Updating system and installing packages...'

export DEBIAN_FRONTEND=noninteractive

sudo apt-get update -y

sudo apt-get install -y nfs-common zfsutils-linux parted

  

# Enable Docker

echo '>>> Enabling Docker service...'

sudo systemctl enable --now docker

sleep 5

  

# Ensure cloud-init user exists

echo '>>> Configuring user ${CLOUD_INIT_USER}...'

if ! id -u ${CLOUD_INIT_USER} &>/dev/null; then

    sudo useradd -m -s /bin/bash ${CLOUD_INIT_USER}

fi

sudo usermod -aG sudo,docker ${CLOUD_INIT_USER} || true

  

# Enable guest agent

echo '>>> Enabling QEMU guest agent...'

sudo systemctl enable --now qemu-guest-agent.service || true

"

  

# --- Add Volume Mounting ---

if [[ ${#VOLUMES[@]} -gt 0 ]]; then

    PROVISION_SCRIPT+="

echo '>>> Configuring volumes...'

"

    VOLUME_INDEX=0

    for VOL_ARG in "${VOLUMES[@]}"; do

        VOLNAME="$(echo "$VOL_ARG" | cut -d: -f1)"

        MOUNT_POINT="/mnt/data/${VOLNAME}"

        VOLUME_INDEX=$((VOLUME_INDEX + 1))

        PROVISION_SCRIPT+="

echo '>>> Setting up volume ${VOLUME_INDEX}: ${VOLNAME}'

sudo mkdir -p ${MOUNT_POINT}

  

# Find raw disks only (exclude partitions)

CURRENT_DISK=0

for dev in \$(ls /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi* 2>/dev/null | grep -v 'part' | sort); do

    # Skip first disk (OS disk at scsi0)

    if [ \$CURRENT_DISK -eq 0 ]; then

        CURRENT_DISK=\$((CURRENT_DISK + 1))

        echo '>>> Skipping OS disk: \$dev'

        continue

    fi

    # Check if this is the disk we want for this volume

    if [ \$CURRENT_DISK -eq ${VOLUME_INDEX} ]; then

        # Check if this specific mount point is already mounted

        if mount | grep -q \"on ${MOUNT_POINT} \"; then

            echo '>>> ${MOUNT_POINT} already mounted, skipping...'

        else

            echo '>>> Formatting \$dev as ext4...'

            sudo mkfs.ext4 -F \$dev

            echo '>>> Adding to /etc/fstab...'

            echo \"\$dev ${MOUNT_POINT} ext4 defaults 0 2\" | sudo tee -a /etc/fstab

            echo '>>> Mounting ${MOUNT_POINT}...'

            sudo mount ${MOUNT_POINT}

            sudo chown -R ${CLOUD_INIT_USER}:${CLOUD_INIT_USER} ${MOUNT_POINT}

            echo '>>> Volume ${VOLNAME} configured successfully at ${MOUNT_POINT}'

        fi

        break

    fi

    CURRENT_DISK=\$((CURRENT_DISK + 1))

done

"

    done

fi

  

# --- Add NFS Mounting ---

if [[ ${#NFS_SHARES[@]} -gt 0 ]]; then

    PROVISION_SCRIPT+="

echo '>>> Configuring NFS mounts...'

sudo mkdir -p /mnt/nfs

"

    for NFS_SPEC in "${NFS_SHARES[@]}"; do

        # Parse NFS spec: [server:]path[:mountname]

        # Split by colons

        IFS=':' read -ra PARTS <<< "$NFS_SPEC"

        if [[ ${#PARTS[@]} -eq 1 ]]; then

            # Format: path (use defaults)

            NFS_SERVER="$DEFAULT_NFS_SERVER"

            NFS_PATH="${PARTS[0]}"

            MOUNT_NAME="$(basename "$NFS_PATH")"

        elif [[ ${#PARTS[@]} -eq 2 ]]; then

            # Could be server:path OR path:mountname

            # Check if first part looks like IP/hostname (contains dots or is hostname-like)

            if [[ "${PARTS[0]}" =~ ^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$ ]] || [[ "${PARTS[0]}" =~ ^[a-zA-Z0-9-]+(\.[a-zA-Z0-9-]+)*$ && ! "${PARTS[0]}" =~ ^/ ]]; then

                # Format: server:path

                NFS_SERVER="${PARTS[0]}"

                NFS_PATH="${PARTS[1]}"

                MOUNT_NAME="$(basename "$NFS_PATH")"

            else

                # Format: path:mountname

                NFS_SERVER="$DEFAULT_NFS_SERVER"

                NFS_PATH="${PARTS[0]}"

                MOUNT_NAME="${PARTS[1]}"

            fi

        elif [[ ${#PARTS[@]} -eq 3 ]]; then

            # Format: server:path:mountname

            NFS_SERVER="${PARTS[0]}"

            NFS_PATH="${PARTS[1]}"

            MOUNT_NAME="${PARTS[2]}"

        else

            echo "ERROR: Invalid NFS spec format: $NFS_SPEC"

            exit 1

        fi

        PROVISION_SCRIPT+="

echo '>>> Mounting NFS share: ${NFS_SERVER}:${NFS_PATH} -> /mnt/nfs/${MOUNT_NAME}'

sudo mkdir -p /mnt/nfs/${MOUNT_NAME}

if ! grep -q '/mnt/nfs/${MOUNT_NAME}' /etc/fstab; then

    echo '${NFS_SERVER}:${NFS_PATH} /mnt/nfs/${MOUNT_NAME} nfs defaults,_netdev 0 0' | sudo tee -a /etc/fstab

fi

"

    done

    PROVISION_SCRIPT+="

echo '>>> Mounting all NFS shares...'

sudo mount -a

"

fi

  

# --- Add Swarm Init for First Manager ---

if [[ "$FIRST_MANAGER" == true ]]; then

    PROVISION_SCRIPT+="

echo '>>> Initializing Docker Swarm...'

for i in {1..3}; do

    if sudo docker swarm init --advertise-addr ${VM_IP}; then

        echo '>>> Docker Swarm initialized successfully'

        sudo sh -c 'docker swarm join-token worker -q > /root/worker_token'

        sudo sh -c 'docker swarm join-token manager -q > /root/manager_token'

        break

    else

        echo '>>> Swarm init failed, retrying (\$i/3)...'

        sleep 5

    fi

done

"

fi

  

PROVISION_SCRIPT+="

echo '========================================='

echo 'Provisioning completed successfully'

echo '========================================='

"

  

# --- Execute Provisioning ---

if [[ "$AUTOSTART" == true ]]; then

    echo ">>> Executing provisioning script on VM..."

    if ! ssh -o StrictHostKeyChecking=no -o ConnectTimeout=30 -o ServerAliveInterval=10 \

        "${SSH_USER}@${VM_IP}" "bash -s" <<< "$PROVISION_SCRIPT" 2>&1 | tee "/tmp/provision_${VMID}.log"; then

        echo ""

        echo "ERROR: Provisioning failed"

        echo ">>> Check logs at: /tmp/provision_${VMID}.log"

        exit 1

    fi

    echo ">>> Provisioning completed successfully"

fi

  

# --- Auto-join Swarm ---

if [[ "$AUTOJOIN" == true && "$FIRST_MANAGER" != true && "$AUTOSTART" == true ]]; then

    echo ">>> Auto-joining Docker Swarm as ${ROLE}..."

    # Validate prerequisites

    if [[ ! -f "$MANAGER_IP_FILE" ]]; then

        echo "ERROR: Manager IP file not found: $MANAGER_IP_FILE"

        echo "       Provision the first manager with --first-manager first"

        exit 1

    fi

    MANAGER_IP=$(<"$MANAGER_IP_FILE")

    if [[ -z "$MANAGER_IP" ]]; then

        echo "ERROR: Manager IP is empty"

        exit 1

    fi

    TOKEN_FILE="$SWARM_WORKER_TOKEN_FILE"

    [[ "$ROLE" == "manager" ]] && TOKEN_FILE="$SWARM_MANAGER_TOKEN_FILE"

    if [[ ! -f "$TOKEN_FILE" ]]; then

        echo "ERROR: Token file not found: $TOKEN_FILE"

        exit 1

    fi

    TOKEN=$(<"$TOKEN_FILE")

    if [[ -z "$TOKEN" ]]; then

        echo "ERROR: Swarm token is empty"

        exit 1

    fi

    # Attempt to join with retries

    JOIN_SUCCESS=false

    for i in {1..3}; do

        echo ">>> Join attempt $i/3..."

        if ssh -o StrictHostKeyChecking=no "${SSH_USER}@${VM_IP}" \

            "sudo docker swarm join --token ${TOKEN} ${MANAGER_IP}:2377" 2>&1 | tee -a "/tmp/swarm_join_${VMID}.log"; then

            echo ">>> Successfully joined swarm as ${ROLE}"

            JOIN_SUCCESS=true

            break

        fi

        if [[ $i -lt 3 ]]; then

            echo ">>> Join failed, retrying in 5 seconds..."

            sleep 5

        fi

    done

    if [[ "$JOIN_SUCCESS" != true ]]; then

        echo "ERROR: Failed to join swarm after 3 attempts"

        echo ">>> Check logs: /tmp/swarm_join_${VMID}.log"

        echo ">>> Manual join command:"

        echo "    ssh ${SSH_USER}@${VM_IP}"

        echo "    sudo docker swarm join --token ${TOKEN} ${MANAGER_IP}:2377"

        exit 1

    fi

fi

  

# --- Retrieve Tokens for First Manager ---

if [[ "$FIRST_MANAGER" == true && "$AUTOSTART" == true ]]; then

    echo ">>> Retrieving swarm tokens..."

    sleep 10

    if ! ssh "${SSH_USER}@${VM_IP}" 'sudo cat /root/worker_token' > "$SWARM_WORKER_TOKEN_FILE"; then

        echo "ERROR: Failed to retrieve worker token"

        exit 1

    fi

    if ! ssh "${SSH_USER}@${VM_IP}" 'sudo cat /root/manager_token' > "$SWARM_MANAGER_TOKEN_FILE"; then

        echo "ERROR: Failed to retrieve manager token"

        exit 1

    fi

    if [[ ! -s "$SWARM_WORKER_TOKEN_FILE" ]] || [[ ! -s "$SWARM_MANAGER_TOKEN_FILE" ]]; then

        echo "ERROR: Token files are empty"

        exit 1

    fi

    echo "${VM_IP}" > "$MANAGER_IP_FILE"

    echo ">>> Tokens saved:"

    echo "    Worker:  $SWARM_WORKER_TOKEN_FILE"

    echo "    Manager: $SWARM_MANAGER_TOKEN_FILE"

    echo "    IP:      $MANAGER_IP_FILE"

fi

  

# --- Verify Guest Agent ---

if [[ "$AUTOSTART" == true ]]; then

    echo ">>> Verifying guest agent..."

    for i in {1..5}; do

        if qm agent "$VMID" ping &>/dev/null; then

            echo ">>> Guest agent is responding"

            break

        fi

        if [[ $i -eq 5 ]]; then

            echo "WARNING: Guest agent not responding (this may be normal)"

        else

            sleep 5

        fi

    done

fi

  

# Remove error trap

trap - ERR INT TERM

  

# --- Final Summary ---

echo ""

echo "=========================================================================="

echo "PROVISIONING COMPLETED SUCCESSFULLY"

echo "=========================================================================="

echo "VM Name:        $VM_NAME"

echo "VM ID:          $VMID"

[[ -n "$VM_IP" ]] && echo "IP Address:     $VM_IP"

echo "MAC Address:    $MAC_ADDR"

echo "VLAN:           $VLAN"

echo "Cloud-init User: $CLOUD_INIT_USER"

echo "SSH User:       $SSH_USER"

[[ ${#VOLUMES[@]} -gt 0 ]] && echo "Volumes:        ${VOLUMES[*]}"

[[ ${#NFS_SHARES[@]} -gt 0 ]] && echo "NFS Shares:     ${NFS_SHARES[*]}"

  

if [[ "$FIRST_MANAGER" == true ]]; then

    echo "Role:           Swarm Manager (First)"

    echo ""

    echo "Next Steps:"

    echo "  1. Configure static DHCP for MAC: $MAC_ADDR"

    echo "  2. Join workers:"

    echo "     $0 --name worker-01 --autojoin --role worker"

elif [[ "$AUTOJOIN" == true ]]; then

    echo "Role:           Swarm ${ROLE^}"

    echo "Manager:        $MANAGER_IP"

    echo ""

    echo "Verify with:"

    echo "  ssh ${SSH_USER}@${VM_IP} 'sudo docker node ls'"

else

    echo ""

    echo "Next Steps:"

    echo "  1. Configure static DHCP for MAC: $MAC_ADDR"

    [[ -n "$VM_IP" ]] && echo "  2. SSH access: ssh ${SSH_USER}@${VM_IP}"

fi

  

echo "=========================================================================="
