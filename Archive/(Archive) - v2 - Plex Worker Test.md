---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
#HomeLabRebuild/Docker #HomeLabRebuild/Proxmox  #HomeLabRebuild/Phase5

# Swarm Worker VM Provisioning Checklist ✅

./provision.sh --autojoin \
  --nfs media,configs \
  --volume db,20G,zfs;configs,10G,zfs;cache,50G,zfs \
  plex-01

## 1. Prepare Parameters
- [x] Choose VM name (e.g., `plex01`) ✅ 2025-08-22
- [x] Decide on VLAN tag (default: 85) ✅ 2025-08-22
- [x] Optional ZFS volume(s) for configs or DBs ✅ 2025-08-22
- [x] Optional NFS shares to mount (media, backups, downloads, etc.) ✅ 2025-08-22
- [x] Decide if VM should auto-join Docker Swarm ✅ 2025-08-22

**Name:** plex-01

## 2. Verify Template
- [x] Template ID is correct (e.g., 103) ✅ 2025-08-22
- [x] Template is updated and cleaned (cloud-init, logs, SSH) ✅ 2025-08-22
- [x] Template has SSH access enabled ✅ 2025-08-22

## 3. Run Provisioning Script
- [x] ❌ Make script executable: `chmod +x provision_worker.sh` ✅ 2025-08-22
    - [x] Run example command: ✅ 2025-10-10
      ```bash
      ./provision_worker.sh plex01 \
        --autojoin \
        --volume plex-config,20G,zfs \
        --nfs media,extras,trailers,backups
      ```
    - [x] Confirm VM is cloned successfully ✅ 2025-10-10
    - [x] Confirm MAC address and VMID output ✅ 2025-10-10

## 4. Verify Cloud-init
- [x] ❌ **Swarm join command added (if `--autojoin`)** ✅ 2025-10-10
	  - **Proxmox-side:**  
	    -  Run `qm cloudinit dump <VMID> user` and confirm the `runcmd:` section contains the Docker Swarm join command. 
	  - **VM-side:**  
	    -  Run `docker info | grep "Swarm:"` → should show `Swarm: active`. 
		    - Note: Confimed not working

## 5. Verify NFS
- [x] ❌ **NFS shares mounted under `/mnt/<share>`** ✅ 2025-10-10
	  - **VM-side:**  
	    -  Run `mount | grep nfs` → NFS mounts should appear.  
		    - Not working
	    -  Run `ls /mnt` → should show share directories (e.g., `/mnt/media`).  
		    - Not working
	    - If missing, check `/etc/fstab` for correct entries.  
		    - No Entries

## 6. Start VM
- [x] Boot the VM #HomeLabRebuild/Phase5 ✅ 2025-08-22
    - In Proxmox UI: Select VM → **Start**  
    - Or via CLI:  
      ```bash
      qm start <vmid>
      ```
- [x] ❌ Verify SSH access with injected key #HomeLabRebuild/Phase5 ✅ 2025-10-10
    - On your workstation:  
      ```bash
      ssh <username>@<vm-ip>
      ```
    - Ensure it logs in without a password prompt.

---

## 7. Test Docker Integration
- [x] ❌ Verify Docker service is running #HomeLabRebuild/Phase5 ✅ 2025-10-10
    - Run:  
      ```bash
      systemctl status docker
      ```  
    - Look for `active (running)`.

- [x] ❌Test container volume access #HomeLabRebuild/Phase5 ✅ 2025-10-10
    - Run test container:  
      ```bash
      docker run --rm -v <volume>:/data busybox ls /data
      ```
    - Replace `<volume>` with one you provisioned.  
    - You should see directory contents or no error.

- [x] ❌Confirm NFS volumes accessible and writable #HomeLabRebuild/Phase5 #Cancelled ✅ 2025-10-10
    - Check mounts:  
      ```bash
      mount | grep nfs
      ```  
    - Test write:  
      ```bash
      touch /mnt/media/<share>/testfile && ls /mnt/media/<share>/testfile
      rm /mnt/media/<share>/testfile
      ```

- [x] ❌Confirm ZFS volumes accessible and writable #HomeLabRebuild/Phase5 ✅ 2025-10-10
    - List datasets:  
      ```bash
      zfs list
      ```  
    - Test write in mounted dataset:  
      ```bash
      touch /mnt/zfs/<dataset>/testfile && ls /mnt/zfs/<dataset>/testfile
      rm /mnt/zfs/<dataset>/testfile
      ```


---

## Issues found

On number 6 it will boot fine but there is an issue getting the static ip address set in pfsense which means the other tasks couldnt be completed anyway.

Confirmed on the switch that VLAN 85 has port 1 and 37 sharing groups as tagged members.

pfsense appears to be setup correctly but need to:
- [x] 🔺  Check vlan connectivity from proxmox #HomeLabRebuild/Phase5 #HomeLabRebuild/Network #HomeLabRebuild/Swarm ✅ 2025-08-24
	- Should I be able to ping 85 and 60 gateway from management/proxmox?
	- Is there a possible issue on the bridge connection on proxmox for the vm?
	- Do i need to test with some test config to check end to end?
	- Are the pfsense rules correct?
