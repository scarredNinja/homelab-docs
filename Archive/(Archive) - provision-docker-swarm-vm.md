
#HomeLabRebuild/Virtualisation #HomeLabRebuild/SwarmManager



#!/bin/bash
# ==============================================================================
# VM Provisioning Script for Proxmox
# ==============================================================================

# --- Configuration ---
# ---------------------
TEMPLATE_ID=101 # Default template ID, can be overridden by --template-id
VMID_BASE=200
BRIDGE_NAME="vmbr0"
DEFAULT_VLAN=85
SSH_PUBLIC_KEY_PATH="/root/.ssh/id_rsa.pub"
SWARM_JOIN_TOKEN_FILE="/root/swarm_worker_token.txt"
SWARM_MANAGER_TOKEN_FILE="/root/swarm_manager_token.txt"
SWARM_MANAGER_IP="10.0.60.30"
ZFS_CONFIG_POOL="rspool/vm-configs"
ZFS_DB_POOL="rspool/vm-db"

# Storage configuration
SNIPPETS_STORAGE="local"
CLOUD_INIT_STORAGE="local-zfs"
SNIPPETS_DIR="/var/lib/vz/snippets"


# --- Functions ---
# -----------------
usage() {
    echo "Usage: $0 [OPTIONS] <vm-name>"
    echo ""
    echo "This script provisions a new VM from a template with user-defined cloud-init settings."
    echo ""
    echo "Options:"
    echo "  --template-id id             The Proxmox template ID to clone from (default: $TEMPLATE_ID)."
    echo "  --role role                  Specifies the Docker Swarm role for the VM. Can be 'manager' or 'worker'."
    echo "  --vlan id                    Optional VLAN tag for the VM's network interface."
    echo "  --nfs share1,share2,...      Comma-separated list of NFS shares to mount at /mnt/media/<share>."
    echo "  --volume volname,size,type;  Semi-colon separated list of volumes to create."
    echo "                             Each volume is name,size,type, where type can be 'zfs' or 'dir'."
    echo "                             Example: 'db,20G,zfs;docker,50G,zfs;docker-config,5G,zfs;cache,5G,dir'."
    echo "  --user name                  Name of the main user inside the VM (default: 'docker')."
    echo "  --uid uid                    Optional UID for the main user."
    echo "  --password pass              Optional password for the main user."
    echo ""
    echo "Arguments:"
    echo "  <vm-name>                    The name of the VM to create."
    exit 1
}

generate_mac() {
    printf '52:54:%02x:%02x:%02x:%02x\n' $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256))
}


# --- Parse Arguments ---
# -----------------------
VM_NAME=""
VLAN_ID=""
SWARM_ROLE=""
NFS_SHARES=()
VOLUMES_RAW=""
MAIN_USER="docker"
MAIN_UID=""
MAIN_PASSWORD=""
ZFS_DOCKER_VOLUME_EXISTS=0

while [[ $# -gt 0 ]]; do
    case "$1" in
        --template-id)
            TEMPLATE_ID="$2"
            shift 2
            ;;
        --role)
            SWARM_ROLE="$2"
            shift 2
            ;;
        --vlan)
            VLAN_ID="$2"
            shift 2
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
        -h|--help)
            usage
            ;;
        *)
            if [[ -z "$VM_NAME" ]]; then
                VM_NAME="$1"
                shift
            else
                echo "Error: Unknown argument '$1'"
                usage
            fi
            ;;
    esac
done

[[ -z "$VM_NAME" ]] && usage


# --- VMID Calculation & Prep ---
VMID_OFFSET=$(echo "$VM_NAME" | grep -o '[0-9]*' | tail -1)
[[ -z "$VMID_OFFSET" ]] && VMID_OFFSET=1
VMID=$((VMID_BASE + VMID_OFFSET))
MAC=$(generate_mac)

echo "Provisioning VM: $VM_NAME (VMID $VMID) with main user $MAIN_USER"

# Ensure snippets directory exists
mkdir -p "$SNIPPETS_DIR"

# --- Pre-VM Creation: Create VM and attach disks ---
# ---------------------------------------------------
echo "Creating VM from template..."
qm clone $TEMPLATE_ID $VMID --name $VM_NAME --full

echo "Configuring VM..."
if [[ -n "$VLAN_ID" ]]; then
    qm set $VMID --net0 virtio,bridge=$BRIDGE_NAME,tag=$VLAN_ID,macaddr=$MAC
else
    qm set $VMID --net0 virtio,bridge=$BRIDGE_NAME,macaddr=$MAC
fi
qm set $VMID --ide2 $CLOUD_INIT_STORAGE:cloudinit
qm set $VMID --boot c --bootdisk scsi0
qm set $VMID --cores 2 --memory 4096

SCSI_COUNTER=1
CLOUD_INIT_VOLUME_SETUP=""
SMB_VOLUME_CONFIG=""

if [[ -n "$VOLUMES_RAW" ]]; then
    IFS=';' read -ra VOLUMES <<< "$VOLUMES_RAW"
    for VOL in "${VOLUMES[@]}"; do
        IFS=',' read -ra PARTS <<< "$VOL"
        VOL_NAME="${PARTS[0]}"
        VOL_SIZE="${PARTS[1]}"
        VOL_TYPE="${PARTS[2]}"

        # Create ZFS volume on host and attach to VM
        if [[ "$VOL_TYPE" == "zfs" ]]; then
            ZFS_VOL_NAME="${ZFS_CONFIG_POOL}/${VM_NAME}-${VOL_NAME}"
            echo "Creating ZFS volume on host: $ZFS_VOL_NAME of size $VOL_SIZE"
            
            # Check if dataset already exists
            if zfs list -H "$ZFS_VOL_NAME" > /dev/null 2>&1; then
                echo "Error: ZFS dataset $ZFS_VOL_NAME already exists. Skipping creation."
                exit 1
            fi
            
            zfs create -V "$VOL_SIZE" "$ZFS_VOL_NAME"
            qm set "$VMID" --scsi${SCSI_COUNTER} "${CLOUD_INIT_STORAGE}:${ZFS_VOL_NAME}"
            
            # Add mount instruction to Cloud-Init
            if [[ "$VOL_NAME" == "docker" ]]; then
                # Special case: configure Docker root and mount to /var/lib/docker
                CLOUD_INIT_VOLUME_SETUP+=" - [ 'sh', '-c', 'mkfs.ext4 /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_scsi${SCSI_COUNTER} && echo UUID=$(blkid -s UUID -o value /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_scsi${SCSI_COUNTER}) /var/lib/docker ext4 defaults 0 0 >> /etc/fstab' ]"
                CLOUD_INIT_VOLUME_SETUP+=" - mount -a"
                SMB_VOLUME_CONFIG+="\\n[docker]\\npath=/var/lib/docker\\nbrowsable=no\\nwritable=yes\\nguest ok=no\\nread only=no\\nforce user=$MAIN_USER\\ncreate mask=0664\\ndirectory mask=0775"
            elif [[ "$VOL_NAME" == "docker-config" ]]; then
                # Special case: configure Docker config directory
                CLOUD_INIT_VOLUME_SETUP+=" - [ 'sh', '-c', 'mkfs.ext4 /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_scsi${SCSI_COUNTER} && echo UUID=$(blkid -s UUID -o value /dev/disk/by-0QEMU_QEMU_HARDDISK_scsi${SCSI_COUNTER}) /etc/docker-new ext4 defaults 0 0 >> /etc/fstab' ]"
                CLOUD_INIT_VOLUME_SETUP+=" - mount -a"
                CLOUD_INIT_VOLUME_SETUP+=" - rsync -a /etc/docker/ /etc/docker-new/"
                CLOUD_INIT_VOLUME_SETUP+=" - rm -rf /etc/docker"
                CLOUD_INIT_VOLUME_SETUP+=" - ln -s /etc/docker-new /etc/docker"
                SMB_VOLUME_CONFIG+="\\n[docker-config]\\npath=/etc/docker-new\\nbrowsable=yes\\nwritable=yes\\nguest ok=no\\nread only=no\\nforce user=$MAIN_USER\\ncreate mask=0664\\ndirectory mask=0775"
            else
                # Standard case: create mount point and mount
                CLOUD_INIT_VOLUME_SETUP+=" - mkdir -p /mnt/$VOL_NAME"
                CLOUD_INIT_VOLUME_SETUP+=" - [ 'sh', '-c', 'mkfs.ext4 /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_scsi${SCSI_COUNTER} && echo UUID=$(blkid -s UUID -o value /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_scsi${SCSI_COUNTER}) /mnt/$VOL_NAME ext4 defaults 0 0 >> /etc/fstab' ]"
                CLOUD_INIT_VOLUME_SETUP+=" - mount -a"
                CLOUD_INIT_VOLUME_SETUP+=" - chown $MAIN_USER:$MAIN_USER /mnt/$VOL_NAME"
                CLOUD_INIT_VOLUME_SETUP+=" - chmod 2775 /mnt/$VOL_NAME"
                SMB_VOLUME_CONFIG+="\\n[$VOL_NAME]\\npath=/mnt/$VOL_NAME\\nbrowsable=yes\\nwritable=yes\\nguest ok=no\\nread only=no\\nforce user=$MAIN_USER\\ncreate mask=0664\\ndirectory mask=0775"
            fi
            
            SCSI_COUNTER=$((SCSI_COUNTER + 1))
        fi
    done
fi


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
    echo "    passwd: $ENCRYPTED_PASS" >> "$CLOUD_INIT_CONFIG"
fi


# --- Build Post-Boot Commands ---
echo "runcmd:" >> "$CLOUD_INIT_CONFIG"

# Add user to Docker group
echo " - usermod -aG docker $MAIN_USER" >> "$CLOUD_INIT_CONFIG"

# Volume setup from pre-creation step
echo "$CLOUD_INIT_VOLUME_SETUP" >> "$CLOUD_INIT_CONFIG"

# NFS mounts
for SHARE in "${NFS_SHARES[@]}"; do
    MOUNT_POINT="/mnt/media/$SHARE"
    echo " - mkdir -p $MOUNT_POINT" >> "$CLOUD_INIT_CONFIG"
    echo " - chown $MAIN_USER:$MAIN_USER $MOUNT_POINT" >> "$CLOUD_INIT_CONFIG"
    echo " - echo '10.0.60.80:/mnt/docker-swarm/volumes/$SHARE $MOUNT_POINT nfs defaults 0 0' >> /etc/fstab" >> "$CLOUD_INIT_CONFIG"
    echo " - mount -a" >> "$CLOUD_INIT_CONFIG"
done

# SMB / Samba setup
# Assuming samba is pre-installed on the template
if [[ -n "$MAIN_PASSWORD" ]]; then
    echo " - echo -e \"${MAIN_PASSWORD}\n${MAIN_PASSWORD}\" | smbpasswd -a -s $MAIN_USER" >> "$CLOUD_INIT_CONFIG"
fi
if [[ ${#NFS_SHARES[@]} -gt 0 || -n "$VOLUMES_RAW" ]]; then
    echo " - echo \"[global]\\nworkgroup = WORKGROUP\\nserver string = $VM_NAME\\nsecurity = user\\nmap to guest = bad user\\ndns proxy = no\\n\" >> /etc/samba/smb.conf" >> "$CLOUD_INIT_CONFIG"
    for SHARE in "${NFS_SHARES[@]}"; do
        SMB_DIR="/mnt/media/$SHARE"
        echo " - echo -e \"[$SHARE]\\npath=$SMB_DIR\\nbrowsable=yes\\nwritable=yes\\nguest ok=no\\nread only=no\\nforce user=$MAIN_USER\\ncreate mask=0664\\ndirectory mask=0775\" >> /etc/samba/smb.conf" >> "$CLOUD_INIT_CONFIG"
    done
    echo " - echo -e \"$SMB_VOLUME_CONFIG\" >> /etc/samba/smb.conf" >> "$CLOUD_INIT_CONFIG"
    echo " - systemctl reload smbd" >> "$CLOUD_INIT_CONFIG"
fi

# Swarm role configuration
if [[ "$SWARM_ROLE" == "worker" ]]; then
    if [[ ! -f "$SWARM_JOIN_TOKEN_FILE" ]]; then
        echo "ERROR: Docker Swarm worker join token file not found at $SWARM_JOIN_TOKEN_FILE"
        exit 1
    fi
    SWARM_TOKEN=$(cat "$SWARM_JOIN_TOKEN_FILE")
    echo " - docker swarm join --token $SWARM_TOKEN $SWARM_MANAGER_IP:2377" >> "$CLOUD_INIT_CONFIG"
elif [[ "$SWARM_ROLE" == "manager" ]]; then
    if [[ ! -f "$SWARM_MANAGER_TOKEN_FILE" ]]; then
        echo " - docker swarm init --advertise-addr $SWARM_MANAGER_IP" >> "$CLOUD_INIT_CONFIG"
        echo " - docker swarm join-token worker > /root/swarm_worker_token.txt" >> "$CLOUD_INIT_CONFIG"
        echo " - docker swarm join-token manager > /root/swarm_manager_token.txt" >> "$CLOUD_INIT_CONFIG"
    else
        SWARM_MANAGER_TOKEN=$(cat "$SWARM_MANAGER_TOKEN_FILE")
        echo " - docker swarm join --token $SWARM_MANAGER_TOKEN --manager $SWARM_MANAGER_IP:2377" >> "$CLOUD_INIT_CONFIG"
    fi
fi

# Verify cloud-init file was created
if [[ ! -f "$CLOUD_INIT_CONFIG" ]]; then
    echo "ERROR: Failed to create cloud-init config file at $CLOUD_INIT_CONFIG"
    exit 1
fi

echo "Cloud-init config created at: $CLOUD_INIT_CONFIG"


# --- Finalize VM Creation ---
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