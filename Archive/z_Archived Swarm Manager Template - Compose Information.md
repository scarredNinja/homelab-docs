---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
Swarm Manager 1 Join Token: SWMTKN-1-06b4vjctnxxh2tpz66shtbh90pnegi276c6vcp7hemcrwn1dh0-1ovi0r7vsv6w80wehindwlimn

# Manager Creation

## VM Creation

qm create 200 \

  --name swarm-manager-01 \

  --memory 6144 \

  --cores 3 \

  --net0 virtio,bridge=vmbr0,tag=60 \

  --scsi0 local-zfs:50 \

  --ostype l26 \

  --cpu host \

  --agent 1

## Set boot order

qm set 200 --boot order=scsi0

- - IP: `10.0.60.10/25`
    - Gateway: `10.0.60.1`
    - DNS: `8.8.8.8,1.1.1.1` (or your preferred)

**System Settings:**

- Hostname: `swarm-manager-01`
- Username: swarm/swarm01
- Enable SSH server: ✅ Yes

**Storage:**

- Use entire disk (the 50GB we allocated)


## Test Config:

This will give you:  
✅ **1. Swarm Compose Templates (grouped & VLAN aware)**  
✅ **2. Mapping of existing stacks → overlay networks + VLANs**  
✅ **3. Step-by-step migration plan (safe cutover, rollback-ready)**

---

# 🛠 **1. Swarm Compose Templates**

Here’s the **base Swarm layout** for your groups. You’ll apply one group at a time.

### ✅ `docker-compose.yml` (example for Media group)

```yaml
version: "3.9"

networks:
  media_net:
    driver: overlay
    attachable: true

services:
  plex:
    image: linuxserver/plex:latest
    deploy:
      placement:
        constraints:
          - node.role == manager
      replicas: 1
    ports:
      - "32400:32400"  # Only open Plex externally
    volumes:
      - /mnt/media/config/plex:/config
      - /mnt/media/library:/data
    networks:
      - media_net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.plex.rule=Host(`plex.yourdomain.com`)"
      - "traefik.http.routers.plex.entrypoints=websecure"
      - "traefik.http.routers.plex.tls.certresolver=myresolver"
      - "traefik.docker.network=media_net"

  downloader:
    image: linuxserver/sabnzbd:latest
    deploy:
      placement:
        constraints:
          - node.role == worker
    volumes:
      - /mnt/media/config/sabnzbd:/config
      - /mnt/media/downloads:/downloads
    networks:
      - media_net
    labels:
      - "traefik.enable=false"
```

### 🌐 **Overlay Networks (Swarm scope)**

| Group               | Network Name | VLAN Tag | Subnet       |
| ------------------- | ------------ | -------- | ------------ |
| Media               | `media_net`  | 50       | 10.0.50.0/25 |
| IoT (HomeAssistant) | `iot_net`    | 20       | 10.0.20.0/25 |
| Infrastructure      | `infra_net`  | 60       | 10.0.60.0/25 |
| Databases           | `db_net`     | 65       | 10.0.65.0/25 |

All these networks will align with **Proxmox vmbrXX VLAN bridges**.

---

# 🗺 **2. Mapping Stacks to Overlay Networks**

|**Stack ID**|**Purpose**|**Mapped Network**|**VLAN**|External Access?|
|---|---|---|---|---|
|101|Plex Media Server|`media_net`|50|✅ (Plex only)|
|55|Radarr, Sonarr, etc.|`mgmt_net`|55|❌ VPN only|
|78|Downloaders|`media_net`|50|❌ VPN only|
|120|Home Assistant|`iot_net`|20|✅ (if needed)|
|109|InfluxDB, MySQL|`db_net`|65|❌ VPN only|
|14|Grafana|`infra_net`|60|❌ VPN only|
|2|Dashy (dashboard)|`infra_net`|60|❌ VPN only|

- 🟢 **Plex & Home Assistant:** exposed via Traefik to WAN.
    
- 🔒 Everything else: VPN-only access.
    

---

# 📦 **3. Migration Plan (Minimized Downtime)**

### 🟢 **Step A: Prepare Environment**

1. Setup VLANs on pfSense and switch trunk ports (you’ve already done this).
    
2. Setup **Proxmox vmbrXX VLAN bridges** for Swarm nodes.
    
3. Deploy Swarm cluster:
    
    ```bash
    docker swarm init --advertise-addr <manager_IP>
    ```
    
4. Create overlay networks:
    
    ```bash
    docker network create --driver=overlay --attachable media_net
    docker network create --driver=overlay --attachable mgmt_net
    ...
    ```
    

---

### 🟢 **Step B: Deploy Traefik in Swarm**

1. Deploy Traefik as your Swarm ingress controller.
    
2. Configure `acme.json` for LetsEncrypt certs.
    
3. Apply external firewall rules:
    
    - Plex & HomeAssistant → allow from WAN
        
    - Everything else → VPN only.
        

---

### 🟢 **Step C: Migrate Stacks**

1. For each stack:
    
    - Update `docker-compose.yml` for Swarm compatibility.
        
    - Add it to the relevant overlay network.
        
2. Deploy in order of **lowest dependencies**:
    
    - Infrastructure (Traefik, dashboards)
        
    - Databases
        
    - Apps (Plex, HomeAssistant)
        
3. Test each service internally and externally.
    

---

### 🔥 Rollback Strategy

✅ Keep original Portainer running during migration.  
✅ Test Swarm deployment stack by stack before shutting down old containers.

---

## 🚀 Next: Storage Configuration

Once you review, we’ll plan:  
✅ **Bind-mounts or NFS shares** for persistent storage.  
✅ Swarm volume drivers (local, NFS, CIFS, etc.).

---

Do you want me to:  
✅ Generate a **full Swarm deployment bundle** (Traefik + all stacks grouped)?  
✅ Or focus first on **Proxmox VLAN bridge + storage binding plan** before generating code?

Which one do you want to tackle next?
