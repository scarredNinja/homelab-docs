---
project_id: Homelab-2025
phase: 'Phase 4: Storage Config'
tags:
  - Storage
  - proxmox
  - migration
status: Reference
---
# 🔄 Same-Disk Migration: Default to ZFS

## 📌 Pre-Migration Check

> [!WARNING] Before moving data on the same physical disk, ensure you have enough free space. Moving data between partitions/datasets requires a temporary "double-up" of space unless you delete as you go (not recommended for safety).

---

## 🟦 Step 1: Prepare the Target ZFS Structure

Even though the data is already on the SSD, it is likely in `/var/lib/docker` or a generic folder. We want to move it into dedicated ZFS datasets to get the benefits of compression and snapshots.

- [x] **Create the ZFS Datasets** (Run on Proxmox Host): ✅ 2026-02-03
    
    Bash
    
    ```
    # Create the parent
    zfs create -o compression=lz4 tank/docker-data
    
    # Monitoring/TSDB Dataset
	zfs create -o recordsize=16k -o logbias=throughput tank/docker-tsdb

	# Media Dataset (Plex)
	zfs create -o recordsize=1M -o atime=off tank/media
    
    # Create specific children for your apps
    zfs create rpool/docker-data/traefik
    zfs create rpool/docker-data/portainer
    zfs create rpool/docker-data/netbox
    ```

---

## 🛠 Proxmox Virtio-FS Setup (The "Cleaner" Way)

This method allows your VM to see the host's ZFS dataset as a local folder. It is faster than NFS and much easier to manage for Docker volumes.

### 1. On the Proxmox Host (Shell)

First, make sure the tool that handles the sharing is installed and your ZFS dataset is mapped in Proxmox.

- [x] **Install virtiofsd:** `apt update && apt install virtiofsd -y` ✅ 2026-02-03
    
- [x] **Create a Directory Mapping:** In the Proxmox Web UI, go to **Datacenter > Directory Mappings**. ✅ 2026-02-03
    
    - Add a new mapping.
        
    - **ID:** `docker-data`
        
    - **Path:** `/mnt/tank/docker-data` (or wherever your ZFS dataset is).
        

### 2. Connect the Mapping to the VM

- [x] Go to your **Manager VM > Hardware > Add > Virtio-FS**. ✅ 2026-02-03
    
- [x] Select the `docker-data` ID you just created. ✅ 2026-02-03
    

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

### 2. Move the Data (The Core Migration)

Now you can move your files from the "Invisible" Docker volumes to the "Visible" Virtio-FS folder.

Bash

```
# Stop your stacks first!
docker stack rm <stack_name>

# Move the data
rsync -avP /var/lib/docker/volumes/netbox_data/_data/ /mnt/shared-storage/netbox/
rsync -avP /var/lib/docker/volumes/portainer_data/_data/ /mnt/shared-storage/portainer/
```

### 3. Make it Permanent

Add the mount to your `/etc/fstab` inside the VM so it survives a reboot.

Plaintext

```
docker-data  /mnt/shared-storage  virtiofs  defaults  0  0
```

---

## 📌 Updated Obsidian Checklist: Migration Final Form

> [!SUCCESS] **Why this is better:** Virtio-FS doesn't require a network stack, so it has lower latency than NFS. It also means your backups (PBS) will see the data directly on the host's ZFS disk.

- [x] **Host Setup:** Install `virtiofsd` and create Directory Mapping in Datacenter. ✅ 2026-02-03
    
- [x] **VM Hardware:** Add Virtio-FS device to VM. ✅ 2026-02-03
    
- [x] **Internal Migration:** ✅ 2026-02-03
    
    - [x] `mkdir /mnt/shared-storage` ✅ 2026-02-03
        
    - [x] `mount -t virtiofs ...` ✅ 2026-02-03
        
    - [x] `rsync` old volumes to new mount. ✅ 2026-02-03
        
- [x] **Docker Pivot:** Update Portainer stacks to use `/mnt/docker-data/app-name` instead of named volumes. ✅ 2026-02-03
