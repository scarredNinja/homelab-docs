#HomeLabRebuild/Docker #HomeLabRebuild/Proxmox  #HomeLabRebuild/Phase5

# Swarm Worker VM Provisioning Checklist ✅

## 1. Prepare Parameters
- [x] Choose VM name (e.g., `plex01`) ✅ 2025-08-22 #HomeLabRebuild/Phase5
- [x] Decide on VLAN tag (default: 85) ✅ 2025-08-22 #HomeLabRebuild/Phase5
- [x] Optional ZFS volume(s) for configs or DBs ✅ 2025-08-22 #HomeLabRebuild/Phase5
- [x] Optional NFS shares to mount (media, backups, downloads, etc.) ✅ 2025-08-22 #HomeLabRebuild/Phase5
- [x] Decide if VM should auto-join Docker Swarm ✅ 2025-08-22 #HomeLabRebuild/Phase5

## 2. Verify Template
- [x] Template ID is correct (e.g., 103) ✅ 2025-08-22 #HomeLabRebuild/Phase5
- [x] Template is updated and cleaned (cloud-init, logs, SSH) ✅ 2025-08-22 #HomeLabRebuild/Phase5
- [x] Template has SSH access enabled ✅ 2025-08-22 #HomeLabRebuild/Phase5

## 3. Run Provisioning Script
- [x] Make script executable: `chmod +x provision_worker.sh` ✅ 2025-08-22
    - [x] Run example command: #HomeLabRebuild/Phase5 ✅ 2025-10-10
      ```bash
      ./provision_worker.sh plex01 \
        --autojoin \
        --volume plex-config,20G,zfs \
        --nfs media,extras,trailers,backups
      ```
    - [x] Confirm VM is cloned successfully #HomeLabRebuild/Phase5 ✅ 2025-08-27
    - [x] Confirm MAC address and VMID output#HomeLabRebuild/Phase5 ✅ 2025-10-10

**VM Name: ** plex-01

## 4. Verify Cloud-init
- [x] ❌ **Swarm join command added (if `--autojoin`)** ✅ 2025-10-10
	  - ❌ **Proxmox-side:**  
	    - [x] Run `qm cloudinit dump <VMID> user` and confirm the `runcmd:` section contains the Docker Swarm join command. #HomeLabRebuild/Phase5 ✅ 2025-10-10
		    - Output: ![[Pasted image 20250822174647.png]]
	  -  ❌**VM-side:**  
	    - [x] Run `docker info | grep "Swarm:"` → should show `Swarm: active`. #HomeLabRebuild/Phase5 ✅ 2025-10-10
		    - Note: Confirmed not working
		    - ![[Pasted image 20250822174746.png]]

## 5. Verify NFS
- [x] **NFS shares mounted under `/mnt/<share>`** ✅ 2025-10-10
	  - **VM-side:**  
	    - [x] ❌ Run `mount | grep nfs` → NFS mounts should appear. ✅ 2025-10-10
		    - **Not working**
	    - [x] ❌ Run `ls /mnt` → should show share directories (e.g., `/mnt/media`). ✅ 2025-10-10
		    - **Not working**
	    - [x] ❌ If missing, check `/etc/fstab` for correct entries. ✅ 2025-10-10
		    - **No Entries**
	

## 6. Start VM
- [x] **[Proxmox]** Boot the VM #HomeLabRebuild/Phase5 ✅ 2025-08-22
    - In Proxmox UI: Select VM → **Start**  
    - Or via CLI:  
      ```bash
      qm start <vmid>
      ```

- [x] ❌ **[VM]** Verify SSH access with injected key #HomeLabRebuild/Phase5 ✅ 2025-10-10
    - On your workstation:  
      ```bash
      ssh <username>@<vm-ip>
      ```  
    - Ensure it logs in without a password prompt.

---

## 7. Test Docker Integration
- [x] **[VM]** Verify Docker service is running #HomeLabRebuild/Phase5 ✅ 2025-08-22
    - Run:  
      ```bash
      systemctl status docker
      ```  
    - Look for `active (running)`.

- [x] ❌ **[VM]** Test container volume access #HomeLabRebuild/Phase5 ✅ 2025-10-10
    - Run test container:  
      ```bash
      docker run --rm -v <volume>:/data busybox ls /data
      ```
    - Replace `<volume>` with one you provisioned.  
    - You should see directory contents or no error.

- [x] ❌ **[VM]** Confirm NFS volumes accessible and writable #HomeLabRebuild/Phase5 ✅ 2025-10-10
    - Check mounts:  
      ```bash
      mount | grep nfs
      ```  
    - Test write:  
      ```bash
      touch /mnt/media/<share>/testfile && ls /mnt/media/<share>/testfile
      rm /mnt/media/<share>/testfile
      ```



- [x] ❌ **[VM]** Confirm ZFS volumes accessible and writable #HomeLabRebuild/Phase5 ✅ 2025-10-10
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

Working on in new checklist. Closed
