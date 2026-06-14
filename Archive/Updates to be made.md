---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
# Provisioning Script Updates – Swarm Managers Storage

## Purpose
Prepare the provisioning script to handle the updated storage plan for Swarm managers:
- NFS for critical configs / HA state
- Local ZFS storage for backups and logs
- Consistent paths and permissions across all three managers

---

## Planned Changes

### 1️⃣ NFS Mounts for Config / HA **- Done**
- Create NFS mount points on all manager nodes:
  - `/mnt/portainer`
  - `/mnt/grafana`

- Mount NFS shares from Synology (`10.0.60.80`) at the above paths.
- Ensure **UID/GID and permissions** match the Docker user.
- These mounts support HA services and persistent configs so containers can move freely between managers.

### 2️⃣ Local Directories for Backups / Logs - **Done**
- Create local ZFS directories on each manager node:
  - `/mnt/backups`
  - `/mnt/logs`
- Bind these paths into containers where necessary.
- Keeps high-I/O operations local and off the NFS server.

### 3️⃣ Docker Volumes / Service Config - Done
- Update Docker volume creation in the script:
  - **Config volumes → NFS paths**
  - **Backup / log volumes → local paths**
- Ensures proper persistence and HA across manager nodes.

### 4️⃣ Verification Steps
- Check that all NFS mounts are **active and writable** before starting services.
- Ensure local backup/log paths exist and have correct ownership.
- Optional: script can alert or abort if mounts/permissions are incorrect.

---

## Notes
- Ephemeral data (temporary logs, caches) should remain on local disks.
- NFS is only for critical persistent data to avoid overloading the NAS.
- These changes allow the Swarm managers to start safely and be ready for HA services.
