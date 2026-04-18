


#!/bin/bash

# --- Config ---
TEMPLATE_ID=110
VMID_BASE=200
BRIDGE_NAME="vmbr0"
STORAGE="local-zfs"
CI_USER="root"
CI_PASSWORD="changeme"
SSH_PUBLIC_KEY_PATH="$HOME/.ssh/id_rsa.pub"
SWARM_MANAGER_IP="10.0.60.30"
NFS_SERVER_IP="10.0.60.80"

# --- Functions ---
usage() {
    cat <<EOF
Usage: $0 <vm-name> [--role manager|worker] [--init] [--autojoin] [--volume volname,size;...] [--nfs share1,share2,...]

Options:
  --role        Role of the VM: manager or worker (default: worker)
  --init        (Managers only) Run 'docker swarm init' (only for the first manager)
  --autojoin    Auto-join swarm (for workers and additional managers)
  --volume      Semicolon-separated list of Proxmox volumes: volname,size
  --nfs         Comma-separated list of NFS shares (only for workers)
EOF
    exit 1
}

generate_mac() {
    echo "02:$(openssl rand -hex 5 | sed 's/\(..\)/\1:/g; s/:$//')"
}

# --- Parse Arguments ---
VM_NAME=""
ROLE="worker"
INIT_MANAGER=false
AUTOJOIN=false
VOLUMES_RAW=""
NFS_SHARES=()

while [[ $# -gt 0 ]]; do
    case "$1" in
        --role)
            ROLE="$2"
            if [[ "$ROLE" != "manager" && "$ROLE" != "worker" ]]; then
                echo "[!] Invalid role: $ROLE"
                usage
            fi
            shift 2
            ;;
        --init)
            INIT_MANAGER=true
            shift
            ;;
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
    --net0 virtio=${MAC},bridge=${BRIDGE_NAME},tag=$([[ "$ROLE" == "worker" ]] && echo 85 || echo 90) \
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

# --- Prepare cloud-init user-data ---
# Store it where Proxmox expects it
SNIPPET_DIR="/var/lib/vz/snippets"
USERDATA_FILE="${SNIPPET_DIR}/userdata-${VMID}.yaml"

cat <<EOF > "$USERDATA_FILE"
#cloud-config
# System Users
users:
  - name: dockeradmin
    groups: sudo, docker
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash

# Native Virtio-FS Mounts
mounts:
  - [ docker-data, /mnt/docker-data, virtiofs, "defaults", "0", "0" ]
  - [ docker-db, /mnt/docker-db, virtiofs, "defaults", "0", "0" ]
  - [ docker-tsdb, /mnt/docker-tsdb, virtiofs, "defaults", "0", "0" ]
  - [ docker-swarm, /mnt/docker-swarm, virtiofs, "defaults", "0", "0" ]

runcmd:
  - mkdir -p /mnt/docker-data /mnt/docker-db /mnt/docker-tsdb /mnt/docker-swarm
  - chown -R 1000:1000 /mnt/docker-data /mnt/docker-db /mnt/docker-tsdb /mnt/docker-swarm
EOF

# Logic for Swarm (using variables from your existing script)
if [ "$ROLE" == "manager" ]; then
    echo "  - docker swarm init --advertise-addr 10.0.60.30" >> "$USERDATA_FILE"
elif [ "$ROLE" == "worker" ]; then
    JOIN_TOKEN=$(cat /root/swarm_worker_token.txt)
    echo "  - docker swarm join --token $JOIN_TOKEN 10.0.60.30:2377" >> "$USERDATA_FILE"
fi

# Link it to Proxmox
qm set $VMID --cicustom "user=local:snippets/userdata-${VMID}.yaml"

# --- Output ---
echo "[✓] VM $VM_NAME (VMID $VMID) provisioned as $ROLE"
echo "    MAC Address: $MAC"
echo "    DHCP reservation recommended before starting the VM."

qm config $VMID
