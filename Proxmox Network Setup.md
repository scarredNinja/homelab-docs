---
project_id: Homelab-2025
phase: 'Phase 4: Storage Config'
tags:
  - Networking
  - Proxmox
  - vlan
  - lacp
  - infrastructure
status: Reference
---
# Proxmox Network Configuration

## Proxmox Host

## Update - v2

- Create 2 new bonded pairs and remove the other.
	- One pair for all App and Human used services
		- Most vlans
	- One for storage
		- VLAN 100
- Update applicable notes
	- Proxmox below
	- Switch notes
	- PFsense notes
	- Make sure all are linkjed

## Update

- 06/02/2026 - There is an issue with TCP connections between management and infrastructure which appear to be related to either the LACP at the server or pfsense. I need to look into the setup and see if this is actually the going to work or if it needs to be reconfigured.
- Review the last output from gemini: https://gemini.google.com/app/4e035025f8706dd2 - need to ensure both proxmox + extreme + pfsense is configured to first rule out that

## Update - old

- ~~06/02/2026 - There is an issue with TCP connections between management and infrastructure which appear to be related to either the LACP at the server or pfsense. I need to look into the setup and see if this is actually the going to work or if it needs to be reconfigured.~~
- ~~Review the last output from gemini: https://gemini.google.com/app/4e035025f8706dd2 - need to ensure both proxmox + extreme + pfsense is configured to first rule out that~~


**Note:** Setup completed 16/08/2025

> [!info] File Location `/etc/network/interfaces`
## Overview

Working configuration with VLAN-aware bridge for VMs and VLAN 90 management interface.

## Network Configuration

```bash
# Loopback interface
auto lo
iface lo inet loopback

# Physical interfaces - set to manual (no IP, used for bonding)
auto eno1
iface eno1 inet manual

auto eno2
iface eno2 inet manual

auto eno3
iface eno3 inet manual

auto eno4
iface eno4 inet manual

# LACP Bond (802.3ad) - aggregates all 4 NICs
auto bond0
iface bond0 inet manual
    bond-slaves eno1 eno2 eno3 eno4
    bond-miimon 100
    bond-mode 802.3ad
    bond-xmit-hash-policy layer3+4
    bond-lacp-rate fast

# VLAN 90 sub-interface - Proxmox Management Interface
# This is where Proxmox host itself lives
auto bond0.90
iface bond0.90 inet static
    address 10.0.90.50/24
    gateway 10.0.90.1

# VLAN-aware bridge for VMs
# VMs can be assigned to any VLAN via Proxmox GUI
auto vmbr0
iface vmbr0 inet manual
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

## Network Layout

|Component|Configuration|
|---|---|
|**Physical NICs**|4x NICs (eno1-eno4) bonded with LACP (802.3ad)|
|**Management IP**|10.0.90.50/24 on VLAN 90|
|**Gateway**|10.0.90.1 (pfSense VLAN 90 interface)|
|**VM Bridge**|vmbr0 (VLAN-aware trunk for VMs)|

## Switch Configuration Required

> [!warning] Important Switch Settings
> 
> - Port connected to Proxmox must be configured as **802.3ad LAG/LACP**
> - Port must be a **trunk** carrying all VLANs (2-4094)
> - VLAN 90 must be **tagged** on the trunk

## VM Configuration in Proxmox GUI - ✅

When creating VMs, configure the network device:

- **Bridge:** `vmbr0`
- **VLAN Tag Examples:**
    - `60` - for Traefik VM
    - `90` - for VMs on management network
    - `[any]` - for VMs on other VLANs

## Apply Configuration - ✅

```bash
# Apply changes without reboot
ifreload -a

# Or for safety (if remote)
reboot
```

## Verify Configuration  - ✅

```bash
# Check IP addresses
ip addr show

# Check routing table
ip route

# Test gateway connectivity
ping 10.0.90.1

# Check bridge configuration
brctl show vmbr0
```



---

## Updates Made:

## 💡 The "Swarm Manager" Fixes

### 1. Pinned the Swarm Control Plane

- **The Problem:** Docker Swarm was confused by having three different network interfaces (VLAN 60, 80, and 90) and was trying to use the wrong one for cluster heartbeats.
    
- **The Fix:** Used `docker swarm init --force-new-cluster --advertise-addr 10.0.60.30` to strictly bind the "brain" of the Swarm to the **Infrastructure VLAN**.
    
- **Why it matters:** This ensures that cluster traffic stays on your internal management network and doesn't leak out to the DMZ.
    

---

## 🌐 The OS & Networking Fixes

### 2. Corrected Interface Naming (Netplan)

- **The Problem:** Your Netplan config was looking for `enp0sX` names, but the OS (Proxmox/Ubuntu) had named them `eth0`, `ens19`, and `ens20`.
    
- **The Fix:** Updated `/etc/netplan/00-installer-config.yaml` to match the actual hardware names found in `ip link`.
    

### 3. Established a Single Default Route

- **The Problem:** Multi-homed VMs often try to create multiple "default gateways," which causes "Asymmetric Routing" (packets go out one door and try to come back in another), breaking the internet connection.
    
- **The Fix:** Ensured only **VLAN 60 (eth0)** has a `routes: - to: default` entry. The DMZ and Management interfaces were set up without gateways to keep traffic local.
    

### 4. Resolved DNS "Blindness"

- **The Problem:** The VM couldn't resolve `docker.io` because the system resolver was timing out, and pfSense was initially blocking DNS queries from the Infrastructure VLAN.
    
- **The Fix:** * Updated pfSense rules to allow DNS (Port 53) on the Infrastructure interface. Fixed by turning on DNS resolver
        

---

## 🔒 The Traefik & Proxy Fixes

Updated /etc/hosts to pass in the correct IP and hostname: 192.168.1.159 pve.homebaconhome.com pve

### 5. Configured "Host Mode" Port Binding

- **The Problem:** Standard Docker ingress hides the real client IP and can bind to all interfaces randomly.
    
- **The Fix:** Used `mode: host` and ~~`host_ip: 10.0.80.30` in the Traefik stack file.~~
    
- **Why it matters:** This forces Traefik to listen **only** on the DMZ interface, keeping your proxy separate from your management traffic.
    

### 6. Bridged the VLAN Gap (File Provider)

- **The Problem:** Traefik in Docker can't "see" the Proxmox UI because Proxmox isn't a Docker container.
    
- **The Fix:** Used a **Dynamic File Provider** (`proxmox.yml`) with `insecureSkipVerify: true` to tell Traefik to manually route traffic to the Proxmox IP (`10.0.90.5:8006`).
    

---

### 📝 Final Cheat Sheet for Updates

When you go to update Traefik or add a new service, remember this "Rule of Three":

1. **Placement:** Always keep Traefik on the `manager` node so it can read the Docker socket.
    
2. **IP Affinity:** If you add a new "DMZ" service, always check which `host_ip` it is binding to.
    
3. **Firewall:** If you get a "504 Timeout" in the future, it is almost certainly a **pfSense rule** blocking the traffic between the VLANs.
    

**Now that the proxy is up, is the Proxmox UI loading correctly through the `proxmox.yourdomain.com` URL?**
