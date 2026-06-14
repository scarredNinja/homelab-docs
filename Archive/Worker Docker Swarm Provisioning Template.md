---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
./provision-swarm-worker-vlan85.sh plex-worker --autojoin --volumes plex-config,plex-data --nfs-shares 10.0.60.80:/volume1/movies,10.0.60.80:/volume1/tvshows, 10.0.60.80:/volume1/animation

#!/bin/bash

# --- Config ---
TEMPLATE_ID=103
VMID_BASE=200
BRIDGE_NAME="vmbr0"
VLAN_TAG=85
NODE_NAME=$(hostname)
STORAGE="local-lvm"
CI_USER="root"
CI_PASSWORD="changeme"
SSH_PUBLIC_KEY_PATH="$HOME/.ssh/id_rsa.pub"
SWARM_JOIN_TOKEN_FILE="/root/swarm_worker_token.txt"
SWARM_MANAGER_IP="10.0.60.30"
NFS_SERVER_IP="10.0.60.80"

# --- Functions ---
usage() {
    echo "Usage: $0 <vm-name> [--autojoin] [--volume <volume-name,size>] [--nfs share1,share2,...]"
    exit 1
}

generate_mac() {
    echo "02:$(openssl rand -hex 5 | sed 's/\(..\)/\1:/g; s/:$//')"
}

# --- Parse Arguments ---
VM_NAME=""
AUTOJOIN=false
VOLUME=""
VOLUME_SIZE="10G"
NFS_SHARES=()

while [[ $# -gt 0 ]]; do
    case "$1" in
        --autojoin)
            AUTOJOIN=true
            shift
            ;;
        --volume)
            # Support multiple volumes as comma separated: vol1,size1;vol2,size2
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
# Extract trailing digits from vm name for VMID offset (default fallback to 1)
VMID_OFFSET=$(echo "$VM_NAME" | grep -o '[0-9]\+' | tail -1)
if [[ -z "$VMID_OFFSET" ]]; then
    VMID_OFFSET=1
fi
VMID=$((VMID_BASE + VMID_OFFSET))

# --- Clone Template ---
echo "[+] Cloning template $TEMPLATE_ID to VM $VMID ($VM_NAME)..."
qm clone $TEMPLATE_ID $VMID --name "$VM_NAME" --full --storage $STORAGE || exit 1

# --- Set Network ---
MAC=$(generate_mac)
qm set $VMID \
    --net0 virtio=${MAC},bridge=${BRIDGE_NAME},tag=${VLAN_TAG} \
    --ciuser "$CI_USER" \
    --cipassword "$CI_PASSWORD" \
    --ipconfig0 ip=dhcp

# --- SSH Key Injection ---
if [[ -f "$SSH_PUBLIC_KEY_PATH" ]]; then
    qm set $VMID --sshkey "$SSH_PUBLIC_KEY_PATH"
    echo "[+] SSH public key injected"
else
    echo "[!] SSH key not found at $SSH_PUBLIC_KEY_PATH, skipping"
fi

# --- Cloud-init user-data for swarm join and NFS mounts ---
USERDATA_FILE="/tmp/userdata-${VMID}.yaml"

echo "#cloud-config" > "$USERDATA_FILE"
echo "runcmd:" >> "$USERDATA_FILE"

# Swarm join command if requested
if $AUTOJOIN; then
    if [[ ! -f "$SWARM_JOIN_TOKEN_FILE" ]]; then
        echo "[!] Swarm join token file not found: $SWARM_JOIN_TOKEN_FILE"
        exit 1
    fi
    JOIN_TOKEN=$(cat "$SWARM_JOIN_TOKEN_FILE")
    echo "  - docker swarm join --token $JOIN_TOKEN $SWARM_MANAGER_IP:2377" >> "$USERDATA_FILE"
    echo "[+] Added swarm join command to cloud-init"
fi

# NFS mounting commands if any shares specified
if [[ ${#NFS_SHARES[@]} -gt 0 ]]; then
    echo "  - apt-get update && apt-get install -y nfs-common" >> "$USERDATA_FILE"
    for share in "${NFS_SHARES[@]}"; do
        echo "  - mkdir -p /mnt/media/${share}" >> "$USERDATA_FILE"
        echo "  - echo '${NFS_SERVER_IP}:/mnt/docker-swarm/volumes/${share} /mnt/media/${share} nfs defaults 0 0' >> /etc/fstab" >> "$USERDATA_FILE"
        echo "  - mount /mnt/media/${share}" >> "$USERDATA_FILE"
        echo "  - mkdir -p /var/lib/docker/volumes/${share}/_data" >> "$USERDATA_FILE"
        echo "  - ln -sf /mnt/media/${share} /var/lib/docker/volumes/${share}/_data" >> "$USERDATA_FILE"
    done
    echo "[+] Added NFS shares to cloud-init: ${NFS_SHARES[*]}"
fi

qm set $VMID --cicustom "user=local:snippets/userdata-${VMID}.yaml"

# --- Handle Proxmox virtual disk volumes if provided ---
if [[ -n "$VOLUMES_RAW" ]]; then
    # Expected format: vol1,size1;vol2,size2;...
    IFS=';' read -ra VOLUMES_ARR <<< "$VOLUMES_RAW"
    for voldef in "${VOLUMES_ARR[@]}"; do
        volname=$(echo "$voldef" | cut -d',' -f1)
        volsize=$(echo "$voldef" | cut -d',' -f2)
        if [[ -z "$volsize" ]]; then
            volsize="10G"
        fi
        echo "[+] Creating Proxmox volume: $volname size $volsize"
        qm disk add $VMID "${STORAGE}:${volname},size=${volsize}" --scsi0
    done
fi

# --- Output Info ---
echo "[✓] VM $VM_NAME (VMID $VMID) provisioned on VLAN $VLAN_TAG"
echo "    MAC Address: $MAC"
echo "    DHCP reservation recommended before starting the VM."

# --- Show config ---
qm config $VMID
