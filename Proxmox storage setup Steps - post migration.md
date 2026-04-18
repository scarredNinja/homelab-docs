---
project_id: Homelab-2025
phase: "Phase 4: Storage Config"
tags:
  - Storage
  - proxmox
  - migration
---

## 🟦 Prepare the Target ZFS Structure

- [x] **Create the ZFS Datasets** (Run on Proxmox Host):  [priority:: 1] ✅ 2026-02-05
    
    Bash
    
    ```    
    # Relational Database
	zfs create -o recordsize=16k rpool/docker-db
    
    # Monitoring/TSDB Dataset
	zfs create -o recordsize=16k -o logbias=throughput rpool/docker-tsdb

	# Media Dataset (Plex)
	zfs create -o recordsize=1M -o atime=off rpool/media
    
    # Create specific children for your apps - look at others to add
    zfs create rpool/docker-data/traefik
    zfs create rpool/docker-data/portainer
    zfs create rpool/docker-data/netbox
    zfs create rpool/docker-data/sonarr
    zfs create rpool/docker-data/radarr
    zfs create rpool/docker-data/transmission
    zfs create rpool/docker-data/prowlarr
    zfs create rpool/docker-data/unifi
    zfs create rpool/docker-data/grafana
    zfs create rpool/docker-data/tautulli
    zfs create rpool/docker-data/backups
    zfs create rpool/docker-data/homeassistant
    
    ```

---

## 🛠 Proxmox Virtio-FS Setup (The "Cleaner" Way)

This method allows your VM to see the host's ZFS dataset as a local folder. It is faster than NFS and much easier to manage for Docker volumes.

### 1. On the Proxmox Host (Shell)

First, make sure the tool that handles the sharing is installed and your ZFS dataset is mapped in Proxmox.
    
- [x] **Create a Directory Mapping:** In the Proxmox Web UI, go to **Datacenter > Directory Mappings**. [priority:: 1] ✅ 2026-02-05
    
    - Add a new mapping.
        
    - **ID:** `docker-data`
        
    - **Path:** `/mnt/rpool/docker-data` (or wherever your ZFS dataset is).

**Note: ** Look at [[ZFS Configuration and Setup]]

### 2. Connect the Mapping to the VM

- [x] Go to your **Manager VM > Hardware > Add > Virtio-FS**. ✅ 2026-02-05
    
- [x] Select the  ID you just created. **From Above**** ✅ 2026-02-05
    

---

## 📂 The "Manual Migration" (Internal to the VM)

Since we aren't doing the "Host-Pull," you will move the data **inside the VM** once the new storage is attached.

### 1. Mount the Virtio-FS inside the VM

Inside your Manager VM, create a folder and mount the new shared storage.

Bash

```
# Create the permanent mount point
mkdir -p /mnt/shared-storage

# Mount it (Temporary test)
mount -t virtiofs docker-data /mnt/shared-storage
```

### 2. Make it Permanent

Add the mount to your `/etc/fstab` inside the VM so it survives a reboot.

Plaintext

```
docker-data  /mnt/data  virtiofs  defaults  0  0
docker-tsdb  /mnt/tsdb  virtiofs  defaults  0  0
media        /mnt/media virtiofs  defaults  0  0
```

---