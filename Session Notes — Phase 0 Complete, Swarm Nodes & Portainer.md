---
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Proxmox
  - NFS
  - ZFS
  - Portainer
  - Traefik
date: 2026-04-05
---

# Session Notes — Phase 0 Complete, Swarm Nodes & Portainer

> [!abstract] Overview
> Completed Phase 0 storage setup. Resolved NFS config issues. Deployed manager-01
> and traefik-dmz-01, resolved Swarm join issues, deployed Portainer in agent mode.
> Traefik deploy deferred to next session via Portainer UI.

---

## Phase 0 Completed

### Step 0.2 — ZFS Tuning + Snapshot Cron
- Git installed on Proxmox host (required adding base Debian bookworm repo)
- GitHub SSH access configured via port 443 workaround (pfSense blocks outbound port 22)
- Repo cloned to `/root/proxmox-swarm` via SSH
- `zfs-tune-datasets.sh` and `zfs-snapshot.sh` deployed to `/usr/local/bin/`
- Cron deployed to `/etc/cron.d/zfs-docker-snapshots`
- Tuning applied and verified

### Step 0.4 — NFS Exports
Fixed several issues from initial attempt:

| Issue | Cause | Fix |
|---|---|---|
| `nfs-kernel-server` not found | Base Debian repo missing from Proxmox sources.list | Added `deb http://deb.debian.org/debian bookworm main` |
| Exports using `/rpool/docker-*` paths | Wrong — NFS exports filesystem paths not ZFS names | Changed to `/mnt/docker-*` |
| Extra VLAN 85 exports appearing | Stale `sharenfs` property on child datasets from old session | `zfs set sharenfs=off rpool/docker-swarm/configs` and `volumes` |
| `ls /mnt/test` permission denied | `docker` user cannot read root-owned NFS mount | Expected — `sudo ls` works, Docker containers run as root |

Correct `/etc/exports`:
```
/mnt/docker-data    10.0.60.0/25(rw,sync,no_subtree_check,no_root_squash)
/mnt/docker-tsdb    10.0.60.0/25(rw,sync,no_subtree_check,no_root_squash)
/mnt/docker-db      10.0.60.0/25(rw,sync,no_subtree_check,no_root_squash)
/mnt/docker-swarm   10.0.60.0/25(rw,sync,no_subtree_check,no_root_squash)
```

NFS mount test from manager-01:
```bash
sudo mkdir -p /mnt/test
sudo mount -t nfs 10.0.90.50:/mnt/docker-data /mnt/test
sudo ls /mnt/test   # works with sudo
sudo umount /mnt/test
```

---

## Swarm Node Issues

### manager-01 reprovisioned
Previous manager-01 was inaccessible — reprovisioned cleanly. As a result:
- Swarm was reinitialised with a new CA
- All previously saved worker tokens became invalid
- traefik-dmz-01 failed to join with "remote CA does not match fingerprint" error

> [!warning] Always get a fresh worker token after manager reprovision
> The token in `/root/swarm_worker_token.txt` goes stale on Swarm reinit.
> Always run `docker swarm join-token worker` on manager-01 to get the current token.

### traefik-dmz-01 joined as manager instead of worker
`03-post-boot.sh` used `--first-manager` flag or wrong token — traefik-dmz-01
initialised its own single-node Swarm as Leader instead of joining manager-01.

Fix:
```bash
# On traefik-dmz-01 — force leave its own swarm
sudo docker swarm leave --force

# Rejoin as worker using fresh token from manager-01
sudo docker swarm join --token <fresh-worker-token> 10.0.60.30:2377
```

> [!note] Check provisioning script worker token handling
> Verify `02-provision-vm.sh` reads from `/root/swarm_worker_token.txt` correctly
> and that file is kept current. This is a known failure point — add to next session
> checklist.

---

## Stacks Deployed

### Portainer ✅
- Running on manager-01, accessible at `https://<manager-01-ip>:9443`
- `portainer_agent` running as global service — auto-expands to new nodes
- Stack file: `/mnt/docker-swarm/stacks/portainer/stack.yml`

**Corrections made to stack before deploying:**

| Issue | Fix |
|---|---|
| `proxy` network referenced throughout | Changed to `traefik-public` |
| `entrypoints=websecure` | Changed to `internal` — Portainer is not external |
| `tls.certresolver=cloudflare` | Removed — internal entrypoint uses local wildcard cert |
| `loadbalancer.server.port=9443` | Changed to `9000` — Traefik talks to internal HTTP port |
| `loadbalancer.server.scheme=https` | Removed — internal comms are plain HTTP |
| `traefik-public` declared as driver overlay | Changed to `external: true` — network pre-created manually |
| Agent on `traefik-public` network | Moved to dedicated `portainer_agent_network` |

### Traefik — pending next session
Stack and config files prepared, node labelled. Deploy via Portainer next session.

**Corrections made to stack/config:**

| Issue | Fix |
|---|---|
| `proxy` network | Changed to `traefik-public` |
| `node.role == manager` constraint | Changed to `node.labels.zone == public` |
| Port `mode: ingress` | Changed to `mode: host` — preserves real client IPs |
| Single `websecure` entrypoint in traefik.yml | Added `internal` entrypoint on `:8443` |
| `network: proxy` in swarm provider | Changed to `traefik-public` |
| `acme.json` / `cf_api_token.txt` permission denied | Fixed ownership on Proxmox host |

---

## Infrastructure Changes

### Git on Proxmox host
```bash
# Required adding base Debian repo first
echo "deb http://deb.debian.org/debian bookworm main contrib non-free" >> /etc/apt/sources.list
apt-get update && apt-get install git -y

# GitHub SSH via port 443 (pfSense blocks port 22 outbound)
cat > /root/.ssh/config << 'EOF'
Host github.com
  HostName ssh.github.com
  Port 443
  User git
  IdentityFile /root/.ssh/github_ed25519
  IdentitiesOnly yes
EOF

git clone git@github.com:scarredNinja/docker-swarm-home.git proxmox-swarm
```

### Stack file location convention
- Repo (source): `/root/proxmox-swarm/stacks/<service>/stack.yml`
- Deployed (live): `/mnt/docker-swarm/stacks/<service>/stack.yml`
- Runtime config: `/mnt/docker-data/<service>/data/`

### Overlay network
```bash
docker network create --driver overlay --attachable traefik-public
```
All services use `traefik-public`. Mark as `external: true` in all stack files.

---

## Current Swarm State

| Node | Role | Status |
|---|---|---|
| manager-01 | Manager / Leader | Running — `10.0.60.30` |
| traefik-dmz-01 | Worker | Running — joined ✅ |

| Service | Status |
|---|---|
| portainer | Running ✅ |
| portainer_agent | Running on all nodes ✅ |
| traefik | Pending — next session |

---

## Next Session

- [ ] Deploy Traefik stack via Portainer UI
- [ ] Verify both entrypoints active (`:443` websecure, `:8443` internal)
- [ ] Check Cloudflare DNS challenge completes and cert is issued
- [ ] Verify Portainer accessible via `portainer.home.purvishome.com` through Traefik internal entrypoint
- [ ] **Check `02-provision-vm.sh` worker token handling** — confirm script reads fresh token correctly, not a stale cached value. This caused traefik-dmz-01 to join as manager.

## Session Outcome

> [!success] Phase 0 complete — 2026-04-05
> All Phase 0 storage setup done. manager-01 and traefik-dmz-01 provisioned and in Swarm.
> Portainer deployed. Traefik deploy is next session's first task.
>
> → Current status: [[Swarm Topology]]

---

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Traefik Routing Architecture]]
- [[Swarm Topology]]
