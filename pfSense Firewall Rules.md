---
project_id: Homelab-2025
phase: "Phase 3: Network Config"
feature: Pasted image 20251224094453.png
thumbnail: thumbnails/resized/d1b412569af74ee47dc710f2d2968d8d_86cf658e.webp
tags:
  - pfSense
  - VLAN
---
## 📋 Table of Contents

- [[#VLAN 10 - Home Network - ✅]]
- [[#VLAN 20 - IoT - ✅]]
- [[#VLAN 30 - Guest - ✅]]
- [[#VLAN 40 - Homelab - ✅]]
- [[#VLAN 50 - Media - ✅]]
- [[#VLAN 60 - Infrastructure - ✅]]
- [[#VLAN 70 - Admin - ✅]]
- [[#VLAN 80 - DMZ - ✅]]
- [[#VLAN 90 - Management - ✅]]



---
# 🔥 pfSense Firewall Rules - Complete Network Configuration

(https://claude.ai/chat/f294cc03-c585-4bdb-b581-2e767652ee21#vlan-90---management)

---

## VLAN 10 - Home Network - ✅

- [x] Review HomeNetwork VLAN Rules [priority:: 1] ✅ 2025-12-23
- [x] Update HomeNetwork VLAN Rules [priority:: 3] ✅ 2025-12-23

**Subnet:** 10.0.10.0/25  
**Interface:** VLAN10_HOME  
**Purpose:** Home devices (computers, phones, tablets)

### Outbound Rules (Allow)

| Action | Interface | Protocol | Source     | Destination     | Port(s) | Description                       | Done |
| ------ | --------- | -------- | ---------- | --------------- | ------- | --------------------------------- | ---- |
| Pass   | VLAN10    | TCP/UDP  | VLAN10 net | PiHole          | 53      | Allow DNS queries to the router   | Y    |
| Pass   | VLAN10    | UDP      | VLAN10 net | 224.0.0.251     | 5353    | mDNS (Bonjour)                    | Y    |
| Pass   | VLAN10    | UDP      | VLAN10 net | 239.255.255.250 | 5353    | SSDP (Discovery)                  | Y    |
| Pass   | VLAN10    | Any      | VLAN10 Net | VLAN20          | *       | Inter-VLAN: Home -> IoT (VLAN 60) | Y    |
| Pass   | VLAN10    | Any      | VLAN10 Net | *               | *       | Allow Internet Access             | Y    |

### Inbound Rules (Deny)

| Action | Interface                     | Protocol | Source     | Destination | Port(s) | Description              | Done |
| ------ | ----------------------------- | -------- | ---------- | ----------- | ------- | ------------------------ | ---- |
| Block  | VLAN10                        | Any      | VLAN10 net | RFC1918     | Any     | Isolate from other VLANs | Y    |
| Block  | [[#VLAN 90 - Management - ✅]] | Any      | VLAN10 net | VLAN90 net  | Any     | Block Home VLAN          | Y    |

![[Pasted image 20251224094453.png]]

---

## VLAN 20 - IoT - ✅

- [x] Review IoT VLAN Rules [priority:: 1] ✅ 2025-12-16
- [x] Update IoT VLAN Rules [priority:: 3] ✅ 2025-12-16

**Subnet:** 10.0.20.0/25  
**Interface:** VLAN20_IOT  
**Purpose:** IoT devices (smart switches, cameras, etc.)

### Alias

| Alias Name             | Type     | Content                                               | Purpose                                                                   | Done |
| ---------------------- | -------- | ----------------------------------------------------- | ------------------------------------------------------------------------- | ---- |
| `RFC1918_PRIVATE`      | Networks | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`       | Used in the BLOCK rule to prevent VLAN 20 talking to any internal subnet. | Y    |
| `INFRA_CONTROLLERS`    | Hosts    | `<HA_IP_on_60>`, `<DNS_IP_on_60>`, `<NTP_IP_on_60>`   | Central servers that IoT devices must talk to.                            |      |
| `ADMIN_WORKSTATION_70` | Hosts    | `<Admin_PC_IP_on_70>`                                 | The trusted device used to SSH/HTTPS into IoT devices.                    |      |
| `HOME_CASTING_CLIENTS` | Hosts    | `<TV_IP_on_10>`, `<Phone_IP_on_10>`                   | Devices on Home (10) that initiate casting.                               |      |
| `IOT_CASTING_DEVICES`  | Hosts    | 10.0.20.20, 10.0.20.21, 10.0.20.22                    | The casting destinations on IoT (20).                                     | Y    |
| `HA_CONTROL_PORTS`     | Ports    | `8123 (HA)`, `1883 (MQTT)`, <Other Proprietary Ports> | Ports needed for the Home Assistant server to control IoT devices.        | Y    |

### Outbound Rules (Allow)

| Action | Interface | Protocol | Source     | Destination     | Port(s)    | Description                                                                      | Done |
| ------ | --------- | -------- | ---------- | --------------- | ---------- | -------------------------------------------------------------------------------- | ---- |
| Pass   | VLAN20    | UDP      | VLAN20 net | PiHole          | 53         | Allow DNS queries to the local server.                                           | Y    |
| Pass   | VLAN20    | UDP      | VLAN20 net | VLAN60 Gateway  | 123        | Allow NTP sync to the local server.                                              | Y    |
| Pass   | VLAN20    | TCP      | VLAN20 net | VLAN60 Net      | 8123, 1883 | HomeAssistant and MQTT                                                           | Y    |
| Block  | VLAN20    | Any      | VLAN20 net | RFC1918_PRIVATE | Any        | **CRITICAL: BLOCK** all traffic to other private subnets (Deny All to Internal). | Y    |
| Pass   | VLAN20    | TCP/UDP  | VLAN20 net | *               | 80, 443    | Allow Internet access for updates/cloud (must be **after** the block rule).      | Y    |
| Pass   | VLAN20    | UDP      | VLAN20 NET | 10.0.20.255     | any        | Allow UDP broadcast within the VLAN 20 subnet (required for DHCP/ARP/Discovery). | Y    |
| Pass   | VLAN20    | UDP      | VLAN20 NET | 224.0.0.251     | 5353       | Allow mDNS (Bonjour/Discovery) to multicast group.                               | Y    |
| Pass   | VLAN20    | UDP      | VLAN20 NET | 224.0.0.251     | 1900       | Allow SSDP (DLNA/Casting) to multicast group.                                    | Y    |

### Inbound Rules (Allow)

| Action | Interface | Protocol | Source     | Destination         | Port(s)         | Description                  | Done |
| ------ | --------- | -------- | ---------- | ------------------- | --------------- | ---------------------------- | ---- |
| Pass   | VLAN20    | TCP/UDP  | VLAN10 net | IOT_CASTING_DEVICES | 1900, 5353      | Casting/discovery from Home  | Y    |
| Pass   | VLAN20    | TCP      | VLAN70 net | VLAN20 net          | 80, 443, 22, 23 | Device management from Admin | Y    |
| Pass   | VLAN20    | TCP/UDP  | VLAN60 net | VLAN20 net          | Any             | HomeAssistant control        | Y    |

### Inbound Rules (Deny)

| Action | Interface              | Protocol | Source     | Destination | Port(s) | Description      | Done |
| ------ | ---------------------- | -------- | ---------- | ----------- | ------- | ---------------- | ---- |
| Block  | VLAN20                 | Any      | Any        | VLAN20 net  | Any     | Block all others | Y    |
| Block  | [[#VLAN 20 - IoT - ✅]] | Any      | VLAN20 net | VLAN90 net  | Any     | Block IoT VLAN   | Y    |

![[Pasted image 20251224105716.png]]

![[Pasted image 20251224105725.png]]

---

## VLAN 30 - Guest - ✅

- [x] Review Guest VLAN Rules [priority:: 1] ✅ 2025-12-16
- [x] Update Guest VLAN Rules [priority:: 3] ✅ 2025-12-16

**Subnet:** 10.0.30.0/25  
**Interface:** VLAN30_GUEST  
**Purpose:** Guest network access (Internet-only)

### Outbound Rules

| Action | Interface                | Protocol | Source     | Destination       | Port(s) | Description                                        | Done |
| ------ | ------------------------ | -------- | ---------- | ----------------- | ------- | -------------------------------------------------- | ---- |
| Pass   | VLAN30                   | UDP      | VLAN30 net | pihole            | 53      | Allow DNS resolution to local server.              | Y    |
| Pass   | VLAN30                   | TCP      | VLAN30 net | Any (WAN)         | 80, 443 | HTTP/HTTPS web browsing                            | Y    |
| Pass   | VLAN30                   | UDP      | VLAN30 net | VLAN60_GATEWAY_IP | 123     | Allow NTP time sync to local server (Specific IP). | Y    |
| Block  | [[#VLAN 30 - Guest - ✅]] | Any      | VLAN30 net | VLAN90 net        | Any     | Block Guest VLAN                                   | Y    |

### Inbound Rules (Deny)

| Action | Interface | Protocol | Source     | Destination      | Port(s) | Description                                                                   | Done |
| ------ | --------- | -------- | ---------- | ---------------- | ------- | ----------------------------------------------------------------------------- | ---- |
| Block  | VLAN30    | Any      | VLAN30 net | RFC1918 networks | Any     | CRITICAL: Block all access to internal networks.                              | Y    |
| Block  | VLAN30    | Any      | VLAN30 net | Any              | 53      | **Block all other DNS** (Forces use of Rule 1).                               | Y    |
| Block  | VLAN30    | Any      | VLAN30     | Any              | Any     | (The final explicit block for logging purposes, capturing all other traffic.) | Y    |

![[Pasted image 20251224100017.png]]

---

## VLAN 40 - Homelab - ✅

- [x] Review HomeLab VLAN Rules [priority:: 3] ✅ 2025-12-24
- [x] Update HomeLab VLAN Rules [priority:: 1] ✅ 2025-12-24

**Subnet:** 10.0.40.0/24  
**Interface:** VLAN40_HOMELAB  
**Purpose:** Homelab servers, VMs, containers (excluding Docker Swarm)

### Outbound Rules (Allow)

| Action | Interface | Protocol | Source     | Destination | Port(s) | Description     | Done |
| ------ | --------- | -------- | ---------- | ----------- | ------- | --------------- | ---- |
| Pass   | VLAN40    | TCP      | VLAN40 net | Pihole IP   | 53      | DNS             | Y    |
| Pass   | VLAN40    | Any      | VLAN40 net | Any         | Any     | Internet access | Y    |

### Inbound Rules (Allow)

| Action | Interface | Protocol | Source     | Destination | Port(s)     | Description             | Done |
| ------ | --------- | -------- | ---------- | ----------- | ----------- | ----------------------- | ---- |


### Inbound Rules (Deny)

| Action    | Interface | Protocol | Source               | Destination | Port(s) | Description                    |     |
| --------- | --------- | -------- | -------------------- | ----------- | ------- | ------------------------------ | --- |


### Outbound Rules (Deny)

| Action    | Interface | Protocol | Source         | Destination | Port(s) | Description                    |     |
| --------- | --------- | -------- | -------------- | ----------- | ------- | ------------------------------ | --- |
| **Block** | VLAN50    | Any      | **VLAN50 net** | RFC1918     | Any     | Block all other internal VLANs | Y   |
| Block     | VLAN50    | Any      | **VLAN50 net** | Vlan90 Net  | Any     | Block Homelab                  | Y   |
| Block     | VLAN50    | Any      | **VLAN50 net** | Vlan90      | Any     | Block Homelab                  |     |

![[Pasted image 20251224110752.png]]


---

## VLAN 50 - Media - ✅

- [x] Review media VLAN Rules [priority:: 1] ✅ 2025-12-23
- [x] Update media VLAN Rules [priority:: 2] ✅ 2025-12-24

**Subnet:** 10.0.50.0/24  
**Interface:** VLAN50_MEDIA  
**Purpose:** Media and media management services (Plex)

### Outbound Rules (Allow)

| **Action** | **Interface** | **Protocol** | **Source** | **Destination** | **Port(s)**    | **Description**                                                               | Done |
| ---------- | ------------- | ------------ | ---------- | --------------- | -------------- | ----------------------------------------------------------------------------- | ---- |
| Pass       | VLAN50        | UDP          | VLAN50 net | pihole          | 53             | DNS Resolution                                                                | Y    |
| Pass       | VLAN50        | TCP/UDP      | VLAN50 net | **VLAN60 net**  | 445, 2049, 111 | Access Storage (NAS)                                                          | Y    |
| Pass       | VLAN50        | **TCP/UDP**  | VLAN50 net | **VLAN60 net**  | **Any**        | **Access Controllers** (Webhooks/API calls to Home Assistant/Unifi)           | Y    |
| Pass       | VLAN50        | Any          | VLAN50 net | *               | *              | Allow Internet                                                                | Y    |
| Block      | VLAN50        | TCP          | VLAN50 Net | *               | *              | (The final explicit block for logging purposes, capturing all other traffic.) | Y    |
| Block      | VLAN50        | UDP          | VLAN50 Net | *               | 53             | Block all other DNS (Forces use of Rule 1).                                   | Y    |

### Inbound Rules (Deny)

| **Action** | **Interface** | **Protocol** | **Source**           | **Destination** | **Port(s)** | **Description**                | Done |
| ---------- | ------------- | ------------ | -------------------- | --------------- | ----------- | ------------------------------ | ---- |
| **Block**  | VLAN50        | Any          | **VLAN20 net** (IoT) | RFC1918         | Any         | Block all other internal VLANs | Y    |

### Inbound Rules (Allow)
| **Action** | **Interface**                   | **Protocol** | **Source**     | **Destination**   | **Port(s)**     | **Description**                                                    | Done |
| ---------- | ------------------------------- | ------------ | -------------- | ----------------- | --------------- | ------------------------------------------------------------------ | ---- |
| Pass       | [[#VLAN 90 - Management - ✅]]   | TCP          | **VLAN90 net** | VLAN50 net        | **22, 80, 443** | **Admin Access** (SSH/Web UIs for management from your Admin VLAN) | Y    |
| Pass       | [[#VLAN 10 - Home Network - ✅]] | TCP          | VLAN10 net     | Plex (10.0.50.20) | 32400           | Home VLAN → Plex                                                   | Y    |
| Pass       | [[#VLAN 20 - IoT - ✅]]          | TCP          | VLAN10 net     | Plex (10.0.50.20) | 32400           |                                                                    | Y    |
| Pass       | [[#VLAN 80 - DMZ - ✅]]          | TCP          | VLAN10 net     | Plex (10.0.50.20) | 32400           |                                                                    | Y    |
![[Pasted image 20251224100337.png]]

---

## VLAN 60 - Infrastructure - ✅
 
- [x] Review Infrastructure VLAN Rules [priority:: 1] ✅ 2025-12-24
- [x] Update Infrastructure VLAN Rules [priority:: 3] ✅ 2025-12-24

**Subnet:** 10.0.60.0/25  
**Interface:** VLAN60_INFRASTRUCTURE  
**Purpose:** Core services, databases, storage, HomeAssistant

### Outbound Rules (Allow)

| Action | Interface | Protocol | Source     | Destination | Port(s)     | Description                   | Done |
| ------ | --------- | -------- | ---------- | ----------- | ----------- | ----------------------------- | ---- |
| Pass   | VLAN60    | TCP/UDP  | VLAN60 net | WAN         | 80, 443, 53 | Internet access               | Y    |


### Inbound Rules (Allow)

| Action | Interface | Protocol | Source     | Destination | Port(s)      | Description                 |
| ------ | --------- | -------- | ---------- | ----------- | ------------ | --------------------------- |
| Pass   | VLAN60    | TCP/UDP  | VLAN70 net | VLAN60 net  | 53           | DNS from all internal VLANs |
| Pass   | VLAN60    | TCP/UDP  | VLAN70 net | VLAN60 net  | Admin_Access | Admin management            |


### Inbound Rules (Deny)

| Action | Interface | Protocol | Source     | Destination     | Port(s) | Description      | Done |
| ------ | --------- | -------- | ---------- | --------------- | ------- | ---------------- | ---- |
| Block  | VLAN60    | Any      | VLAN60 net | RFC1918_PRIVATE | Any     | Block all others | Y    |

---

## VLAN 70 - Admin - ✅

- [x] Review Infrastructure VLAN Rules [priority:: 1] ✅ 2025-12-24
- [x] Update Infrastructure VLAN Rules [priority:: 3] ✅ 2025-12-24

**Subnet:** 10.0.70.0/24  
**Interface:** VLAN70_ADMIN  
**Purpose:** Trusted admin devices (laptops, jump boxes)

### Outbound Rules (Allow)

| Action | Interface | Protocol | Source     | Destination     | Port(s)           | Description                       | Done |
| ------ | --------- | -------- | ---------- | --------------- | ----------------- | --------------------------------- | ---- |
| Pass   | VLAN70    | TCP      | VLAN70 net | VLAN70 net      | 8006, 22, 443, 80 | Proxmox and infrastructure access | Y    |
| Pass   | VLAN70    | TCP      | VLAN70 net | VLAN60 net      | 3306, 8086        | Database management               | Y    |
| Pass   | VLAN70    | TCP      | VLAN70 net | VLAN50 net      | 32400             | Plex access                       | Y    |
| Pass   | VLAN70    | TCP      | VLAN70 net | VLAN55 net      | 7878, 8989, 3579  | Radarr, Sonarr, Ombi              | Y    |
| Pass   | VLAN70    | TCP/UDP  | VLAN70 net | pfSense LAN IP  | 443, 22, 80       | pfSense Web UI and SSH            | Y    |
| Pass   | VLAN70    | TCP/ UDP | VLAN70 net | pihole          | 53                | DNS queries                       | Y    |
| Pass   | VLAN70    | Any      | VLAN70 net | Any (WAN)       | Any               | Internet access                   | Y    |
| Block  | VLAN70    | Any      | VLAN70 net | RFC1918_PRIVATE | Any               | Isolate from other vlans          | Y    |


### Inbound Rules (Deny)

| Action | Interface | Protocol | Source     | Destination | Port(s) | Description                 |
| ------ | --------- | -------- | ---------- | ----------- | ------- | --------------------------- |

![[Pasted image 20251224102613.png]]

---

## VLAN 80 - DMZ - ✅

- [x] Review DMZ VLAN Rules [priority:: 1] ✅ 2025-12-16
- [x] Update DMZ VLAN Rules [priority:: 3] ✅ 2025-12-16

**Subnet:** 10.0.80.0/24  
**Interface:** VLAN80_DMZ  
**Purpose:** Isolated zone for externally accessible services

### Outbound Rules (Allow)

| Action | Interface | Protocol | Source     | Destination    | Port(s)          | Description                                    | Done |
| ------ | --------- | -------- | ---------- | -------------- | ---------------- | ---------------------------------------------- | ---- |
| Pass   | VLAN80    | UDP      | VLAN80 net | pihole         | 53               | Allow DNS lookups.                             | Y    |
| Pass   | VLAN80    | UDP      | VLAN80 net | VLAN60 Gateway | 123              | Allow time sync.                               | Y    |
| Pass   | VLAN80    | TCP/UDP  | VLAN80 net | WAN            | WAN_OUTBOUND_DMZ | Allow restricted Internet access for services. | Y    |
| Pass   | VLAN80    | UDP      | VLAN80 net | PlexIP         | 123              | DMZ VLAN → Plex                                | Y    |
|        |           |          |            |                |                  |                                                |      |

### Inbound Rules (Deny)

| Action | Interface              | Protocol | Source     | Destination     | Port(s) | Description                                                                        | Done |
| ------ | ---------------------- | -------- | ---------- | --------------- | ------- | ---------------------------------------------------------------------------------- | ---- |
| Block  | VLAN80                 | Any      | VLAN80 net | RFC1918_PRIVATE | Any     | **CRITICAL: Deny ALL internal network access.** (Consolidates all your Deny rules) | Y    |
| Block  | VLAN80                 | Any      | Any        | ANy             | Any     | (The final implicit/explicit block.)                                               | Y    |
| Block  | [[#VLAN 80 - DMZ - ✅]] | Any      | VLAN80 net | VLAN90 net      | Any     | Block DMZ                                                                          | Y    |

![[Pasted image 20251223184112.png]]


---

## VLAN 90 - Management - ✅

- [x] Review Management VLAN Rules [priority:: 1] ✅ 2025-12-23
- [x] Update Management VLAN Rules [priority:: 1] ✅ 2025-12-24

**Subnet:** 10.0.90.0/25  
**Interface:** VLAN90_MANAGEMENT  
**Purpose:** Management interfaces only (IPMI, iLO, switch management)

### Outbound Rules (Allow)

| Action | Interface                     | Protocol | Source         | Destination     | Port(s)         | Description                                                        | Done |
| ------ | ----------------------------- | -------- | -------------- | --------------- | --------------- | ------------------------------------------------------------------ | ---- |
| Pass   | VLAN90                        | TCP/UDP  | VLAN90 net     | WAN             | 80, 443, 53     | Internet access for updates                                        | Y    |
| Pass   | [[#VLAN 90 - Management - ✅]] | TCP      | **VLAN90 net** | VLAN70 net      | **22, 80, 443** | **Admin Access** (SSH/Web UIs for management from your Admin VLAN) | Y    |
| Pass   | VLAN90                        | Any      | VLAN90 net     | VLAN90 net      | Any             | Internal management communication                                  | Y    |
| Pass   | VLAN90                        | UDP      | VLAN90 net     | WAN             | 123             | NTP time sync                                                      | Y    |
| Pass   | VLAN90                        | UDP      | VLAN90         | VLAN90          | 53              | DNS                                                                | Y    |
| Pass   | VLAN90                        | UDP      | 10.0.90.20/21  | 1.1.1.1/1.0.0.1 | 53              | CloudFlare DNS                                                     | Y    |
| Block  | VLAN90                        | TCP      | VLAN90 net     | RFC1918_PRIVATE | Any             | Prevent access to other LANs                                       | Y    |



### Inbound Rules (Allow)

| Action | Interface | Protocol | Source     | Destination | Port(s)                | Description                   | Done |
| ------ | --------- | -------- | ---------- | ----------- | ---------------------- | ----------------------------- | ---- |



### Inbound Rules (Deny)

| Action | Interface                       | Protocol | Source     | Destination | Port(s) | Description                   | Done |
| ------ | ------------------------------- | -------- | ---------- | ----------- | ------- | ----------------------------- | ---- |
| Block  | [[#VLAN 10 - Home Network - ✅]] | Any      | VLAN10 net | VLAN90 net  | Any     | Block Home VLAN               | Y    |
| Block  | [[#VLAN 20 - IoT - ✅]]          | Any      | VLAN20 net | VLAN90 net  | Any     | Block IoT VLAN                | Y    |
| Block  | [[#VLAN 30 - Guest - ✅]]        | Any      | VLAN30 net | VLAN90 net  | Any     | Block Guest VLAN              | Y    |
| Block  | [[#VLAN 40 - Homelab]]          | Any      | VLAN40 net | VLAN90 net  | Any     | Block Homelab (unless needed) | Y    |
| Block  | [[#VLAN 50 - Media]]            | Any      | VLAN50 net | VLAN90 net  | Any     | Block Media VLAN              | Y    |
| Block  | [[#VLAN 60 - Infrastructure]]   | Any      | VLAN60 net | VLAN90 net  | Any     | Block Infrastructure          | Y    |
| Block  | [[#VLAN 80 - DMZ - ✅]]          | Any      | VLAN80 net | VLAN90 net  | Any     | Block DMZ                     | Y    |
| Block  | VLAN90                          | Any      | Any        | Any         | Any     | Default Deny                  | Y    |

![[Pasted image 20251224101207.png]]


---
## VLAN 100 - Storage



----

### 🔐 pfSense Firewall Rules

#### Allow Swarm Control Traffic (VLAN 65 ↔ 85)

On **VLAN 65 → 85**:

- Allow TCP `2377` to VLAN 85 (Swarm control plane)
    
- Allow TCP/UDP `7946` to VLAN 85 (Node discovery)
    
- Allow UDP `4789` to VLAN 85 (Overlay networking)
    

On **VLAN 85 → 65**:

- Same rules in reverse to allow bidirectional traffic
    

#### Restrict Other VLAN Access

- Block all other VLANs (Guest, IoT, Media) from accessing VLAN 65 or 85
    
- Allow Admin (VLAN 70) and VPN (VLAN 95) to access VLAN 65 for manager GUI/SSH


---

[[Pfsense Configuration Details]]