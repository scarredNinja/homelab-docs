---
date: '2026-04-10T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
session_type: Debugging + Deploy
status: Completed
tags:
  - SessionNotes
  - DockerSwarm
  - Traefik
  - Portainer
  - Proxmox
---

# Session Notes — 2026-04-10 — Traefik + Portainer Deployment

## Summary

Long session covering Swarm manager recovery, virtiofs permission fixes, and full
deployment of Traefik + Portainer with working HTTPS routing via Cloudflare DNS
challenge wildcard certs.

---

## Issues Found and Fixed

### Bug 5 (Known) — Swarm State Loss on Reboot

**Symptom:** `manager-01` reported as worker node after reboot. `docker node ls`
returned: "This node is not a swarm manager."

**Root cause:** Two contributing factors:
1. Docker creates subdirectories under `/mnt/docker-data` with `700` permissions.
   virtiofs exposes host-side permissions directly to the VM with no remapping. On
   reboot Docker couldn't read its own Raft state DB (bbolt nil pointer), failed
   Swarm init, and demoted itself to worker.
2. The `swarm/` directory was missing entirely on this rebuild — Docker had never
   successfully persisted state into it.

**Recovery applied this session:**
```bash
# On Proxmox host — fix permissions at the ZFS mountpoint level
sudo chmod 755 /mnt/docker-data/swarm
sudo chmod 755 /mnt/docker-data/network
sudo chmod 755 /mnt/docker-data/runtimes
sudo chmod 755 /mnt/docker-data/tmp
sudo chmod 755 /mnt/docker-data/volumes
sudo chmod 755 /mnt/docker-data/plugins
sudo chmod 755 /mnt/docker-data/image
sudo chmod 755 /mnt/docker-data/containers

# On manager-01 — stop Docker, clear stale state, restart
sudo systemctl stop docker
sudo rm -rf /mnt/docker-data/network
sudo systemctl start docker

# Re-initialise Swarm
docker swarm init --advertise-addr 10.0.60.30

# Re-apply zone=public label to traefik-dmz-01
docker node update --label-add zone=public <NODE_ID>

# Rejoin traefik-dmz-01
docker swarm leave --force  # on traefik-dmz-01
docker swarm join --token <TOKEN> 10.0.60.30:2377  # on traefik-dmz-01
```

**Key learning:** Always fix Docker data subdirectory permissions at the Proxmox
host ZFS mountpoint, not inside the VM. The fix inside the VM does not survive
a Docker restart because Docker recreates those dirs with `700`.

---

### docker-swarm Ownership Inconsistency

`backups/`, `configs/`, `volumes/` under `/mnt/docker-swarm` were owned by
`admin:admin` from the old setup. Fixed to `root:root`:

```bash
sudo chown -R root:root /mnt/docker-swarm/backups
sudo chown -R root:root /mnt/docker-swarm/configs
sudo chown -R root:root /mnt/docker-swarm/volumes
```

---

### Traefik Service Discovery

Traefik couldn't discover Swarm services via Docker socket directly. Switched to
Docker socket proxy (`tcp://docker-proxy:2375`) as the Swarm endpoint. Keeps the
Docker socket off the Traefik container entirely.

---

### `internal-only` Middleware Missing

Middleware was defined in a root-level file Traefik wasn't loading. Moved into
`traefik/dynamic/middlewares.yml` which is correctly mounted into the container.

---

### Deprecated `traefik.docker.*` Labels

Stack files used `traefik.docker.network` — deprecated in Traefik v3. Replaced
with `traefik.swarm.network` across all stack files.

---

### Portainer Stack — Missing certresolver Label

`stack-portainer.yml` had `tls=true` but no `tls.certresolver=cloudflare`. Added:

```yaml
- "traefik.http.routers.portainer.tls.certresolver=cloudflare"
```

---

### pihole Redirect Wrong Domain

pihole redirect rule referenced `example.com` placeholder. Updated to `purvishome.com`.

---

## Deployments This Session

- `traefik-public` overlay network created
- Portainer deployed in agent mode via `stack-portainer.yml`
- Traefik config copied from repo to `/mnt/docker-data/traefik/data/` via
  `copy-traefik-config.sh` (created this session, lives on Proxmox host at `/root/`)
- Traefik stack deployed, pinned to `traefik-dmz-01` via `node.labels.zone == public`
- CF API token Docker secret created

---

## State at End of Session

| Service | URL | Status |
|---|---|---|
| Traefik dashboard | `https://traefik.home.purvishome.com:8443` | ✅ |
| Portainer | `https://portainer.home.purvishome.com` | ✅ |
| Portainer agent | All nodes | ✅ |

| Node | Role | Status | Labels |
|---|---|---|---|
| manager-01 | Manager/Leader | Ready | `role=manager` |
| traefik-dmz-01 | Worker | Ready | `zone=public` |

---

## ZFS Observations

- `rpool/docker-data` confirmed: `lz4` compression, `atime=off`, `recordsize=64K`
- Docker consistently creates data subdirs with `700` — this is by design and will
  recur on every fresh Docker start against a clean virtiofs mount
- Internal DNS resolving via pihole v6 local DNS records with pfSense DHCP domain
  set to `lan`
