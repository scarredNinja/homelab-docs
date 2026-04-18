---
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Proxmox
  - ZFS
  - Traefik
  - Storage
date: 2026-03-30
---

# Session Notes — Post-Boot Improvements, ZFS Tuning & Traefik Stack

> [!abstract] Overview
> Overhauled `03-post-boot.sh` to be fully idempotent with numbered step logging, per-step verification gates, a `--dry-run` mode, `--label` support, and optional NFS fstab handling. Added ZFS dataset tuning and snapshot scripts to address a previously unprotected backup gap (PBS covers VM zvols, not host-side virtiofs datasets). Wrote the production Traefik v3 stack for `traefik-dmz-01` with `mode: host` ports and `zone=public` placement, and renamed the overlay network from `proxy` to `traefik-public`.

---

## Changes Made

### Goal 1 — `03-post-boot.sh` v2.0

> [!note] File: `proxmox-swarm/scripts/03-post-boot.sh`

#### What changed

- **Step numbering**: `step()` now emits `[STEP N/M]` where M is computed dynamically by `compute_steps()` after registry load. Conditional steps (NFS, Docker install, Swarm, labels) are only counted if they will actually run.
- **Dry-run mode**: `--dry-run` wraps `remote()` and `remote_sudo()`. `remote_sudo` now does `script=$(cat)` to capture stdin before branching — this is what makes it work with both heredocs and brace-group pipes.
- **Node labels**: `--label key=value` (repeatable). `apply_node_labels()` SSHes to the manager to run `docker node update --label-add`. For first manager, applies to self. Idempotent — `--label-add` on an already-set label is a no-op.
- **NFS support**: `configure_nfs()` reads optional `nfs[]` array from VM registry JSON. Uses `defaults,_netdev,x-systemd.automount,noatime,soft,timeo=30` — mounts are lazy (first access), not blocking boot.
- **Per-step gates**: After virtiofs → verify all 4 mountpoints. After Docker install → verify `docker --version`. After Docker daemon → verify `docker info` returns `fuse-overlayfs`. After swarm join/init → verify `Swarm.LocalNodeState == active`.
- **CI_USER default**: Changed from `ubuntu` to `docker`.
- **Sanity report**: Includes virtiofs rows, NFS rows, labels, warnings section.

#### Argument passing with SSH pipe (documentation)

The script runs **on the Proxmox host** — it is not designed to be piped into a VM. If you ever need the pipe pattern for a different in-VM script:

```bash
ssh -i /root/.ssh/homelab_ed25519 docker@<VM_IP> bash -s -- --arg1 val < some-in-vm-script.sh
```

`bash -s` reads stdin as the script; `--` ends `bash` options; everything after is `$1`, `$2`, etc. inside the piped script.

#### Re-run example after partial failure

```bash
# Docker already installed, just re-join swarm and apply labels
./03-post-boot.sh worker-media-01 --skip-docker --label type=media --label zone=private

# Preview before running
./03-post-boot.sh worker-media-01 --dry-run --label type=media --label zone=private
```

---

### Goal 2 — ZFS Dataset Tuning + Snapshot Cron

> [!note] Files: `proxmox-swarm/scripts/zfs-tune-datasets.sh`, `zfs-snapshot.sh`, `proxmox-swarm/cron/zfs-docker-snapshots`

#### Tuning properties

| Dataset | recordsize | compression | atime | xattr |
|---|---|---|---|---|
| rpool/docker-data | 64K | lz4 | off | sa |
| rpool/docker-tsdb | 64K | lz4 | off | sa |
| rpool/docker-db | 8K | lz4 | off | sa |
| rpool/docker-swarm | 64K | lz4 | off | sa |

`xattr=sa` stores xattrs in the inode — Docker sets many xattrs; this prevents an extra metadata block read/write per file.

> [!warning] recordsize applies to new writes only
> Existing blocks are not rewritten. To apply to existing data, `zfs send | zfs recv` the dataset. Run `zfs-tune-datasets.sh` before writing significant data for the first time.

#### Snapshot schedule

```
:05 hourly  → @hourly-YYYYMMDD-HHmm  (keep 24 rolling)
02:00 daily → @daily-YYYYMMDD        (keep 7)
:20 hourly  → prune hourly (keep 24)
02:30 daily → prune daily  (keep 7)
```

`flock -n` on the snapshot jobs prevents overlap. Prune runs are not locked — they're fast and idempotent.

#### Deploy

```bash
# Copy scripts
cp proxmox-swarm/scripts/zfs-snapshot.sh /usr/local/bin/
cp proxmox-swarm/scripts/zfs-tune-datasets.sh /usr/local/bin/
chmod +x /usr/local/bin/zfs-snapshot.sh /usr/local/bin/zfs-tune-datasets.sh

# Deploy cron
cp proxmox-swarm/cron/zfs-docker-snapshots /etc/cron.d/
chmod 644 /etc/cron.d/zfs-docker-snapshots

# Apply tuning (check first)
zfs-tune-datasets.sh --dry-run
zfs-tune-datasets.sh

# Check snapshot status
zfs-snapshot.sh status
```

---

### Goal 3 — Traefik Stack + Network Migration

> [!note] Files: `proxmox-swarm/stacks/traefik/stack.yml`, `.env.template`, `traefik/traefik.yaml`

#### Stack location convention

All stacks live at `/mnt/docker-swarm/stacks/<service>/stack.yml` on the host, visible from every VM via the `docker-swarm` virtiofs mount.

```bash
# Standard deploy command
docker stack deploy -c /mnt/docker-swarm/stacks/<service>/stack.yml <service>
```

#### One-time setup (run on manager before first deploy)

```bash
# Create overlay network
docker network create --driver overlay --attachable traefik-public

# Cert storage (must be 600 — Traefik will refuse acme.json with wider permissions)
mkdir -p /mnt/docker-data/traefik/data/dynamic
install -m 600 /dev/null /mnt/docker-data/traefik/data/acme.json

# CF API token secret file
printf '%s' '<your-cloudflare-api-token>' > /mnt/docker-data/traefik/cf_api_token.txt
chmod 600 /mnt/docker-data/traefik/cf_api_token.txt

# Label the DMZ node
docker node update --label-add zone=public traefik-dmz-01
```

#### Deploy Traefik

```bash
# Generate dashboard password hash
htpasswd -nB admin | sed 's/\$/\$\$/g'
# → paste result into .env as TRAEFIK_DASHBOARD_USERS

cd /mnt/docker-swarm/stacks/traefik
cp .env.template .env
# edit .env with real TRAEFIK_DASHBOARD_USERS value

set -a; source .env; set +a
docker stack deploy -c stack.yml traefik
```

#### Downstream service opt-in example (Portainer)

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    networks:
      - traefik-public
    deploy:
      placement:
        constraints:
          - node.role == manager
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.portainer.rule=Host(`portainer.home.purvishome.com`)"
        - "traefik.http.routers.portainer.entrypoints=websecure"
        - "traefik.http.routers.portainer.tls=true"
        - "traefik.http.routers.portainer.tls.certresolver=cloudflare"
        - "traefik.http.services.portainer.loadbalancer.server.port=9000"
        - "traefik.docker.network=traefik-public"

networks:
  traefik-public:
    external: true
```

> [!tip] Network migration from `proxy`
> Existing stacks (`stacks/stack-portainer.yml`, etc.) reference `proxy`. Migration path:
> 1. Create `traefik-public` overlay network
> 2. Add `traefik-public` as a second network to Traefik temporarily
> 3. Update each service stack to use `traefik-public`
> 4. Remove `proxy` from Traefik, then remove the `proxy` network

---

## Key Decisions

**`remote_sudo()` uses `script=$(cat)` pattern**: Capturing stdin with `cat` before branching is the only way to support both dry-run (print) and live (exec) from the same function when called with heredocs. The alternative — duplicating every heredoc block — would be unmaintainable.

**NFS uses `x-systemd.automount`**: Prevents boot stall if NFS server (TrueNAS/host) is temporarily unavailable. The tradeoff is a ~1-2s delay on the first access to each mountpoint. For media/DB workloads this is acceptable.

**`docker-db` recordsize = 8K**: SQLite and Postgres default to 8K pages. A ZFS recordsize matching the DB page size means every DB write is exactly one ZFS record — no read-modify-write amplification on partial blocks.

**Traefik ports `mode: host`**: Ingress mesh would rewrite source IPs to the Swarm routing mesh IP. `mode: host` pins the service to the node and preserves real client IPs — required for rate limiting, geo-blocking, and ACME HTTP-01 challenges. Acceptable because Traefik is single-replica on a dedicated DMZ node.

**File-based Traefik config (not CLI flags)**: Matching the existing working setup. `--configfile` makes the static config reviewable and diff-able in git. The dynamic config directory (`/etc/traefik/dynamic/`) handles non-Swarm services (pfSense, Proxmox, pihole) without requiring a Traefik restart.

**CF API token via Docker secret, not env var**: Env vars in Swarm services are visible in `docker service inspect`. Docker secrets are mounted as tmpfs files accessible only to the container, never logged or inspectable post-deploy.

---

## Session Outcome

> [!success] Closed — 2026-03-30
> 03-post-boot v2 complete. ZFS tuning scripts written. Traefik v3 stack written.
>
> All outstanding tasks promoted to [[Swarm Topology]] and [[Docker Swarm Infrastructure Runbook]].
> → Current status: [[Swarm Topology]]
