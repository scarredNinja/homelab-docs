#HomeLabRebuild/Swarm #HomeLabRebuild/Automation
# Docker Swarm Worker VM Provisioning (NFS + multi ZFS + autojoin)

## Usage

Usage: ./provision_vm.sh [--autojoin] [--nfs share1,share2,...] [--volume volname,size,type;...] [--user name] [--uid uid] [--password pass] <vm-name>

|Option|Description|
|---|---|
|`--autojoin`|Automatically join the VM to your Docker Swarm cluster.|
|`--nfs share1,share2,...`|Comma-separated list of NFS shares to mount at `/mnt/media/<share>`.|
|`--volume volname,size,type;...`|Semi-colon separated list of volumes to create. Each volume is `name,size,type`, where `type` can be `zfs` or `dir`. Example: `db,20G,zfs;cache,5G,dir`.|
|`--user name`|Name of the main user inside the VM (default: `docker`).|
|`--uid uid`|Optional UID for the main user.|
|`--password pass`|Optional password for the main user.|
|`<vm-name>`|The name of the VM to create.|


./provision-swarm-worker.sh --autojoin --user plex --uid 1500 --password 'Plexface!' --nfs movies,tvshows,animation --volume configs,100G,zfs  plex-01
  
## Script

#!/bin/bash

# ==============================================================================
# VM Provisioning Script for pfSense-assigned IPs
# (Proxmox storage fixed)
# ==============================================================================

# --- Configuration ---
# ---------------------
TEMPLATE_ID=101
VMID_BASE=200
BRIDGE_NAME="vmbr0"
DEFAULT_VLAN=85
SSH_PUBLIC_KEY_PATH="/root/.ssh/id_rsa.pub"
SWARM_JOIN_TOKEN_FILE="/root/swarm_worker_token.txt"
SWARM_MANAGER_IP="10.0.60.30"
ZFS_CONFIG_POOL="rspool/vm-configs"
ZFS_DB_POOL="rspool/vm-db"

# Storage configuration
# Use 'local' for snippets (supports snippets)
# Use 'local-zfs' for images (supports images)
SNIPPETS_STORAGE="local"
CLOUD_INIT_STORAGE="local-zfs"
SNIPPETS_DIR="/var/lib/vz/snippets" # Path for local storage snippets


# --- Functions ---
# -----------------
usage() {
    echo "Usage: $0 [--autojoin] [--nfs share1,share2,...] [--volume volname,size,type;...] [--user name] [--uid uid] [--password pass] <vm-name>"
    exit 1
}

generate_mac() {
    printf '52:54:%02x:%02x:%02x:%02x\n' $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256))
}


# --- Parse Arguments ---
# -----------------------
VM_NAME=""
AUTOJOIN=false
NFS_SHARES=()
VOLUMES_RAW=""
MAIN_USER="docker"
MAIN_UID=""
MAIN_PASSWORD=""

while [[ $# -gt 0 ]]; do
    case "$1" in
        --autojoin)
            AUTOJOIN=true
            shift
            ;;
        --nfs)
            IFS=',' read -ra NFS_SHARES <<< "$2"
            shift 2
            ;;
        --volume)
            VOLUMES_RAW="$2"
            shift 2
            ;;
        --user)
            MAIN_USER="$2"
            shift 2
            ;;
        --uid)
            MAIN_UID="$2"
            shift 2
            ;;
        --password)
            MAIN_PASSWORD="$2"
            shift 2
            ;;
        *)
            if [[ -z "$VM_NAME" ]]; then
                VM_NAME="$1"
                shift
            else
                usage
            fi
            ;;
    esac
done

[[ -z "$VM_NAME" ]] && usage


# --- VMID Calculation & Prep ---
# -------------------------------
VMID_OFFSET=$(echo "$VM_NAME" | grep -o '[0-9]*' | tail -1)
[[ -z "$VMID_OFFSET" ]] && VMID_OFFSET=1
VMID=$((VMID_BASE + VMID_OFFSET))
MAC=$(generate_mac)

echo "Provisioning VM: $VM_NAME (VMID $VMID) with main user $MAIN_USER"

# Ensure snippets directory exists
mkdir -p "$SNIPPETS_DIR"


# --- Build Cloud-Init YAML ---
# -----------------------------
CLOUD_INIT_CONFIG="$SNIPPETS_DIR/${VM_NAME}-cloud-init.yaml"

cat > "$CLOUD_INIT_CONFIG" <<EOF
#cloud-config
users:
  - name: $MAIN_USER
EOF

if [[ -n "$MAIN_UID" ]]; then
    echo "    uid: $MAIN_UID" >> "$CLOUD_INIT_CONFIG"
fi

cat >> "$CLOUD_INIT_CONFIG" <<EOF
  sudo: ["ALL=(ALL) NOPASSWD:ALL"]
  shell: /bin/bash
  ssh_authorized_keys:
    - $(cat "$SSH_PUBLIC_KEY_PATH")
EOF

if [[ -n "$MAIN_PASSWORD" ]]; then
    ENCRYPTED_PASS=$(openssl passwd -6 "$MAIN_PASSWORD")
    echo "  passwd: $ENCRYPTED_PASS" >> "$CLOUD_INIT_CONFIG"
fi


# --- Build Post-Boot Commands ---
# --------------------------------
echo "runcmd:" >> "$CLOUD_INIT_CONFIG"

# Add user to Docker group
echo " - usermod -aG docker $MAIN_USER" >> "$CLOUD_INIT_CONFIG"

# NFS mounts
for SHARE in "${NFS_SHARES[@]}"; do
    MOUNT_POINT="/mnt/media/$SHARE"
    echo " - mkdir -p $MOUNT_POINT" >> "$CLOUD_INIT_CONFIG"
    echo " - chown $MAIN_USER:$MAIN_USER $MOUNT_POINT" >> "$CLOUD_INIT_CONFIG"
    echo " - echo '10.0.60.80:/mnt/docker-swarm/volumes/$SHARE $MOUNT_POINT nfs defaults 0 0' >> /etc/fstab" >> "$CLOUD_INIT_CONFIG"
    echo " - mount -a" >> "$CLOUD_INIT_CONFIG"
done

# SMB / Samba setup
if [[ -n "$NFS_SHARES" ]]; then
    echo " - apt-get update" >> "$CLOUD_INIT_CONFIG"
    echo " - apt-get install -y samba" >> "$CLOUD_INIT_CONFIG"
    for SHARE in "${NFS_SHARES[@]}"; do
        SMB_DIR="/mnt/media/$SHARE"
        echo " - mkdir -p $SMB_DIR" >> "$CLOUD_INIT_CONFIG"
        echo " - chown $MAIN_USER:$MAIN_USER $SMB_DIR" >> "$CLOUD_INIT_CONFIG"
        echo " - chmod 2775 $SMB_DIR" >> "$CLOUD_INIT_CONFIG"
        echo " - echo -e \"[$SHARE]\\npath=$SMB_DIR\\nbrowsable=yes\\nwritable=yes\\nguest ok=no\\nread only=no\\nforce user=$MAIN_USER\" >> /etc/samba/smb.conf" >> "$CLOUD_INIT_CONFIG"
    done
    echo " - systemctl enable smbd" >> "$CLOUD_INIT_CONFIG"
    echo " - systemctl restart smbd" >> "$CLOUD_INIT_CONFIG"
fi

# Volume creation
if [[ -n "$VOLUMES_RAW" ]]; then
    IFS=';' read -ra VOLUMES <<< "$VOLUMES_RAW"
    for VOL in "${VOLUMES[@]}"; do
        IFS=',' read -ra PARTS <<< "$VOL"
        VOL_NAME="${PARTS[0]}"
        VOL_SIZE="${PARTS[1]}"
        VOL_TYPE="${PARTS[2]}"
        
        if [[ "$VOL_TYPE" == "zfs" ]]; then
            echo " - zfs create -V $VOL_SIZE ${ZFS_CONFIG_POOL}/${VM_NAME}-${VOL_NAME}" >> "$CLOUD_INIT_CONFIG"
            echo " - mkdir -p /mnt/$VOL_NAME" >> "$CLOUD_INIT_CONFIG"
            echo " - mount -t zfs ${ZFS_CONFIG_POOL}/${VM_NAME}-${VOL_NAME} /mnt/$VOL_NAME" >> "$CLOUD_INIT_CONFIG"
        else
            echo " - mkdir -p /mnt/$VOL_NAME" >> "$CLOUD_INIT_CONFIG"
        fi
        echo " - chown $MAIN_USER:$MAIN_USER /mnt/$VOL_NAME" >> "$CLOUD_INIT_CONFIG"
        echo " - chmod 2775 /mnt/$VOL_NAME" >> "$CLOUD_INIT_CONFIG"
    done
fi

# Swarm autojoin
if $AUTOJOIN; then
    SWARM_TOKEN=$(cat "$SWARM_JOIN_TOKEN_FILE")
    echo " - docker swarm join --token $SWARM_TOKEN $SWARM_MANAGER_IP:2377" >> "$CLOUD_INIT_CONFIG"
fi

# Verify cloud-init file was created
if [[ ! -f "$CLOUD_INIT_CONFIG" ]]; then
    echo "ERROR: Failed to create cloud-init config file at $CLOUD_INIT_CONFIG"
    exit 1
fi

echo "Cloud-init config created at: $CLOUD_INIT_CONFIG"


# --- VM Creation & Configuration ---
# -----------------------------------
echo "Creating VM from template..."
qm clone $TEMPLATE_ID $VMID --name $VM_NAME --full

echo "Configuring VM..."
qm set $VMID --net0 virtio,bridge=$BRIDGE_NAME,macaddr=$MAC
qm set $VMID --ide2 $CLOUD_INIT_STORAGE:cloudinit
qm set $VMID --boot c --bootdisk scsi0
qm set $VMID --cores 2 --memory 4096

# Mixed storage approach: local for snippets, local-zfs for cloud-init disk
echo "Setting cloud-init configuration..."
qm set $VMID --cicustom "user=$SNIPPETS_STORAGE:snippets/${VM_NAME}-cloud-init.yaml"


# --- VM Created (but not started) ---
# ------------------------------------
echo "=================================="
echo "VM $VM_NAME provisioning complete!"
echo "=================================="
echo "VM ID: $VMID"
echo "VM Name: $VM_NAME"
echo "MAC Address: $MAC"
echo "=================================="
echo ""
echo "IMPORTANT: VM is created but NOT started"
echo ""
echo "Next steps:"
echo "1. Set static IP in pfSense for MAC: $MAC"
echo "2. Start the VM: qm start $VMID"
echo "3. Monitor cloud-init progress: qm monitor $VMID"
echo "4. Check VM status: qm status $VMID"