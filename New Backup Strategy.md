#Storage

### **Backup Strategy**

- Since app data backups go daily to your Synology NAS, you get extra protection.
    
- Proxmox backups (snapshots or vzdump) should target a separate backup storage (e.g., Synology NFS or SMB share).
    
- OS backups can be done via Proxmox backup tools or periodic image backups.


# Backup Configuration for Proxmox to Synology NAS

### Prepare Synology NAS

- Enable NFS or SMB share on Synology for backups.
    
- For NFS:
    
    - Create a shared folder (e.g., `proxmox_backups`).
        
    - Enable NFS and allow your Proxmox server IP access.
        

### Mount Synology Share on Proxmox

**For NFS:**

1. Create mount point on Proxmox:
    

bash

CopyEdit

`mkdir -p /mnt/synology_backup`

2. Mount the NFS share:
    

bash

CopyEdit

`mount -t nfs <synology_ip>:/volume1/proxmox_backups /mnt/synology_backup`

3. To mount automatically on boot, edit `/etc/fstab` and add:
    

ruby

CopyEdit

`<synology_ip>:/volume1/proxmox_backups  /mnt/synology_backup  nfs  defaults  0  0`

### Add Backup Storage in Proxmox GUI

1. Go to **Datacenter > Storage > Add > Directory**.
    
2. Set:
    
    - ID: `synology_backup`
        
    - Directory: `/mnt/synology_backup`
        
    - Content: `Backup`
        
    - Enable `Shared` if you have multiple nodes.
        
3. Save.
    

### Schedule Backups

1. Go to **Datacenter > Backup**.
    
2. Create a new backup job targeting the `synology_backup` storage.
    
3. Schedule daily backups, select VMs/LXCs to back up.
    
4. Set retention and compression options as desired.