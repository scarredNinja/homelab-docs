---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
#HomeLabRebuild/Virtualisation #HomeLabRebuild/SwarmManager


#!/bin/bash
# ==============================================================================
# VM Provisioning Script for Proxmox (Updated for Swarm scripts & NFS only)
# ==============================================================================

# --- Configuration ---
TEMPLATE_ID=101
VMID_BASE=200
BRIDGE_NAME="vmbr0"
DEFAULT_VLAN=85
SSH_PUBLIC_KEY_PATH="/root/.ssh/id_rsa.pub"
SWARM_MANAGER_IP="10.0.60.30"
SWARM_SCRIPTS_DIR="/root/provisioning-scripts/swarm"
ZFS_CONFIG_POOL="rspool/vm-configs"
ZFS_DB_POOL="rspool/vm-db"

SNIPPETS_STORAGE="local"
CLOUD_INIT_STORAGE="local-zfs"
SNIPPETS_DIR="/var/lib/vz/snippets"

# --- Functions ---
usage() {
    echo "Usage: $0 [OPTIONS] <vm-name>"
    echo ""
    echo "Options:"
    echo "  --template-id id        Template ID (default: $TEMPLATE_ID)"
    echo "  --role role             Swarm role: manager or worker"
    echo "  --vlan id               VLAN tag"
    echo "  --nfs share1,share2     Comma-separated NFS shares to mount"
    echo "  --volume volname,size,type;  Volumes to create (name,size,type)"
    echo "  --user name             Main VM user (default: docker)"
    echo "  --uid uid               Optional UID for main user"
    echo "  --password pass         Optional password for main user"
    exit 1
}

generate_mac() {
    printf '52:54:%02x:%02x:%02x:%02x\n' $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256))
}

# --- Parse Arguments ---
VM_NAME=""
VLAN_ID=""
SWARM_ROLE=""
NFS_SHARES=()
VOLUMES_RAW=""
MAIN_USER="docker"
MAIN_UID=""
MAIN_PASSWORD=""

while [[ $# -gt 0 ]]; do
    case "$1" in
        --template-id) TEMPLATE_ID="$2"; shift 2 ;;
        --role) SWARM_ROLE="$2"; shift 2 ;;
        --vlan) VLAN_ID="$2"; shift 2 ;;
        --nfs) IFS=',' read -ra NFS_SHARES <<< "$2"; shift 2 ;;
        --volume) VOLUMES_RAW="$2"; shift 2 ;;
        --user) MAIN_USER="$2"; shift 2 ;;
        --uid) MAIN_UID="$2"; shift 2 ;;
        --password) MAIN_PASSWORD="$2"; shift 2 ;;
        -h|--help) usage ;;
        *) 
            if [[ -z "$VM_NAME" ]]; then VM_NAME="$1"; shift
            else echo "Unknown argument $1"; usage
            fi
            ;;
    esac
done

[[ -z "$VM_NAME" ]] && usage

# --- VMID & MAC ---
VMID_OFFSET=$(echo "$VM_NAME" | grep -o '[0-9]*' | tail -1)
[[ -z "$VMID_OFFSET" ]] && VMID_OFFSET=1
VMID=$((VMID_BASE + VMID_OFFSET))
MAC=$(generate_mac)

echo "Provisioning VM: $VM_NAME (VMID $VMID) with user $MAIN_USER"
mkdir -p "$SNIPPETS_DIR"

# --- Clone Template ---
qm clone $TEMPLATE_ID $VMID --name $VM_NAME --full
if [[ -n "$VLAN_ID" ]]; then
    qm set $VMID --net0 virtio,bridge=$BRIDGE_NAME,tag=$VLAN_ID,macaddr=$MAC
else
    qm set $VMID --net0 virtio,bridge=$BRIDGE_NAME,macaddr=$MAC
fi
qm set $VMID --ide2 $CLOUD_INIT_STORAGE:cloudinit
qm set $VMID --boot c --bootdisk scsi0
qm set $VMID --cores 2 --memory 4096

# --- Handle Volumes ---
SCSI_COUNTER=1
CLOUD_INIT_VOLUME_SETUP=""

if [[ -n "$VOLUMES_RAW" ]]; then
    IFS=';' read -ra VOLUMES <<< "$VOLUMES_RAW"
    for VOL in "${VOLUMES[@]}"; do
        IFS=',' read -ra PARTS <<< "$VOL"
        VOL_NAME="${PARTS[0]}"
        VOL_SIZE="${PARTS[1]}"
        VOL_TYPE="${PARTS[2]}"
        if [[ "$VOL_TYPE" == "zfs" ]]; then
            ZFS_VOL_NAME="${ZFS_CONFIG_POOL}/${VM_NAME}-${VOL_NAME}"
            [[ -z $(zfs list -H "$ZFS_VOL_NAME" 2>/dev/null) ]] && zfs create -V "$VOL_SIZE" "$ZFS_VOL_NAME"
            qm set "$VMID" --scsi${SCSI_COUNTER} "${CLOUD_INIT_STORAGE}:${ZFS_VOL_NAME}"
            CLOUD_INIT_VOLUME_SETUP+=" - mkdir -p /mnt/$VOL_NAME\n - chown $MAIN_USER:$MAIN_USER /mnt/$VOL_NAME"
            SCSI_COUNTER=$((SCSI_COUNTER + 1))
        else
            CLOUD_INIT_VOLUME_SETUP+=" - mkdir -p /mnt/$VOL_NAME\n - chown $MAIN_USER:$MAIN_USER /mnt/$VOL_NAME"
        fi
    done
fi

# --- Cloud-Init YAML ---
CLOUD_INIT_CONFIG="$SNIPPETS_DIR/${VM_NAME}-cloud-init.yaml"
cat > "$CLOUD_INIT_CONFIG" <<EOF
#cloud-config
users:
  - name: $MAIN_USER
EOF
[[ -n "$MAIN_UID" ]] && echo "    uid: $MAIN_UID" >> "$CLOUD_INIT_CONFIG"
cat >> "$CLOUD_INIT_CONFIG" <<EOF
  sudo: ["ALL=(ALL) NOPASSWD:ALL"]
  shell: /bin/bash
  ssh_authorized_keys:
    - $(cat "$SSH_PUBLIC_KEY_PATH")
EOF
[[ -n "$MAIN_PASSWORD" ]] && echo "    passwd: $(openssl passwd -6 "$MAIN_PASSWORD")" >> "$CLOUD_INIT_CONFIG"

# --- Post-Boot Commands ---
echo "runcmd:" >> "$CLOUD_INIT_CONFIG"
echo " - usermod -aG docker $MAIN_USER" >> "$CLOUD_INIT_CONFIG"

# Volume setup
IFS=$'\n'
for CMD in $CLOUD_INIT_VOLUME_SETUP; do
    echo " - $CMD" >> "$CLOUD_INIT_CONFIG"
done

# NFS mounts
for SHARE in "${NFS_SHARES[@]}"; do
    MOUNT_POINT="/mnt/media/$SHARE"
    echo " - mkdir -p $MOUNT_POINT" >> "$CLOUD_INIT_CONFIG"
    echo " - chown $MAIN_USER:$MAIN_USER $MOUNT_POINT" >> "$CLOUD_INIT_CONFIG"
    echo " - echo '10.0.60.80:/mnt/docker-swarm/volumes/$SHARE $MOUNT_POINT nfs defaults 0 0' >> /etc/fstab" >> "$CLOUD_INIT_CONFIG"
    echo " - mount -a" >> "$CLOUD_INIT_CONFIG"
done

# --- Swarm Setup via helper scripts ---
if [[ "$SWARM_ROLE" == "worker" ]]; then
    echo " - bash $SWARM_SCRIPTS_DIR/join-worker.sh $SWARM_MANAGER_IP $(cat /root/swarm_worker_token.txt)" >> "$CLOUD_INIT_CONFIG"
elif [[ "$SWARM_ROLE" == "manager" ]]; then
    # Auto-detect if first manager
    CHECK_MANAGER=$(ssh root@$SWARM_MANAGER_IP "docker info >/dev/null 2>&1 && echo exists" || echo none)
    if [[ "$CHECK_MANAGER" == "none" ]]; then
        echo " - echo 'No existing manager found. Initializing first manager.'" >> "$CLOUD_INIT_CONFIG"
        echo " - bash $SWARM_SCRIPTS_DIR/swarm-init.sh $SWARM_MANAGER_IP" >> "$CLOUD_INIT_CONFIG"
    else
        echo " - echo 'Existing manager detected. Joining as additional manager.'" >> "$CLOUD_INIT_CONFIG"
        echo " - bash $SWARM_SCRIPTS_DIR/join-manager.sh $SWARM_MANAGER_IP $(cat /root/swarm_manager_token.txt)" >> "$CLOUD_INIT_CONFIG"
    fi
fi

# --- Final Output ---
[[ ! -f "$CLOUD_INIT_CONFIG" ]] && { echo "ERROR: Cloud-init config not created"; exit 1; }
echo "Cloud-init config created at: $CLOUD_INIT_CONFIG"

echo "=================================="
echo "VM $VM_NAME provisioning complete!"
echo "VM ID: $VMID"
echo "MAC Address: $MAC"
echo "IMPORTANT: VM is created but NOT started"
echo "Next steps:"
echo "1. Set static IP in pfSense for MAC: $MAC"
echo "2. Start the VM: qm start $VMID"
echo "3. Monitor cloud-init progress: qm monitor $VMID"
echo "4. Check VM status: qm status $VMID"
