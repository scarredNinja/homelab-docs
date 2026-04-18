# Swarm Worker VM Provisioning Checklist ✅

./provision.sh --autojoin \
  --nfs media,configs \
  --volume db,20G,zfs;configs,10G,zfs;cache,50G,zfs \
  plex-01

## 1. Prepare Parameters
- [ ] Choose VM name (e.g., `plex01`) 
- [ ] Decide on VLAN tag (default: 85) 
- [ ] Optional ZFS volume(s) for configs or DBs 
- [ ] Optional NFS shares to mount (media, backups, downloads, etc.) 
- [ ] Decide if VM should auto-join Docker Swarm 

## 2. Verify Template
- [ ] Template ID is correct (e.g., 103) 
- [ ] Template is updated and cleaned (cloud-init, logs, SSH) 
- [ ] Template has SSH access enabled 

## 3. Run Provisioning Script
- [ ] Make script executable: `chmod +x provision_worker.sh`
    - [ ] Run example command: 
      ```bash
      ./provision_worker.sh plex01 \
        --autojoin \
        --volume plex-config,20G,zfs \
        --nfs media,extras,trailers,backups
      ```
    - [ ] Confirm VM is cloned successfully 
    - [ ] Confirm MAC address and VMID output

## 4. Verify Cloud-init
- [ ] **Swarm join command added (if `--autojoin`)**  
	  - **Proxmox-side:**  
	    - [ ] Run `qm cloudinit dump <VMID> user` and confirm the `runcmd:` section contains the Docker Swarm join command. 
	  - **VM-side:**  
	    - [  ] Run `docker info | grep "Swarm:"` → should show `Swarm: active`. 
		    - Note: Confimed not working

## 5. Verify NFS
- [ ] **NFS shares mounted under `/mnt/<share>`**   
	  - **VM-side:**  
		- [ ] Run `mount | grep nfs` → NFS mounts should appear.  
		    - Not working
	    - [ ] Run `ls /mnt` → should show share directories (e.g., `/mnt/media`).  
		    - Not working
	    - [ ] If missing, check `/etc/fstab` for correct entries.  
		    - No Entries

## 6. Start VM
- [ ] Boot the VM
    - In Proxmox UI: Select VM → **Start**  
    - Or via CLI:  
      ```bash
      qm start <vmid>
      ```
- [ ] Verify SSH access with injected key
    - On your workstation:  
      ```bash
      ssh <username>@<vm-ip>
      ```
    - Ensure it logs in without a password prompt.

---

## 7. Test Docker Integration
- [ ] Verify Docker service is running 
    - Run:  
      ```bash
      systemctl status docker
      ```  
    - Look for `active (running)`.

- [ ] Test container volume access 
    - Run test container:  
      ```bash
      docker run --rm -v <volume>:/data busybox ls /data
      ```
    - Replace `<volume>` with one you provisioned.  
    - You should see directory contents or no error.

- [ ] Confirm NFS volumes accessible and writable #Templates
    - Check mounts:  
      ```bash
      mount | grep nfs
      ```  
    - Test write:  
      ```bash
      touch /mnt/media/<share>/testfile && ls /mnt/media/<share>/testfile
      rm /mnt/media/<share>/testfile
      ```

- [ ] Confirm ZFS volumes accessible and writable 
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