---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
# Swarm Worker VM Provisioning Checklist ✅

./provision-swarm-vm.sh \
  --name swarm-manager-01 \
  --vlan 60 \
  --first-manager \
  --user dockeradmin \
  --password 'ManagerFace28!' \
  --volume data:50G \
  --zfs-pool rpool

**Status:** Failed

---

## Debugging Tasks

- [x] Run the ZFS disk work separatly to figure out the issue #HomeLabRebuild/Phase5 🆔 DockerSwarmTemplate ⏫ ✅ 2025-09-08
	- - **Get a VM ID**: Choose a VM that you want to attach a disk to. For this example, let's use `201`.
    
	- **Determine the SCSI counter**: Run `qm config <vmid>` to see how many SCSI disks are already attached to the VM. The next available slot will be `scsi` followed by the next number. For example, if you see `scsi0`, the next one is `scsi1`.
    
	- **Run the `qm set` command manually**: From the Proxmox command line, run the following command, replacing `<vmid>`, `<scsicounter>`, and `<size>` with your values:
		- qm set <vmid> --scsi<scsicounter> <storage>:<size>G,discard=on
		- qm set 101 --scsi1 local-zfs:50G,discard=on

---

## 1. Prepare Parameters
- [x] Choose VM name (e.g., `plex01`) ✅ 2025-08-31
- [x] Decide on VLAN tag (default: 85) ✅ 2025-08-31
- [x] Optional ZFS volume(s) for configs or DBs ✅ 2025-08-31
- [x] Optional NFS shares to mount (media, backups, downloads, etc.) ✅ 2025-08-31
- [x] Decide if VM should auto-join Docker Swarm ✅ 2025-08-31

## 2. Verify Template
- [x] Template ID is correct (e.g., 103) ✅ 2025-08-31
- [x] Template is updated and cleaned (cloud-init, logs, SSH) ✅ 2025-08-31
- [x] Template has SSH access enabled ✅ 2025-08-31

## 3. Run Provisioning Script
- [x] Make script executable: `chmod +x provision_worker.sh` ✅ 2025-09-08
    - [x] Run example command: ✅ 2025-08-31
      ```bash
      ./provision_worker.sh plex01 \
        --autojoin \
        --volume plex-config,20G,zfs \
        --nfs media,extras,trailers,backups
      ```
    -  Confirm VM is cloned successfully 
    -  Confirm MAC address and VMID output

## 4. Verify Cloud-init
- [x] **Swarm join command added (if `--autojoin`)** ✅ 2025-09-08
	  - **Proxmox-side:**  
	    - Run `qm cloudinit dump <VMID> user` and confirm the `runcmd:` section contains the Docker Swarm join command. 
	  - **VM-side:**  
	    - Run `docker info | grep "Swarm:"` → should show `Swarm: active`. 
		    - Note: Confimed not working

## 5. Verify NFS
- [x] **NFS shares mounted under `/mnt/<share>`** ✅ 2025-09-08
	  - **VM-side:**  
	    -  Run `mount | grep nfs` → NFS mounts should appear.  
		    - Not working
	    - Run `ls /mnt` → should show share directories (e.g., `/mnt/media`).  
		    - Not working
	    -  If missing, check `/etc/fstab` for correct entries.  
		    - No Entries

## 6. Start VM
- [x] Boot the VM #HomeLabRebuild/Phase5 ✅ 2025-09-08
    - In Proxmox UI: Select VM → **Start**  
    - Or via CLI:  
      ```bash
      qm start <vmid>
      ```
- [x] Verify SSH access with injected key #HomeLabRebuild/Phase5 ✅ 2025-09-08
    - On your workstation:  
      ```bash
      ssh <username>@<vm-ip>
      ```
    - Ensure it logs in without a password prompt.

---

## 7. Test Docker Integration
- [x] Verify Docker service is running #HomeLabRebuild/Phase5 ✅ 2025-09-08
    - Run:  
      ```bash
      systemctl status docker
      ```  
    - Look for `active (running)`.

- [x] Test container volume access #HomeLabRebuild/Phase5 ✅ 2025-09-08
    - Run test container:  
      ```bash
      docker run --rm -v <volume>:/data busybox ls /data
      ```
    - Replace `<volume>` with one you provisioned.  
    - You should see directory contents or no error.

- [x] Confirm NFS volumes accessible and writable #HomeLabRebuild/Phase5 ✅ 2025-10-10
    - Check mounts:  
      ```bash
      mount | grep nfs
      ```  
    - Test write:  
      ```bash
      touch /mnt/media/<share>/testfile && ls /mnt/media/<share>/testfile
      rm /mnt/media/<share>/testfile
      ```

- [x] Confirm ZFS volumes accessible and writable #HomeLabRebuild/Phase5 ✅ 2025-10-10
    - List datasets:  
      ```bash
      zfs list
      ```  
    - Test write in mounted dataset:  
      ```bash
      touch /mnt/zfs/<dataset>/testfile && ls /mnt/zfs/<dataset>/testfile
      rm /mnt/zfs/<dataset>/testfile
      ```


## 8. Optional Post-Provision Steps
- [x] Install additional packages needed by the worker ✅ 2025-10-10
- [x] Configure monitoring or backups ✅ 2025-10-10
- [x] Document VM and attached volumes in cluster map ✅ 2025-10-10
