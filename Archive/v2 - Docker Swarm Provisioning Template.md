#HomeLabRebuild/Swarm #HomeLabRebuild/Automation
# Docker Swarm Worker VM Provisioning (NFS + multi ZFS + autojoin)

## Usage

./provision-swarm-worker.sh --autojoin \
  --nfs movies,tvshows, animation \
  --volume configs,100G,zfs; \
  plex-01
## Script

#!/bin/bash
# --- Config ---
TEMPLATE_ID=101
VMID_BASE=200
BRIDGE_NAME="vmbr0"
DEFAULT_VLAN=85
SSH_PUBLIC_KEY_PATH="/root/.ssh/id_rsa.pub"
SWARM_JOIN_TOKEN_FILE="/root/swarm_worker_token.txt"
SWARM_MANAGER_IP="10.0.60.30"
NFS_SERVER_IP="10.0.60.80"

# ZFS pools
ZFS_CONFIG_POOL="rspool/vm-configs"
ZFS_DB_POOL="rspool/vm-db"   # reserved for DBs if needed

# --- Functions ---
usage() {
  echo "Usage: $0 [--autojoin] [--nfs share1,share2,...] [--volume volname,size,type;...] <vm-name>"
  echo "   Example: $0 --autojoin --nfs media,configs --volume db,20G,zfs;configs,10G,zfs plex01"
  exit 1
}

generate_mac() {
  printf '52:54:%02x:%02x:%02x:%02x\n' $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256))
}

# --- Parse Arguments ---
VM_NAME=""
AUTOJOIN=false
NFS_SHARES=()
VOLUMES_RAW=""

while [[ $# -gt 0 ]]; do
  case "$1" in
    --autojoin) AUTOJOIN=true; shift ;;
    --nfs) IFS=',' read -r -a NFS_SHARES <<< "$2"; shift 2 ;;
    --volume) VOLUMES_RAW="$2"; shift 2 ;;
    -h|--help) usage ;;
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
VMID_OFFSET=$(echo "$VM_NAME" | grep -o '[0-9]*' | tail -1)
[[ -z "$VMID_OFFSET" ]] && VMID_OFFSET=1
VMID=$((VMID_BASE + VMID_OFFSET))

MAC=$(generate_mac)

# --- Clone Template ---
echo "[+] Cloning template $TEMPLATE_ID to $VM_NAME (VMID $VMID)..."
qm clone "$TEMPLATE_ID" "$VMID" --full --name "$VM_NAME" --storage local-zfs || exit 1

# --- Network Setup ---
qm set "$VMID" --net0 "virtio=$MAC,bridge=$BRIDGE_NAME,tag=$DEFAULT_VLAN" --ipconfig0 ip=dhcp

# --- SSH Key Injection ---
if [[ -f "$SSH_PUBLIC_KEY_PATH" ]]; then
  qm set "$VMID" --sshkey "$SSH_PUBLIC_KEY_PATH"
  echo "[+] SSH key injected"
else
  echo "[!] SSH key not found at $SSH_PUBLIC_KEY_PATH"
fi

# --- Cloud-init user-data ---
USERDATA_FILE="/var/lib/vz/snippets/userdata-${VMID}.yaml"

cat > "$USERDATA_FILE" <<EOF
#cloud-config
hostname: $VM_NAME
manage_etc_hosts: true
ssh_pwauth: false
package_update: true
runcmd:
  - apt-get update
  - apt-get install -y docker.io nfs-common
EOF

# --- Swarm Join ---
if $AUTOJOIN; then
  if [[ ! -f "$SWARM_JOIN_TOKEN_FILE" ]]; then
    echo "[!] Swarm join token file not found"
    exit 1
  fi
  JOIN_TOKEN=$(cat "$SWARM_JOIN_TOKEN_FILE")
  cat >> "$USERDATA_FILE" <<EOF
  - docker swarm join --token $JOIN_TOKEN $SWARM_MANAGER_IP:2377
EOF
  echo "[+] Added swarm join command"
fi

# -----------------------------
# NFS Mounts (multiple, VM paths under /mnt/)
# -----------------------------
if [[ ${#NFS_SHARES[@]} -gt 0 ]]; then
  echo "[INFO] Adding NFS shares to cloud-init..."
  echo "mounts:" >> "$USERDATA_FILE"
  for share in "${NFS_SHARES[@]}"; do
    MOUNT_POINT="/mnt/$share"
    # cloud-init snippet
    echo "  - [ \"$NFS_SERVER_IP:/volume1/$share\", \"$MOUNT_POINT\", \"nfs\", \"defaults\", \"0\", \"0\" ]" >> "$USERDATA_FILE"
  done
  echo "[+] Added NFS mounts: ${NFS_SHARES[*]}"
fi

# -----------------------------
# Wait for SSH and ensure NFS mounts
# -----------------------------
echo "[INFO] Waiting for VM $VM_NAME to boot and SSH to be ready..."
wait_for_ssh "$VM_NAME"

if [[ ${#NFS_SHARES[@]} -gt 0 ]]; then
  echo "[INFO] Ensuring NFS mounts are active inside VM..."
  for share in "${NFS_SHARES[@]}"; do
    MOUNT_POINT="/mnt/$share"
    ssh root@"$VM_NAME" "
      mkdir -p $MOUNT_POINT &&
      mountpoint -q $MOUNT_POINT || mount -t nfs $NFS_SERVER_IP:/volume1/$share $MOUNT_POINT &&
      grep -q '$MOUNT_POINT' /etc/fstab || echo '$NFS_SERVER_IP:/volume1/$share $MOUNT_POINT nfs defaults 0 0' >> /etc/fstab
    "
  done
  echo "[+] All NFS shares mounted and persisted under /mnt/"
fi


# --- ZFS Volumes (multi) ---
if [[ -n "$VOLUMES_RAW" ]]; then
  IFS=';' read -ra VOLUMES_ARR <<< "$VOLUMES_RAW"
  disk_index=1  # start at scsi1 (scsi0 = root disk)

  for voldef in "${VOLUMES_ARR[@]}"; do
    volname=$(echo "$voldef" | cut -d',' -f1)
    volsize=$(echo "$voldef" | cut -d',' -f2)
    voltype=$(echo "$voldef" | cut -d',' -f3)

    [[ -z "$volsize" ]] && volsize="10G"

    if [[ "$voltype" == "zfs" ]]; then
      echo "[+] Creating ZFS volume $volname ($volsize)"
      qm set "$VMID" --scsi${disk_index} "$ZFS_CONFIG_POOL:$volsize"

      device_letter=$(printf "\\$(printf '%03o' $((97 + disk_index)))") # 97='a' => b,c,d...
      dev_path="/dev/sd${device_letter}"

      # cloud-init: format, mount, fstab entry, symlink for Docker
      cat >> "$USERDATA_FILE" <<EOF
  - mkdir -p /mnt/volumes/$volname
  - mkfs.ext4 -F $dev_path || true
  - mount $dev_path /mnt/volumes/$volname
  - echo "$dev_path /mnt/volumes/$volname ext4 defaults 0 2" >> /etc/fstab
  - mkdir -p /var/lib/docker/volumes/$volname/_data
  - ln -sf /mnt/volumes/$volname /var/lib/docker/volumes/$volname/_data
EOF

      echo "[+] Attached $volname as $dev_path → /mnt/volumes/$volname"
      disk_index=$((disk_index+1))
    else
      echo "[!] Unknown volume type: $voltype (skipping)"
    fi
  done
fi

# --- Apply Cloud-init ---
qm set "$VMID" --cicustom "user=local:snippets/userdata-${VMID}.yaml"

# --- Finished ---
echo "[✓] VM $VM_NAME (VMID $VMID) provisioned."
echo " MAC Address: $MAC"
echo " DHCP recommended before starting."
qm config $VMID
