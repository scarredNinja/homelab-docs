---
project_id: Homelab-2025
status: Reference
phase: 'Phase 5: Docker Swarm'
tags:
  - reference
  - architecture
---
# Proxmox Host — Cron Jobs

Drop the file below into `/etc/cron.d/backup-jobs` on the Proxmox host.
All scripts must already be installed (e.g. via `cp` or symlink) under `/usr/local/bin/`.

## Install scripts

```bash
cp proxmox-swarm/proxmox-swarm/scripts/proxmox-config-backup.sh /usr/local/bin/proxmox-config-backup.sh
cp proxmox-swarm/proxmox-swarm/scripts/zfs-snapshot.sh          /usr/local/bin/zfs-snapshot.sh
cp proxmox-swarm/proxmox-swarm/scripts/restic-backup.sh         /usr/local/bin/restic-backup.sh
chmod +x /usr/local/bin/proxmox-config-backup.sh \
         /usr/local/bin/zfs-snapshot.sh \
         /usr/local/bin/restic-backup.sh
```

## `/etc/cron.d/backup-jobs`

```cron
# Proxmox backup jobs
# m  h  dom mon dow  user  command
0    1  *   *   *    root  /usr/local/bin/proxmox-config-backup.sh >> /var/log/proxmox-config-backup.log 2>&1
0    2  *   *   *    root  /usr/local/bin/zfs-snapshot.sh send daily >> /var/log/zfs-snapshot.log 2>&1
0    3  *   *   *    root  /usr/local/bin/restic-backup.sh >> /var/log/restic-backup.log 2>&1
```

> The existing `/etc/cron.d/zfs-docker-snapshots` file schedules local snapshot creation
> and pruning (hourly + daily). The `send daily` job above is separate and runs after
> the 02:00 daily snapshot has been taken.

## Alert configuration

Create `/etc/backup-alerts.conf` with your Discord webhook URL:

```bash
DISCORD_WEBHOOK_URL="https://discord.com/api/webhooks/<id>/<token>"
```

This file is sourced at runtime by all three scripts. Keep it root-readable only:

```bash
chmod 600 /etc/backup-alerts.conf
```

## Restic prerequisites

```bash
# Install restic
apt-get install restic

# Write the repository password
echo 'your-strong-password' > /etc/restic-password
chmod 600 /etc/restic-password

# Generate SSH key for Synology access
ssh-keygen -t ed25519 -f /root/.ssh/id_backup_synology -N ''
# Then add the public key to backup-agent's authorized_keys on the Synology

# Add SSH config entry so restic can connect without inline options
cat >> /root/.ssh/config <<'EOF'

Host synology-backup
    HostName 10.0.100.20
    User backup-agent
    IdentityFile /root/.ssh/id_backup_synology
    StrictHostKeyChecking no
    BatchMode yes
EOF
chmod 600 /root/.ssh/config

# Test the connection
ssh synology-backup

# Initialise the restic repository (first time only)
RESTIC_PASSWORD_FILE=/etc/restic-password \
restic -r sftp:synology-backup:/volume1/docker-backups init
```
