---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
#HomeLabRebuild/Swarm #HomeLabRebuild/Automation

#!/bin/bash

#!/bin/bash

# === CONFIGURATION ===
VLAN_INTERFACE="bond0.60"

SWARM_MANAGER_IP="10.0.60.30"
SWARM_JOIN_TOKEN=" SWMTKN-1-06b4vjctnxxh2tpz66shtbh90pnegi276c6vcp7hemcrwn1dh0-1ovi0r7vsv6w80wehindwlimn"
SWARM_ROLE="worker"  # or "manager"

# === CONFIGURE DHCP (netplan) ===
NETPLAN_FILE="/etc/netplan/01-netcfg.yaml"

echo "[+] Creating DHCP netplan config for VLAN interface..."
cat <<EOF > $NETPLAN_FILE
network:
  version: 2
  ethernets:
    $VLAN_INTERFACE: {}
  vlans:
    vlan60:
      id: 60
      link: $VLAN_INTERFACE
      dhcp4: true
  renderer: networkd
EOF

echo "[+] Applying netplan config..."
netplan apply
sleep 5  # Wait a moment for IP lease

# === GET IP AND SET HOSTNAME ===
VM_IP=$(hostname -I | awk '{print $1}')
OCTET=$(echo "$VM_IP" | cut -d '.' -f 4)
NEW_HOSTNAME="swarm-worker-$OCTET"

echo "[+] Detected IP: $VM_IP"
echo "[+] Setting hostname to $NEW_HOSTNAME..."
hostnamectl set-hostname $NEW_HOSTNAME
echo "127.0.1.1 $NEW_HOSTNAME" >> /etc/hosts

# === JOIN SWARM ===
echo "[+] Checking for existing swarm membership..."
if docker info | grep -q "Swarm: active"; then
  echo "[!] Already part of a swarm, leaving..."
  docker swarm leave --force
fi

echo "[+] Joining Docker Swarm as $SWARM_ROLE..."
docker swarm join --token $SWARM_JOIN_TOKEN $SWARM_MANAGER_IP:2377

if [ $? -eq 0 ]; then
  echo "[✓] Successfully joined the swarm as $NEW_HOSTNAME!"
else
  echo "[✗] Failed to join the swarm!"
  exit 1
fi

echo "[✓] VM Provisioning complete!"



docker swarm join --token SWMTKN-1-06b4vjctnxxh2tpz66shtbh90pnegi276c6vcp7hemcrwn1dh0-1ovi0r7vsv6w80wehindwlimn 10.0.60.30:2377
