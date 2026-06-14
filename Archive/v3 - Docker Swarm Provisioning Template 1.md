---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
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


./provision-swarm-worker.sh --autojoin --user docker --password 'Plexface!' --nfs media/movies,media/tvshows,meda/animation --volume configs,100G,zfs  plex-01
  
## Script

#!/bin/bash
# ==========================================
# Docker Swarm Worker VM Provisioning Script
# ==========================================
# Features:
# - Deploy VM from template
# - Optional ZFS-backed volumes
# - Optional NFS mounts (dynamic paths)
# - Non-root user with sudo + docker group
# - Auto-SSH and Docker Swarm join
# - VLAN & IP handling
# - Cloud-init injection
# ------------------------------------------

# ---------- Config ----------
TEMPLATE_ID=101
VMID_BASE=200
BRIDGE_NAME="vmbr0"
DEFAULT_VLAN=85
STORAGE_LOCAL="local"          # Default storage for VM disk
NFS_SERVER="10.0.60.80"
NFS_BASE_PATH="/mnt/docker-swarm/volumes"  # NFS share base on server
LOCAL_MOUNT_BASE="/mnt"                     # Hardcoded local base path

# ---------- Usage ----------
usage() {
    echo "Usage: $0 --name <vmname> --user <username> --user-pass <password> [--vlan <vlan>] [--volume <name:size>] [--nfs <relpath1,relpath2>] [--autojoin]"
    exit 1
}

# ---------- Parse arguments ----------
while [[ $# -gt 0 ]]; do
    case "$1" in
        --name) VM_NAME="$2"; shift 2;;
        --vlan) VLAN="$2"; shift 2;;
        --volume) VOLUMES="$2"; shift 2;;       # Format: vol1:20G,vol2:10G
        --nfs) NFS_SHARES="$2"; shift 2;;       # Format: media/plex,media/movies
        --user) VM_USER="$2"; shift 2;;
        --user-pass) VM_USER_PASS="$2"; shift 2;;
        --autojoin) AUTOJOIN=1; shift;;
        *) echo "Unknown option $1"; usage;;
    esac
done

[[ -z "$VM_NAME" ]] && usage
[[ -z "$VM_USER" ]] && usage
[[ -z "$VM_USER_PASS" ]] && usage

VLAN=${VLAN:-$DEFAULT_VLAN}
VMID=$((VMID_BASE + RANDOM % 1000))   # Randomize VMID to avoid collision

echo "Provisioning VM '$VM_NAME' (ID $VMID) on VLAN $VLAN"

# ---------- Create VM from template ----------
echo "Creating VM from template $TEMPLATE_ID..."
qm clone $TEMPLATE_ID $VMID --name $VM_NAME --full true
qm set $VMID --net0 virtio,bridge=$BRIDGE_NAME,tag=$VLAN

# ---------- Add optional ZFS volumes ----------
if [[ -n "$VOLUMES" ]]; then
    IFS=',' read -ra VOL_ARR <<< "$VOLUMES"
    for vol in "${VOL_ARR[@]}"; do
        name="${vol%%:*}"
        size="${vol##*:}"
        echo "Creating ZFS volume $name ($size) for VM $VM_NAME..."
        qm set $VMID --scsi$(qm config $VMID | grep -c '^scsi'): zfs:$name,size=$size
    done
fi

# ---------- Generate Cloud-init user info ----------
CLOUD_USER_FILE="/tmp/${VM_NAME}-cloud-init.yaml"
USER_PASS_ENC=$(openssl passwd -6 "$VM_USER_PASS")

cat <<EOF > $CLOUD_USER_FILE
#cloud-config
users:
  - name: $VM_USER
    passwd: $USER_PASS_ENC
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: docker
    ssh-authorized-keys:
      - $(cat $HOME/.ssh/id_rsa.pub)
ssh_pwauth: True
EOF

echo "Non-root user '$VM_USER' created with sudo & docker group privileges."

qm set $VMID --ciuser $VM_USER --ipconfig0=ip=dhcp --cicustom "user=local:$CLOUD_USER_FILE"

# ---------- Optional NFS mounts ----------
if [[ -n "$NFS_SHARES" ]]; then
    IFS=',' read -ra NFS_ARR <<< "$NFS_SHARES"
    for share in "${NFS_ARR[@]}"; do
        TARGET="$LOCAL_MOUNT_BASE/$share"
        echo "Creating NFS mount point $TARGET..."
        mkdir -p "$TARGET"
        echo "Mounting NFS share $share to $TARGET..."
        echo "$NFS_SERVER:$NFS_BASE_PATH/$share $TARGET nfs defaults 0 0" >> /etc/fstab
    done
fi

# ---------- Final VM setup ----------
echo "VM $VM_NAME (ID $VMID) ready. Not starting automatically. You can reserve a static IP if needed."

# ---------- Optional auto-SSH and Swarm join ----------
if [[ -n "$AUTOJOIN" ]]; then
    echo "Auto-joining Docker Swarm cluster..."
    read -p "Start VM manually and press Enter once SSH is available..."
    VM_IP=$(qm guest cmd $VMID network-get-interfaces | grep inet | awk '{print $2}' | cut -d/ -f1)
    ssh -o StrictHostKeyChecking=no $VM_USER@$VM_IP "docker swarm join --token <SWARM_TOKEN> <MANAGER_IP>:2377"
    echo "VM $VM_NAME joined Swarm cluster."
fi

echo "Provisioning complete!"
