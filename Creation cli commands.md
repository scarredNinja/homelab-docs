---
project_id: Homelab-2025
status: Reference
phase: 'Phase 5: Docker Swarm'
tags:
  - reference
  - architecture
---
## [[(Archived) - provision-docker-swarm-vm v12]] Usaage

# Docker Swarm VM Provisioning Script Options (v9)

| Option | Description | Example |
|--------|-------------|---------|
| `--name <vmname>` | Name of the VM and hostname inside the OS | `--name swarm-manager-01` |
| `--vlan <tag>` | VLAN tag for the VM network interface | `--vlan 90` |
| `--volume <name:size>` | ZFS volume to create and mount at `/mnt/<name>` | `--volume data:50G` |
| `--nfs <share1,share2,...>` | Synology NFS shares to mount at `/mnt/media/<share>` | `--nfs movies,tv,music` |
| `--first-manager` | Bootstraps swarm as the first manager and **auto-saves tokens** | `--first-manager` |
| `--autojoin` | Auto-joins swarm using Proxmox-saved tokens | `--autojoin --role worker` |
| `--role <worker|manager>` | Role for autojoin nodes | `--role manager` |
| `--user <username>` | Creates a non-root user inside the VM with sudo privileges | `--user dockeradmin` |
| `--password <plaintext>` | Plaintext password for the user (auto-hashed for cloud-init) | `--password MyPlaintextPassword` |
| `--template-id <id>` | Override default Proxmox template ID | `--template-id 105` |
| `--autostart <true|false>` | Automatically start the VM after creation | `--autostart false` |

---

nano provision-swarm-vm.sh
chmod +x provision-swarm-vm.sh
## Notes

- **Swarm Tokens Saved Automatically (First Manager):**
  - Worker token: `/root/swarm_worker_token.txt` on Proxmox  
  - Manager token: `/root/swarm_manager_token.txt` on Proxmox  
  - Manager IP: `/root/swarm_manager_ip.txt` on Proxmox  
- Workers or additional managers use `--autojoin` to automatically join the swarm.  
- ZFS volumes and NFS shares are mounted automatically inside the VM.  
- Hostname inside the VM matches the VM name.  
- Docker installed and service enabled automatically.  
- Passwords can now be provided in plaintext; script hashes them for cloud-init.  

--
## Manager usage:

- First manager with init + local volume for swarm state:
    

bash

CopyEdit

`./provision-swarm-vm.sh manager-01 --role manager --init --volume swarmdata,50G`

- Additional manager joining swarm:
    

bash

CopyEdit

`./provision-swarm-vm.sh manager-02 --role manager --autojoin --volume swarmdata,50G`

- Worker with NFS shares + additional volume:
    

bash

CopyEdit

`./provision-swarm-vm.sh worker-01 --role worker --autojoin --nfs tvshows,movies --volume plexconfig,20G`

## Worker Usage

./provision_swarm_vm.sh <vm-name> --autojoin --volume configs,20G,zfs --nfs media


./provision-swarm-vm.sh  plex-01 \
  --autojoin \
  --volume plex-config,100G,zfs \
  --nfs media/TVShows,media/Movies,media/Animation

----

### 🔑 Usage Examples

**1. First manager with ZFS + NFS**

./provision-swarm-vm.sh \
  --name swarm-manager-01 \
  --vlan 60 \
  --first-manager \
  --user dockeradmin \
  --password 'ManagerFace28!' \
  --volume data:50G \


**2. Plex worker with ZFS + NFS**

`./provision-docker-swarm-vm.sh \   --name plex-worker-01 \   --vlan 85 \   --autojoin \   --role worker \   --volume plex-config:20G \   --nfs movies,tv,music`

**3. Additional manager with ZFS volume**

`./provision-docker-swarm-vm.sh \   --name swarm-manager-02 \   --vlan 90 \   --autojoin \   --role manager \   --volume manager-data:100G`

**4. Worker with only NFS mounts**

`./provision-docker-swarm-vm.sh \   --name download-worker-01 \   --vlan 85 \   --autojoin \   --role worker \   --nfs downloads,temp`
