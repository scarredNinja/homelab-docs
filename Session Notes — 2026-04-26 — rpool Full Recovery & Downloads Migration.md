---
date: 2026-04-26
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
session_type: debugging, infrastructure, migration
status: in-progress
tags:
  - DockerSwarm
  - ZFS
  - Storage
  - Traefik
  - Downloads
  - Migration
  - Proxmox
---

# Session Notes — 2026-04-26 — rpool Full Recovery & Downloads Migration

> [!abstract] Overview
> Emergency session triggered by rpool running out of space, causing Proxmox host and all VMs to become unresponsive. Root causes identified and resolved: stale ZFS datasets consuming ~580G, downloads disk on SSD instead of HDD. Downloads migrated to MainStorage virtiofs. Old LXC subvolumes cleared. Traefik entrypoint NIC-binding planned.

---

## Session Goal

Recover rpool disk space, restore VM access, and correctly place downloads storage on MainStorage (HDD) rather than rpool (SSD).

---

## Completed

### 1. rpool Full — Diagnosis

- `df -h` on Proxmox host showed every rpool dataset at 100% used / 0B available
- `zpool list rpool` showed 928G pool with 0B free — genuinely full, not a quota issue
- `zfs get quota -r rpool` confirmed no quotas set

**Root causes identified:**

| Dataset | Used | Issue |
|---------|------|-------|
| `rpool/data/vm-204-downloads` | 153G | Downloads on SSD (wrong pool) |
| `rpool/manager_test` | 102G | Stale test dataset |
| `rpool/swarm-manager-01_certs` | 10.2G | Stale old provisioning |
| `rpool/swarm-manager-01_config` | 30.5G | Stale |
| `rpool/swarm-manager-01_data` | 50.8G | Stale |
| `rpool/swarm-manager-01_logs` | 50.8G | Stale |
| `rpool/swarm-mgr1_backups` | 50.8G | Stale |
| `rpool/swarm-mgr1_logs` | 50.8G | Stale |
| `rpool/swarm-mgr2_backups` | 50.8G | Stale |
| `rpool/swarm-mgr2_logs` | 50.8G | Stale |
| `rpool/swarm-mgr3_backups` | 50.8G | Stale |
| `rpool/swarm-mgr3_logs` | 50.8G | Stale |

> [!note] The swarm-mgr* datasets were sparse placeholder datasets — REFER values of 1-2MB vs USED of 50.8G each. Artifact of early provisioning scripts that pre-allocated pool space accounting without writing real data.

### 2. Stale Datasets Destroyed

Confirmed not mounted to any active LXC (`/etc/pve/lxc/` was empty) before destroying:

```bash
zfs destroy rpool/manager_test
zfs destroy rpool/swarm-manager-01_certs
zfs destroy rpool/swarm-manager-01_config
zfs destroy rpool/swarm-manager-01_data
zfs destroy rpool/swarm-manager-01_logs
zfs destroy rpool/swarm-mgr1_backups
zfs destroy rpool/swarm-mgr1_logs
zfs destroy rpool/swarm-mgr2_backups
zfs destroy rpool/swarm-mgr2_logs
zfs destroy rpool/swarm-mgr3_backups
zfs destroy rpool/swarm-mgr3_logs
```

rpool recovered to 577G free. VMs restarted successfully.

### 3. pmxcfs I/O Errors

During recovery, `/etc/pve` FUSE filesystem was throwing I/O errors — `pvesm add` and `qm set` both failing with lock timeout and input/output errors.

```bash
systemctl restart pve-cluster
```

Resolved. Root cause: pmxcfs was already struggling before the disk full event.

> [!bug] See Gotcha #40

### 4. Downloads Storage — Architecture Decision

Downloads should be on **MainStorage (HDD)**, not rpool (SSD):
- Temporary staging data — deleted after import
- Large and unpredictable (4K remuxes = 60-80GB each)
- Write-heavy churn — high ZFS write amplification on SSD
- Sequential I/O — HDDs handle large sequential writes fine

The `vm-204-downloads` 500G zvol existed on `rpool/data` — wrong pool entirely.

### 5. MainStorage/downloads Created

```bash
zfs create MainStorage/downloads
zfs set compression=lz4 MainStorage/downloads
zfs set atime=off MainStorage/downloads
zfs set recordsize=1M MainStorage/downloads
```

### 6. Virtiofs Directory Mapping Added

Discovered virtiofs shares are defined in `/etc/pve/mapping/directory.cfg`, not `storage.cfg`. Added:

```
downloads
        map node=pve,path=/MainStorage/downloads
```

> [!bug] First write attempt failed with I/O error (pmxcfs issue — see above). Entry was written twice due to retry after restart. Required manual cleanup of duplicate in nano.

Attached to VM 204:
```bash
qm set 204 --virtiofs4 downloads
```

### 7. Downloads Migrated — SSD → MainStorage

Inside `worker-mediamanagement-01`:

```bash
# Mounted old SSD disk
mount /dev/sdb /mnt/downloads-old

# Mounted new virtiofs
mkdir -p /mnt/downloads
mount -t virtiofs downloads /mnt/downloads

# Rsynced 153G
rsync -av --progress /mnt/downloads-old/ /mnt/downloads/

# Added to fstab
echo "downloads  /mnt/downloads  virtiofs  defaults  0  0" >> /etc/fstab
```

> [!bug] fstab had three entries after the session — old UUID ext4 entry from the SSD disk plus two virtiofs entries (duplicate from earlier session). Cleaned up manually. See Gotcha #41.

Detached and destroyed the SSD zvol on Proxmox host:
```bash
qm set 204 --delete scsi1
zfs destroy rpool/data/vm-204-downloads
```

rpool recovered additional 500G → total 577G free.

### 8. Transmission Config Verified

`/mnt/docker-data/transmission/settings.json` confirmed:
- `download-dir`: `/downloads/complete`
- `incomplete-dir`: `/downloads/incomplete`

`compose-vpn.yml` volume mapping confirmed correct: `/mnt/downloads:/downloads`

No changes required to Transmission config.

### 9. Prowlarr Configured in Sonarr and Radarr

Prowlarr indexer sync configured in both Sonarr and Radarr via Settings → Indexers → Prowlarr.

### 10. Old LXC Subvolumes Cleared from MainStorage

Confirmed all service data migrated to virtiofs datasets before clearing:
- `/mnt/docker-data/sonarr/` ✅
- `/mnt/docker-data/radarr/` ✅
- `/mnt/docker-data/plex/` ✅
- `/mnt/docker-data/homeassistant/` ✅

Destroyed cleared subvolumes:
```bash
zfs destroy MainStorage/subvol-101-disk-0   # 71.8G — Plex/Tautulli
zfs destroy MainStorage/subvol-102-disk-0   # 4.1G  — HA/Prowlarr
zfs destroy MainStorage/subvol-103-disk-0   # 6.9G  — web/media
zfs destroy MainStorage/subvol-104-disk-0   # 17.3G — Sonarr/Radarr
zfs destroy MainStorage/subvol-105-disk-1   # 30.7G — monitoring stack
zfs destroy MainStorage/subvol-109-disk-0   # 19.6G — media
```

Total ~150G recovered on MainStorage.

### 11. Remaining Downloads Migration (In Progress)

`MainStorage/subvol-110-disk-0` contains 440G of downloads in `/home/downloads/complete/` — the old mediamanagement LXC. These need seeding preserved.

Moving to `MainStorage/downloads` (same pool — mv not rsync but cross-dataset so actual copy):

```bash
mv /MainStorage/subvol-110-disk-0/home/downloads/complete/* \
   /MainStorage/downloads/complete/
```

Still in progress at session end.

### 12. Traefik Entrypoint Issue Identified

Internal services require `:8443` appended to URLs. Root cause: `internal` entrypoint bound to `:8443` on all interfaces instead of `10.0.60.40:443`.

**Current config:**
```yaml
entryPoints:
  websecure:
    address: ':443'      # all interfaces — external
  internal:
    address: ':8443'     # all interfaces — internal
```

**Planned fix** — bind each entrypoint to its specific NIC IP:
```yaml
entryPoints:
  web:
    address: '10.0.80.20:80'
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true
  web-internal:
    address: '10.0.60.40:80'
    http:
      redirections:
        entryPoint:
          to: internal
          scheme: https
  websecure:
    address: '10.0.80.20:443'
    forwardedHeaders:
      trustedIPs: [Cloudflare ranges]
  internal:
    address: '10.0.60.40:443'
```

Also found: most internal services in `infrastructure.yml` are on `websecure` entrypoint — should be `internal`.

Also open: `60-nic2-dmz.yaml` needs `use-routes: false` permanently (currently only applied via temp `ip route del`).

Queued for Claude Code session.

---

## Current State

| Item | Status |
|------|--------|
| rpool free space | 577G free ✅ |
| Downloads on MainStorage | ✅ 153G migrated |
| Old SSD zvol destroyed | ✅ |
| Old LXC subvolumes cleared | ✅ (101-105, 109) |
| subvol-110 downloads migration | 🔄 In progress |
| fstab cleaned up | ✅ |
| Prowlarr → Sonarr/Radarr | ✅ |
| Traefik entrypoint NIC binding | ⏳ Next session |

---

## Next Session Priorities

1. **Verify subvol-110 mv completed** — check `ls /MainStorage/downloads/complete/ | wc -l` and destroy subvol-110 once confirmed
2. **Transmission setup** — start `compose-vpn.yml`, verify downloads land in `/mnt/downloads/complete`
3. **Traefik entrypoint fix** (Claude Code) — bind `websecure` to VLAN 80 NIC, `internal` to VLAN 60 NIC on port 443, fix `infrastructure.yml` entrypoint labels, fix `60-nic2-dmz.yaml` netplan permanently
4. **Deploy arr stack** — Sonarr, Radarr, Prowlarr
5. **Deploy Plex stack** — after media migration confirmed

---

## Bugs Found This Session

> [!bug] Bug #40 — pmxcfs I/O errors under full disk
> **Symptom:** `pvesm add`, `qm set`, file writes to `/etc/pve` all fail with "Input/output error" and lock timeouts
> **Cause:** pmxcfs FUSE filesystem becomes unstable when underlying disk is full
> **Fix:** `systemctl restart pve-cluster` after disk space is recovered
> **See Gotcha #40**

> [!bug] Bug #41 — Duplicate fstab entries after interrupted session
> **Symptom:** Three `downloads` mount entries in `/etc/fstab` — old UUID ext4 (stale SSD), plus two virtiofs entries
> **Cause:** fstab entries added in multiple steps across session; old SSD entry never removed after disk destroyed
> **Fix:** Manual cleanup in nano — keep only `downloads  /mnt/downloads  virtiofs  defaults  0  0`
> **See Gotcha #41**

> [!bug] Bug #42 — Virtiofs directory mapping written twice
> **Symptom:** `cloud-init` error on VM start: "More than one directory mapping for node pve"
> **Cause:** First write to `/etc/pve/mapping/directory.cfg` appeared to fail (pmxcfs I/O error) but actually succeeded. Retry after pmxcfs restart wrote a second entry.
> **Fix:** Manual edit of `directory.cfg` to remove duplicate entry
> **See Gotcha #42**

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]]
- [[ZFS Configuration and Setup]]
- [[Traefik]]
- [[Mount Point Map]]
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
