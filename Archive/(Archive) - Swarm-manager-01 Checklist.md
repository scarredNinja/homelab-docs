---
feature: 10 - Projects/Pasted image 20250928183720.png
---
#HomeLabRebuild/Docker #HomeLabRebuild/Proxmox  #HomeLabRebuild/Phase5 #HomeLabRebuild/Virtualisation

# Swarm Worker VM Provisioning Checklist ✅

./provision.sh --autojoin \
  --nfs media,configs \
  --volume db,20G,zfs;configs,10G,zfs;cache,50G,zfs \
  plex-01

## 1. Prepare Parameters
- [x] Choose VM name (e.g., `plex01`) ✅ 2025-09-26
- [x] Decide on VLAN tag (default: 85) ✅ 2025-09-26
- [x] Optional ZFS volume(s) for configs or DBs ✅ 2025-09-26
- [x] Optional NFS shares to mount (media, backups, downloads, etc.) ✅ 2025-09-26
- [x] Decide if VM should auto-join Docker Swarm ✅ 2025-09-26

## 2. Verify Template
- [x] Template ID is correct (e.g., 103) ✅ 2025-09-26
- [x] Template is updated and cleaned (cloud-init, logs, SSH) ✅ 2025-09-26
- [x] Template has SSH access enabled ✅ 2025-09-26

## 3. Run Provisioning Script
- [x] Make script executable: `chmod +x provision_worker.sh` ✅ 2025-09-26
    - [x] Run example command: ✅ 2025-09-26
      ```bash
      ./provision_worker.sh plex01 \
        --autojoin \
        --volume plex-config,20G,zfs \
        --nfs media,extras,trailers,backups
      ```
    - [x] Confirm VM is cloned successfully ✅ 2025-09-26
    - [x] Confirm MAC address and VMID output ✅ 2025-09-26

## 4. Verify Cloud-init
- [x] **Swarm join command added (if `--autojoin`)** ✅ 2025-09-28
	  - **Proxmox-side:**  
	    - [x] Run `qm cloudinit dump <VMID> user` and confirm the `runcmd:` section contains the Docker Swarm join command. ✅ 2025-09-28
	  - **VM-side:**  
		  - [x] Run `docker info | grep "Swarm:"` → should show `Swarm: active`. ✅ 2025-09-28
		    - Note: Confimed not working

## 5. Verify NFS
- [x] **NFS shares mounted under `/mnt/<share>`** ✅ 2025-09-28
	- **VM-side:**  
	    - [x] Run `mount | grep nfs` → NFS mounts should appear. ✅ 2025-09-28
		    - Not working
	    - [x] Run `ls /mnt` → should show share directories (e.g., `/mnt/media`). ✅ 2025-09-28
		    - Not working
	    - [x] If missing, check `/etc/fstab` for correct entries. ✅ 2025-09-28
		    - No Entries

## 6. Start VM
- [x] Boot the VM #HomeLabRebuild/Phase5 ✅ 2025-09-28
    - In Proxmox UI: Select VM → **Start**  
    - Or via CLI:  
      ```bash
      qm start <vmid>
      ```
- [x] Verify SSH access with injected key #HomeLabRebuild/Phase5 ✅ 2025-09-28
    - On your workstation:  
      ```bash
      ssh <username>@<vm-ip>
      ```
    - Ensure it logs in without a password prompt.

---

## 7. Test Docker Integration
- [x] Verify Docker service is running #HomeLabRebuild/Phase5 ✅ 2025-09-28
    - Run:  
      ```bash
      systemctl status docker
      ```  
    - Look for `active (running)`.

- [x] Test container volume access #HomeLabRebuild/Phase5 ✅ 2025-09-30
    - Run test container:  
      ```bash
      docker run --rm -v <volume>:/data busybox ls /data
      ```
    - Replace `<volume>` with one you provisioned.  
    - You should see directory contents or no error.

- [x] Confirm NFS volumes accessible and writable #HomeLabRebuild/Phase5 #HomeLabRebuild/Templates ✅ 2025-09-30
    - Check mounts:  
      ```bash
      mount | grep nfs
      ```  
    - Test write:  
      ```bash
      touch /mnt/media/<share>/testfile && ls /mnt/media/<share>/testfile
      rm /mnt/media/<share>/testfile
      ```

- [x] Confirm ZFS volumes accessible and writable #HomeLabRebuild/Phase5 ✅ 2025-09-30
    - List datasets:  
      ```bash
      zfs list
      ```  
    - Test write in mounted dataset:  
      ```bash
      touch /mnt/zfs/<dataset>/testfile && ls /mnt/zfs/<dataset>/testfile
      rm /mnt/zfs/<dataset>/testfile
      ```


## 8. Errors:

- ![[Pasted image 20250926184533.png]]
- **Update**: Added sudo on useradd and permissions changes
- To Look At: Syntax error:
	  ![[Pasted image 20250928183720.png]]