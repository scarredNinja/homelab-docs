#HomeLabRebuild/Docker #HomeLabRebuild/Proxmox  #HomeLabRebuild/Phase5

# Swarm Worker VM Provisioning Checklist ✅

## 1. Prepare Parameters
- [x] Choose VM name (e.g., `plex01`) ✅ 2025-08-22
- [x] Decide on VLAN tag (default: 85) ✅ 2025-08-22
- [x] Optional ZFS volume(s) for configs or DBs ✅ 2025-08-22
- [x] Optional NFS shares to mount (media, backups, downloads, etc.) ✅ 2025-08-22
- [x] Decide if VM should auto-join Docker Swarm ✅ 2025-08-22

## 2. Verify Template
- [x] Template ID is correct (e.g., 103) ✅ 2025-08-22
- [x] Template is updated and cleaned (cloud-init, logs, SSH) ✅ 2025-08-22
- [x] Template has SSH access enabled ✅ 2025-08-22

## 3. Run Provisioning Script
- [x] Make script executable: `chmod +x provision_worker.sh` ✅ 2025-08-22
    - [ ] Run example command: 
      ```bash
      ./provision-swarm-worker.sh --autojoin \
  --nfs 0.0.60.80:/volume1/movies,10.0.60.80:/volume1/tvshows, 10.0.60.80:/volume1/animation \
  --volume configs,100G,zfs \
  plex-01
    - [ ] Confirm VM is cloned successfully 
    - [ ] Confirm MAC address and VMID output

## 4. Verify Cloud-init
- [x] **Swarm join command added (if `--autojoin`)** ✅ 2025-08-22
	  - **Proxmox-side:**  
	    - [x] Run `qm cloudinit dump <VMID> user` and confirm the `runcmd:` section contains the Docker Swarm join command. ✅ 2025-08-22
	  - **VM-side:**  
	    - [ ] ❌  Run `docker info | grep "Swarm:"` → should show `Swarm: active`. 
		    - Note: Confimed not working

## 5. Verify NFS
- [x] **NFS shares mounted under `/mnt/<share>`** ✅ 2025-08-22
	  - **VM-side:**  
	    - [ ] ❌ Run `mount | grep nfs` → NFS mounts should appear.  
		    
	    - [ ] ❌ Run `ls /mnt` → should show share directories (e.g., `/mnt/media`).  
		    
	    - [ ] ❌ If missing, check `/etc/fstab` for correct entries.  

## 6. Start VM
- [x] Boot the VM #HomeLabRebuild/Phase5 ✅ 2025-08-22
    - In Proxmox UI: Select VM → **Start**  
    - Or via CLI:  
      ```bash
      qm start <vmid>
      ```
- [x] Verify SSH access with injected key #HomeLabRebuild/Phase5 ✅ 2025-08-22
    - On your workstation:  
      ```bash
      ssh <username>@<vm-ip>
      ```
    - Ensure it logs in without a password prompt.

---

## 7. Test Docker Integration
- [x] Verify Docker service is running #HomeLabRebuild/Phase5 ✅ 2025-08-22
    - Run:  
      ```bash
      systemctl status docker
      ```  
    - Look for `active (running)`.

- [x] Test container volume access #HomeLabRebuild/Phase5 ✅ 2025-08-22
    - Run test container:  
      ```bash
      docker run --rm -v <volume>:/data busybox ls /data
      ```
    - Replace `<volume>` with one you provisioned.  
    - You should see directory contents or no error.

- [x] Confirm NFS volumes accessible and writable #HomeLabRebuild/Phase5 ✅ 2025-08-22
    - Check mounts:  
      ```bash
      mount | grep nfs
      ```  
    - Test write:  
      ```bash
      touch /mnt/media/<share>/testfile && ls /mnt/media/<share>/testfile
      rm /mnt/media/<share>/testfile
      ```

- [x] Confirm ZFS volumes accessible and writable #HomeLabRebuild/Phase5 ✅ 2025-08-22
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
- [ ] Install additional packages needed by the worker 
- [ ] Configure monitoring or backups 
- [ ] Document VM and attached volumes in cluster map 