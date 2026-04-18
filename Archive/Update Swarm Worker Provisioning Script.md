./provision_swarm_vm.sh <vm-name> --autojoin --volume configs,20G,zfs --nfs media


./provision-swarm-vm.sh  plex-01 \
  --autojoin \
  --volume plex-config,100G,zfs \
  --nfs media/TVShows,media/Movies,media/Animation






#!/bin/bash
# Docker Swarm Worker VM Provisioning with optional ZFS volumes and generic NFS mounts

# --- Config ---
TEMPLATE_ID=101
VMID_BASE=200
BRIDGE_NAME="vmbr0"
DEFAULT_VLAN=85
NODE_NAME=$(hostname)
CI_USER="root"
CI_PASSWORD="changeme"
SSH_PUBLIC_KEY_PATH="$HOME/.ssh/id_rsa.pub"
SWARM_JOIN_TOKEN_FILE="/root/swarm_worker_token.txt"
SWARM_MANAGER_IP="10.0.60.30"
NFS_SERVER_IP="10.0.60.80"

# Optional ZFS pools
ZFS_CONFIG_POOL="rspool/vm-configs"
ZFS_DB_POOL="rspool/vm-db"

# --- Functions ---
usage() {
    echo "Usage: $0 <vm-name> [--autojoin] [--volume <volname,size,type>] [--nfs share1,share2,...]"
    exit 1
}

generate_mac() {
    echo "02:$(openssl rand -hex 5 | sed 's/\(..\)/\1:/g; s/:$//')"
}

# --- Parse Arguments ---
VM_NAME=""
AUTOJOIN=false
VOLUMES_RAW=""
NFS_SHARES=()

while [[ $# -gt 0 ]]; do
    case "$1" in
        --autojoin)
            AUTOJOIN=true
            shift
            ;;
        --volume)
            VOLUMES_RAW="$2"
            shift 2
            ;;
        --nfs)
            IFS=',' read -r -a NFS_SHARES <<< "$2"
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

# --- VMID Calculation ---
VMID_OFFSET=$(echo "$VM_NAME" | grep -o '[0-9]\+' | tail -1)
if [[ -z "$VMID_OFFSET" ]]; then VMID_OFFSET=1; fi
VMID=$((VMID_BASE + VMID_OFFSET))

# --- Clone Template ---
echo "[+] Cloning template $TEMPLATE_ID to VM $VMID ($VM_NAME)..."
qm clone $TEMPLATE_ID $VMID --name "$VM_NAME" --full --storage local-zfs || exit 1

# --- Network Setup ---
MAC=$(generate_mac)
qm set $VMID \
    --net0 virtio=${MAC},bridge=${BRIDGE_NAME},tag=${DEFAULT_VLAN} \
    --ciuser "$CI_USER" \
    --cipassword "$CI_PASSWORD" \
    --ipconfig0 ip=dhcp

# --- SSH Key Injection ---
if [[ -f "$SSH_PUBLIC_KEY_PATH" ]]; then
    qm set $VMID --sshkey "$SSH_PUBLIC_KEY_PATH"
    echo "[+] SSH key injected"
else
    echo "[!] SSH key not found at $SSH_PUBLIC_KEY_PATH, skipping"
fi

# --- Cloud-init user-data ---
USERDATA_FILE="/tmp/userdata-${VMID}.yaml"
echo "#cloud-config" > "$USERDATA_FILE"
echo "runcmd:" >> "$USERDATA_FILE"

# Swarm join
if $AUTOJOIN; then
    if [[ ! -f "$SWARM_JOIN_TOKEN_FILE" ]]; then
        echo "[!] Swarm join token file not found"
        exit 1
    fi
    JOIN_TOKEN=$(cat "$SWARM_JOIN_TOKEN_FILE")
    echo "  - docker swarm join --token $JOIN_TOKEN $SWARM_MANAGER_IP:2377" >> "$USERDATA_FILE"
    echo "[+] Added swarm join command"
fi

# NFS mounts (generic)
if [[ ${#NFS_SHARES[@]} -gt 0 ]]; then
    echo "  - apt-get update && apt-get install -y nfs-common" >> "$USERDATA_FILE"
    for share in "${NFS_SHARES[@]}"; do
        echo "  - mkdir -p /mnt/${share}" >> "$USERDATA_FILE"
        echo "  - echo '${NFS_SERVER_IP}:/mnt/docker-swarm/volumes/${share} /mnt/${share} nfs defaults 0 0' >> /etc/fstab" >> "$USERDATA_FILE"
        echo "  - mount /mnt/${share}" >> "$USERDATA_FILE"
        echo "  - mkdir -p /var/lib/docker/volumes/${share}/_data" >> "$USERDATA_FILE"
        echo "  - ln -sf /mnt/${share} /var/lib/docker/volumes/${share}/_data" >> "$USERDATA_FILE"
    done
    echo "[+] NFS shares added to cloud-init: ${NFS_SHARES[*]}"
fi

qm set $VMID --cicustom "user=local:snippets/userdata-${VMID}.yaml"

# --- Optional Local ZFS Volumes ---
if [[ -n "$VOLUMES_RAW" ]]; then
    IFS=';' read -ra VOLUMES_ARR <<< "$VOLUMES_RAW"
    for voldef in "${VOLUMES_ARR[@]}"; do
        volname=$(echo "$voldef" | cut -d',' -f1)
        volsize=$(echo "$voldef" | cut -d',' -f2)
        voltype=$(echo "$voldef" | cut -d',' -f3) # zfs or nfs
        if [[ -z "$volsize" ]]; then volsize="10G"; fi
        if [[ "$voltype" == "zfs" ]]; then
            echo "[+] Creating ZFS disk $volname ($volsize)"
            qm disk add $VMID "${ZFS_CONFIG_POOL}:${volname},size=${volsize}" --scsi0
            echo "  - mkdir -p /mnt/volumes/${volname}" >> "$USERDATA_FILE"
            echo "  - ln -sf /mnt/volumes/${volname} /var/lib/docker/volumes/${volname}/_data" >> "$USERDATA_FILE"
        else
            echo "[!] Skipping unknown volume type or handled separately"
        fi
    done
fi

# --- Finished ---
echo "[✓] VM $VM_NAME (VMID $VMID) provisioned."
echo "    MAC Address: $MAC"
echo "    DHCP recommended before starting."
qm config $VMID

