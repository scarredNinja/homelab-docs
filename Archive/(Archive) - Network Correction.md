# Proxmox Network Recovery & Security Setup

## Phase 1: Restore Proxmox GUI Access

		- [ ] 🔺 Go over network arcitecture again and consider the points made in: [[Homelab Rebuild 2025/Infrastructure/Networking/Notes|Notes]] #HomeLabRebuild/Network #HomeLabRebuild/Checklist  #todoist
### Step 1: Backup Current Configuration

```bash
# SSH into Proxmox (if possible) or use console
cp /etc/network/interfaces /etc/network/interfaces.backup.$(date +%Y%m%d)
cp /etc/pve/nodes/$(hostname)/config /etc/pve/nodes/$(hostname)/config.backup
```

### Step 2: Simplified Management Interface

```bash
# Edit network configuration
nano /etc/network/interfaces

# Remove complex bonds and replace with:
auto lo
iface lo inet loopback

# Management Interface (VLAN 90) - Critical for GUI access
auto eno1
iface eno1 inet manual

# Management Bridge with VLAN 90
auto vmbr90
iface vmbr90 inet static
    address 10.0.90.10/25
    gateway 10.0.90.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
    # VLAN 90 will be untagged on this bridge
```

### Step 3: Apply Network Configuration

```bash
# Restart networking (be careful - you might lose connection)
systemctl restart networking

# Or safer approach:
ifdown vmbr90 && ifup vmbr90

# Test connectivity
ping 10.0.90.1  # pfSense gateway
curl -k https://10.0.90.10:8006  # Test Proxmox GUI
```

## Phase 2: Multi-VLAN Bridge Setup with Traffic Separation

### Complete Network Configuration

```bash
# /etc/network/interfaces - Full configuration

auto lo
iface lo inet loopback

# === Physical Interfaces ===
auto eno1
iface eno1 inet manual

auto eno2  
iface eno2 inet manual

auto eno3
iface eno3 inet manual

auto eno4
iface eno4 inet manual

# === Management Bridge (VLAN 90) ===
auto vmbr90
iface vmbr90 inet static
    address 10.0.90.10/25
    gateway 10.0.90.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 90
    # DNS servers
    dns-nameservers 10.0.60.1 1.1.1.1

# === Infrastructure Bridge (VLAN 60) ===
auto vmbr60
iface vmbr60 inet manual
    bridge-ports eno2
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 60

# === Workload Separation Bridge (VLAN 85) ===
auto vmbr85
iface vmbr85 inet manual
    bridge-ports eno2
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 85

# === Media Bridge (VLAN 50) ===
auto vmbr50
iface vmbr50 inet manual
    bridge-ports eno3
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 50

# === VPN/Secure Bridge (VLAN 75) - ISOLATED ===
auto vmbr75
iface vmbr75 inet manual
    bridge-ports eno4
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 75
    # Dedicated NIC for VPN traffic isolation

# === Client Access Bridge (Multiple VLANs) ===
auto vmbr-multi
iface vmbr-multi inet manual
    bridge-ports eno3
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 10,20,30,40,70,80
```

## Phase 3: VLAN Connectivity Testing

### Test Script for VLAN Access

```bash
#!/bin/bash
# /root/test-vlan-connectivity.sh

echo "=== Testing VLAN Connectivity from Proxmox ==="

VLANS=(
    "60:10.0.60.1:Infrastructure"
    "50:10.0.50.1:Media"
    "85:10.0.85.1:Docker_Workers"
    "75:10.0.75.1:VPN"
    "10:10.0.10.1:Home"
    "20:10.0.20.1:IoT"
    "40:10.0.40.1:Homelab"
    "70:10.0.70.1:Admin"
    "80:10.0.80.1:DMZ"
)

for vlan_info in "${VLANS[@]}"; do
    IFS=':' read -r vlan gateway name <<< "$vlan_info"
    echo "Testing VLAN $vlan ($name) - Gateway: $gateway"
    
    if ping -c 2 -W 2 "$gateway" > /dev/null 2>&1; then
        echo "✅ VLAN $vlan ($name) - REACHABLE"
    else
        echo "❌ VLAN $vlan ($name) - UNREACHABLE"
    fi
    echo ""
done

# Test external connectivity
echo "Testing external connectivity..."
if ping -c 2 -W 2 1.1.1.1 > /dev/null 2>&1; then
    echo "✅ Internet connectivity - OK"
else
    echo "❌ Internet connectivity - FAILED"
fi
```

## Phase 4: HomeLab Security Best Practices

### Traffic Separation Strategy

```
HIGH SECURITY ZONES:
├── VLAN 90 (Management) - Proxmox GUI, IPMI, switches
├── VLAN 60 (Infrastructure) - Docker managers, DNS, NTP
└── VLAN 75 (VPN) - COMPLETELY ISOLATED

MEDIUM SECURITY ZONES:  
├── VLAN 85 (Docker Workers) - Container workloads
├── VLAN 50 (Media) - Plex, NAS access
└── VLAN 70 (Admin) - Admin workstations

LOW SECURITY ZONES:
├── VLAN 10 (Home) - Trusted devices
├── VLAN 20 (IoT) - Smart devices
├── VLAN 40 (Homelab) - Testing/development
├── VLAN 30 (Guest) - Untrusted devices
└── VLAN 80 (DMZ) - Public services
```

### pfSense Firewall Rules (Critical)

```
VLAN 90 (Management) Rules:
- Allow 70 → 90 (Admin workstation access)
- Allow 75 → 90 (VPN admin access)  
- Allow 90 → 60 (Infrastructure services)
- Allow 90 → Internet (Updates only)
- BLOCK ALL OTHER → 90

VLAN 75 (VPN) Rules - STRICT ISOLATION:
- Allow 75 → Internet (VPN traffic only)
- Allow 75 → 90 (Management - admin only)
- BLOCK 75 → ALL OTHER VLANs
- BLOCK ALL OTHER VLANs → 75

VLAN 60 (Infrastructure) Rules:
- Allow 60 → Internet (Updates, NTP, DNS)
- Allow 60 → 50 (Media storage access)
- Allow 60 → 90 (Management reporting)
- BLOCK 60 → Client VLANs (10,20,30)

VLAN 85 (Docker Workers) Rules:  
- Allow 85 → 60 (Manager communication)
- Allow 85 → 50 (Media access)
- Allow 85 → Internet (Container registries)
- BLOCK 85 → Management VLANs
```

### Network Monitoring Setup

```bash
# Install network monitoring tools
apt update && apt install -y tcpdump iftop vnstat

# Monitor VLAN traffic
tcpdump -i vmbr90 -nn host 10.0.90.1  # Management traffic
tcpdump -i vmbr75 -nn                 # VPN traffic (should be minimal)

# Check for unexpected cross-VLAN traffic
tcpdump -i any -nn | grep -E "(10\.0\.90|10\.0\.75)"
```

## Phase 5: Verification Checklist

### Before Proceeding to VMs:

- [ ] Proxmox GUI accessible via https://10.0.90.10:8006 #todoist
- [ ] Can ping all VLAN gateways from Proxmox host
- [ ] VPN VLAN (75) properly isolated #todoist
- [ ] No unexpected cross-VLAN traffic #todoist
- [ ] pfSense firewall rules implemented #todoist
- [ ] Network monitoring active #todoist

### Troubleshooting Commands

```bash
# Check bridge status
brctl show

# Verify VLAN configuration  
bridge vlan show

# Test specific VLAN routing
ping -I vmbr60 10.0.60.1  # Test via specific bridge

# Check routing table
ip route show

# Monitor real-time traffic
watch -n 1 'cat /proc/net/dev'

# Check for dropped packets
netstat -i
```

### Recovery Plan (If Things Break)

```bash
# Emergency network reset
systemctl stop networking
cp /etc/network/interfaces.backup.* /etc/network/interfaces
systemctl start networking

# Or use console access to reset to simple config
```

## Next Steps After Verification

1. **Confirm GUI Access**: Verify Proxmox web interface works reliably
2. **Test VLAN Routing**: Run connectivity test script
3. **Implement Security**: Apply pfSense firewall rules
4. **Monitor Traffic**: Ensure VPN isolation works
5. **Proceed to VM Templates**: Once networking is stable
# Extreme Switch Configuration for Proxmox VLANs

## Current VLAN Structure (From Your Table)

```
VLAN 10: Home (10.0.10.0/25)
VLAN 20: IoT (10.0.20.0/25) 
VLAN 30: Guest (10.0.30.0/25)
VLAN 40: Homelab (10.0.40.0/25)
VLAN 50: Media (10.0.50.0/25)
VLAN 60: Infrastructure (10.0.60.0/25)
VLAN 70: Admin (10.0.70.0/25)
VLAN 75: VPN (10.0.75.0/25)
VLAN 80: DMZ (10.0.80.0/25)
VLAN 85: Docker Workers (10.0.85.0/25) [NEW]
VLAN 90: Management (10.0.90.0/25)
```

## Extreme Switch Port Configuration

### Step 1: Create VLANs (if not already created)

```bash
# Connect to Extreme switch
configure

# Create any missing VLANs
create vlan "Docker-Workers" 
configure vlan Docker-Workers tag 85
# Note: No IP needed on switch - pfSense handles routing/DHCP

# Verify all VLANs exist
show vlan
```

### Step 2: Configure Proxmox Server Ports

**Assuming your Proxmox server connects to ports 1-4 on the switch:**

### Step 3: Verify pfSense Bond (Ports 1 & 3)

**Your pfSense is already bonded on ports 1 & 3, so verify the current setup:**

```bash
# Check existing bond configuration
show sharing

# Verify all VLANs are present on the pfSense bond
show vlan Management
show vlan Infrastructure
show vlan Docker-Workers
show vlan Home
show vlan IoT
show vlan Guest  
show vlan Homelab
show vlan Media
show vlan Admin
show vlan VPN
show vlan DMZ

# Ensure all VLANs are tagged on ports 1,3 (pfSense bond)
configure vlan Management add ports 1,3 tagged
configure vlan Home add ports 1,3 tagged
configure vlan IoT add ports 1,3 tagged
configure vlan Guest add ports 1,3 tagged
configure vlan Homelab add ports 1,3 tagged
configure vlan Media add ports 1,3 tagged
configure vlan Infrastructure add ports 1,3 tagged
configure vlan Admin add ports 1,3 tagged
configure vlan VPN add ports 1,3 tagged
configure vlan DMZ add ports 1,3 tagged
configure vlan Docker-Workers add ports 1,3 tagged
```

### Step 4: Security Configuration

```bash
# Disable unused ports (security best practice) 
# Skip ports 1,3 (pfSense), 37,39,41,43 (Proxmox)
configure ports 2,4-36,38,40,42,44-48 display-string "UNUSED-DISABLED"
disable ports 2,4-36,38,40,42,44-48

# Enable port security on Proxmox ports
configure ports 37 learning max-count 5
configure ports 39 learning max-count 10  
configure ports 41 learning max-count 10
configure ports 43 learning max-count 5

# Configure STP for loop prevention
configure stp s0 add vlan Management ports all
enable stp s0
configure stp s0 priority 4096
```

## Verification Commands

### Check VLAN Configuration

```bash
# Verify VLAN membership
show vlan Management
show vlan Infrastructure  
show vlan Docker-Workers
show vlan VPN

# Check port configuration
show ports 37,39,41,43 information detail
show ports 1,3 information detail

# Verify VLAN tags
show fdb vlan Management
```

### Test Connectivity

```bash
# From switch, test VLAN gateways
ping 10.0.90.1   # Management
ping 10.0.60.1   # Infrastructure
ping 10.0.85.1   # Docker Workers
ping 10.0.75.1   # VPN
```

## Alternative Configuration (If Current Setup Differs)

### Option A: Single Trunk Port Configuration

If you prefer all VLANs on one connection:

```bash
# Configure single trunk port (e.g., port 1) to Proxmox
configure vlan Management add ports 1 tagged
configure vlan Home add ports 1 tagged
configure vlan IoT add ports 1 tagged
configure vlan Guest add ports 1 tagged
configure vlan Homelab add ports 1 tagged
configure vlan Media add ports 1 tagged
configure vlan Infrastructure add ports 1 tagged
configure vlan Admin add ports 1 tagged
configure vlan VPN add ports 1 tagged
configure vlan DMZ add ports 1 tagged
configure vlan Docker-Workers add ports 1 tagged
configure vlan default delete ports 1
```

**Corresponding Proxmox Config:**

```bash
# Single interface with all VLANs
auto eno1
iface eno1 inet manual

auto vmbr-all
iface vmbr-all inet manual
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 10,20,30,40,50,60,70,75,80,85,90

# Management with static IP
auto vmbr-all.90
iface vmbr-all.90 inet static
    address 10.0.90.50/25
    gateway 10.0.90.1
```

## Troubleshooting Common Issues

### Issue: VMs Can't Reach Other VLANs

```bash
# Check if inter-VLAN routing is enabled on pfSense
# Verify VLAN tagging on switch ports
show ports 1-4 vlan

# Check for VLAN mismatches
show log | include VLAN
```

### Issue: Proxmox GUI Not Accessible

```bash
# Verify management VLAN configuration
show vlan Management ports
show fdb vlan Management

# Check port status
show ports 1 statistics
```

### Issue: Performance Problems

```bash
# Check for errors/drops
show ports 37,39,41,43 statistics | include error
show ports 37,39,41,43 statistics | include drop

# Enable flow control if needed
configure flow-control 37,39,41,43 rx on tx on
```

## Security Recommendations

### Port Security

```bash
# Limit MAC addresses per port
configure ports 37,39,41,43 learning max-count 10
configure ports 37,39,41,43 security max-count 10

# Enable violation actions
configure ports 37,39,41,43 security violation shutdown-port
```

### VLAN Isolation

```bash
# Ensure VPN VLAN is isolated
configure vlan VPN private-vlan primary
configure access-list deny-inter-vlan "deny any to vlan VPN"
```

## Configuration Backup

```bash
# Save configuration
save configuration

# Backup to TFTP (optional)
upload configuration 10.0.90.x backup-switch-config.cfg
```

## Next Steps After Switch Configuration

1. **Apply switch configuration** following the steps above
2. **Verify VLAN connectivity** from switch to pfSense
3. **Test Proxmox network configuration** with proper switch support
4. **Validate inter-VLAN routing** through pfSense
5. **Proceed with VM provisioning templates**

**Which switch model/OS version are you running?** This will help me provide more specific commands if needed. Also, **which ports is your Proxmox server currently connected to?**
