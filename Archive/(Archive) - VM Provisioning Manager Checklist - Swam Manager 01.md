---
feature: 10 - Projects/Pasted image 20250911222828.png
---
#HomeLabRebuild/Docker #HomeLabRebuild/Proxmox  #HomeLabRebuild/Phase5 #HomeLabRebuild/Virtualisation

# Swarm Worker VM Provisioning Checklist ✅

./provision-swarm-vm.sh --name swarm-manager-01 \
--vlan 60 --first-manager \
--user dockeradmin --password 'ManagerFace28!' \
--volume data:50GB --zfs-pool rpool \
--boot-timeout 90 --no-cleanup

## 1. Prepare Parameters
- [x] Choose VM name (e.g., `plex01`) ✅ 2025-09-08
- [x] Decide on VLAN tag (default: 85) ✅ 2025-09-08
- [x] Optional ZFS volume(s) for configs or DBs ✅ 2025-09-08
- [x] Optional NFS shares to mount (media, backups, downloads, etc.) ✅ 2025-09-08
- [x] Decide if VM should auto-join Docker Swarm ✅ 2025-09-24

## 2. Verify Template
- [x] Template ID is correct (e.g., 103) ✅ 2025-09-08
- [x] Template is updated and cleaned (cloud-init, logs, SSH) ✅ 2025-09-08
- [x] Template has SSH access enabled ✅ 2025-09-08

## 3. Run Provisioning Script
- [x] Make script executable: `chmod +x provision_worker.sh` ✅ 2025-09-08

    - [x] Confirm VM is cloned successfully ✅ 2025-09-08
    - [x] Confirm MAC address and VMID output ✅ 2025-09-08

## Debug Next

- [x] Not starting in time. Need to extend the timeout a little on boot. #HomeLabRebuild/Debugging ⏫ ⏳ 2025-09-14 ✅ 2025-09-11
- [x] Need to also have an option to clean up on failure in case I need to debug #HomeLabRebuild/Debugging ⏫ ⏳ 2025-09-14 ✅ 2025-09-11
- [x] Output the Manager and work token as part of the script #HomeLabRebuild/Debugging 🔼 ⏳ 2025-09-14 ✅ 2025-09-11\
- [x] Look into why its not picking up the ip on quest-agent #HomeLabRebuild/Debugging ⏫ ⏳ 2025-09-12 ✅ 2025-09-12
- [x] ![[Pasted image 20250911222828.png]] ✅ 2025-09-26
Debug ssh issue and  correct password authentication prompt as this may have been missed: 
      ![[Pasted image 20250912223006.png]]
