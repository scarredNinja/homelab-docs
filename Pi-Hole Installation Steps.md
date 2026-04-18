---
project_id: Homelab-2025
phase: "Phase 3: Network Config"
tags:
  - pihole
---

## Tasks

- [x] - Review Pi hole setup with two pihole instances on raspberry pi. [priority:: 1] ✅ 2026-02-17
	- Status: Currently not HA, only on one 
 - [x] format both pis ✅ 2026-02-18
	 - [x] pi 1 ✅ 2026-02-18
	 - [x] p2 ✅ 2026-02-18
- [x] Pi 1 - Set ip to VLAN 60 ✅ 2026-02-24
	- Update the ports VLAN 

**Core Components:**

- 2x Raspberry Pis (`pihole1` and `pihole2`)
    
- Pi-hole on each node
    
- Nebula overlay network (for secure connectivity and sync)
    
- Gravity Sync (for automatic Pi-hole database synchronization)
    
- (Optional but recommended) Keepalived or DNS load-balancing via pfSense / DHCP round-robin
    

---

## 🪜 Step-by-Step Setup

### **1️⃣ Prepare the Pis**

On both Raspberry Pis:

`sudo apt update && sudo apt upgrade -y sudo apt install curl git unzip -y hostnamectl set-hostname pihole1  # and pihole2 on the other`

---

### **2️⃣ Install Pi-hole on Each**

Use the automated installer:

`curl -sSL https://install.pi-hole.net | bash`

When prompted:

- Interface: `eth0`
    
- Static IPs (make sure both are fixed)
    
    - Pi 1 → `10.0.50.10`
        
    - Pi 2 → `10.0.50.11`
        
- Upstream DNS: `1.1.1.1` or whatever you want (we’ll sync this later)
    

After install, confirm:

`pihole -v`

and web interfaces work:  
`http://10.0.50.10/admin` and `http://10.0.50.11/admin`

---

### **3️⃣ Install and Configure Nebula**

This gives you a private mesh network so both Pi-holes talk securely even across VLANs or WANs.

#### On your management system (like your PC or pfSense box):

1. Install Nebula tools:
    
    `curl -LO https://github.com/slackhq/nebula/releases/latest/download/nebula-linux-amd64.tar.gz tar xvf nebula-linux-amd64.tar.gz`
    
2. Generate CA and certificates:
    
    `./nebula-cert ca -name "pihole-net" ./nebula-cert sign -name "pihole1" -ip "10.255.0.1/24" ./nebula-cert sign -name "pihole2" -ip "10.255.0.2/24"`
    
3. Copy each node’s `.crt`, `.key`, and `ca.crt` to `/etc/nebula/` on the Pis.
    

#### On each Pi:

1. Install Nebula:
    
    `sudo mkdir -p /etc/nebula cd /etc/nebula wget https://github.com/slackhq/nebula/releases/latest/download/nebula-arm64.tar.gz tar xvf nebula-arm64.tar.gz sudo mv nebula /usr/local/bin/`
    
2. Place certs and config in `/etc/nebula/config.yml`, for example:
    
    `pki:   ca: /etc/nebula/ca.crt   cert: /etc/nebula/pihole1.crt   key: /etc/nebula/pihole1.key  static_host_map:   "10.255.0.2": ["10.0.50.11:4242"]  # pihole2  lighthouse:   am_lighthouse: true   interval: 60  listen:   host: 0.0.0.0   port: 4242  punchy:   punch: true  tun:   dev: nebula1   drop_local_broadcast: false   drop_multicast: false`

Enable on boot:

`sudo nano /etc/systemd/system/nebula.service`

`[Unit] Description=Nebula Network After=network-online.target  [Service] ExecStart=/usr/local/bin/nebula -config /etc/nebula/config.yml Restart=always  [Install] WantedBy=multi-user.target`

`sudo systemctl enable nebula --now`
