---
project_id: Homelab-2025
status: Reference
phase: 'Phase 4: Storage Config'
tags:
  - reference
  - storage
  - nfs
  - synology
---
# Synology NAS NFS Configuration Rebuild Guide

Following a Synology OS reset (system partition redo only; data volumes and shared folders remain intact), the NFS export permissions, IP routing, and client privileges must be restored to re-establish stable connections for the VMs and containers in the Docker Swarm homelab.

---

## 1. Network & IP Configuration

To ensure that internal network routing, backup traffic, and reverse-proxy configurations continue to work, configure the Synology NAS network interfaces with the following static IPs:

* **NIC 1 (VLAN 100 - Storage):** Static IP **`10.0.100.20`**
  * *Purpose:* Dedicated high-speed bulk media storage traffic.
* **NIC 2 (VLAN 60 - Infrastructure):** Static IP **`10.0.60.45`**
  * *Purpose:* Backups and Traefik DSM web interface access (`http://10.0.60.45:5000`).

---

## 2. NFS Service & Export Permissions

### Step A: Enable NFS Service
* Navigate to **DSM Control Panel → File Services → NFS**.
* Check **Enable NFS service**.
* Support both **NFSv3** and **NFSv4** (restic-rest-server mounts via `nfsvers=3`, Proxmox `Virtio-FS` documentation mounts via `nfsvers=4`).

### Step B: Configure NFS Export Permissions
For each shared folder (`movies`, `tvshows`, `animation`, `docker-backups`), navigate to **Control Panel → Shared Folder → [Select Folder] → Edit → NFS Permissions** and configure the client rules:

#### Folder: `movies` / `tvshows` / `animation`
Since VM client mounts target `10.0.100.20`, connection requests originate from the VMs' **VLAN 100 Storage interface IPs** (not their primary VLAN 50 IPs). Add the following client rules:
1. **Client: `10.0.100.40`** (`worker-media-01` Plex VM - VLAN 100 Storage Interface)
   * **Privilege:** Read/Write (or Read-Only if restricting at the export level)
   * **Squash:** Map all users to admin
   * **Asynchronous:** Yes
   * **Non-privileged port:** Denied
   * **Cross-mount:** Allowed
2. **Client: `10.0.100.41`** (`worker-mediamanagement-01` Arr VM - VLAN 100 Storage Interface)
   * **Privilege:** Read/Write (`rw`)
   * **Squash:** Map all users to admin
   * **Asynchronous:** Yes
   * **Non-privileged port:** Denied
   * **Cross-mount:** Allowed

#### Folder: `docker-backups`
1. **Client: `10.0.60.0/24`** (Monitoring / Docker Swarm VM subnet)
   * **Privilege:** Read/Write (`rw`)
   * **Squash:** No mapping (`no_root_squash`)
   * **Security:** sys
   * **Asynchronous:** Yes
   * *Note:* Used by `restic-rest-server` running on `worker-monitoring-01` (`10.0.60.41`).
2. **Client: `10.0.90.0/25`** (Proxmox host VLAN 90)
   * **Privilege:** Read-Only (`ro`) or Read/Write
   * **Squash:** No mapping
   * **Security:** sys
   * **Asynchronous:** Yes
   * *Note:* Required for the hourly `disk-space-check.sh` script on the Proxmox host.

---

## 3. SSH & Backup Agent Setup

Nightly cron tasks and backup synchronization utilize SSH access to the Synology NAS.
* Enable SSH Service under **DSM Control Panel → Terminal & SNMP**.
* Re-create the `backup-agent` user on the Synology NAS if it was deleted.
* Re-add the Proxmox host's public key (stored at `/root/.ssh/id_backup_synology`) to:
  `/volume1/homes/backup-agent/.ssh/authorized_keys` (ensure permissions are `0700` for `.ssh` and `0600` for `authorized_keys`).

---

## 4. Recovering from Stale NFS Mounts on Clients

Because the Synology OS was reset, active mount points on the VM clients will be in a **stale** state. You must clear them to resume normal operations.

### For `worker-media-01` (Plex VM)
`worker-media-01` mounts NFS shares via `autofs` triggers on demand.
1. **SSH into `worker-media-01`** (`10.0.50.50`).
2. **Force unmount any stale mountpoints:**
   ```bash
   sudo umount -f -l /mnt/media/Movies
   sudo umount -f -l /mnt/media/TVShows
   sudo umount -f -l /mnt/media/Animation
   ```
3. **Restart the autofs service:**
   ```bash
   sudo systemctl restart autofs
   ```
4. **Trigger re-mount:**
   ```bash
   ls /mnt/media/Movies
   ```
5. **Force update the Plex Swarm service** to clear any internal stale mount references inside the container:
   ```bash
   docker service update --force plex_plex
   ```

### For `worker-mediamanagement-01` (Arr VM)
1. **SSH into `worker-mediamanagement-01`** (`10.0.50.51`).
2. **Force unmount stale `/mnt/media`:**
   ```bash
   sudo umount -f -l /mnt/media
   ```
3. **Re-mount:**
   ```bash
   sudo mount -a
   ```
4. **Force update the Arr Swarm service** (Sonarr, Radarr, etc.) to refresh containers:
   ```bash
   docker service update --force arr_sonarr
   docker service update --force arr_radarr
   ```
   *Also restart plain Docker Compose containers if needed:*
   ```bash
   cd /opt/compose-vpn
   sudo docker compose restart
   ```

### For `worker-monitoring-01` (Restic Server VM)
1. **Force update the restic stack** in Portainer or via CLI:
   ```bash
   docker service update --force restic_restic-rest-server
   ```
2. If restic client lock error occurs on Proxmox host, clear it:
   ```bash
   restic -r rest:http://10.0.60.41:8000/ --password-file /etc/restic-password unlock --remove-all
   ```
