---
date: '2026-03-31T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
session_type: Debugging + Deploy
status: Completed
tags:
  - SessionNotes
  - DockerSwarm
  - Proxmox
  - CloudInit
  - Docker
  - Debugging
  - Portainer
  - Traefik
---

# 🛠️ Session Notes — Docker Swarm Pipeline Fixes

> [!abstract] Overview This session focused on hardening the three-script provisioning pipeline. Multiple bugs were squashed across `01-create-template.sh` and `02-provision-vm.sh`, the Portainer stack was rewritten for proper Swarm agent mode, and the first two Swarm nodes (`manager-01` and `traefik-dmz-01`) were successfully provisioned and joined.

---

## 🐛 Bugs Fixed

### `01-create-template.sh`

|Bug|Root Cause|Fix|
|---|---|---|
|`grep -q 'cicustom='` never matched|`qm config` outputs `key: value` format, not `key=value`|Changed to `grep -q '^cicustom:'`|
|Stray `st` token at top of script|Leftover edit artifact before `set -euo pipefail`|Removed the stray token|
|Em dash `—` (UTF-8) in vendor YAML heredoc|cloud-init YAML parser silently fails on non-ASCII characters|Replaced with plain hyphens; added `sed -i 's/\r//'` CRLF strip|
|`CI_USER` defaulted to `ubuntu`|Wrong default for this stack's user convention|Changed default to `docker`|

### `02-provision-vm.sh`

|Bug|Root Cause|Fix|
|---|---|---|
|`grep -q 'cicustom='` in verify step|Same `qm config` format mismatch as above|Changed to `grep -q '^cicustom:'`|
|Per-VM cloud-init snippet only set hostname/fqdn|`users:` block was never written into the per-VM snippet|Added full `users:` block with `ssh_authorized_keys` injection|
|`CI_USER` defaulted to `ubuntu` (mismatched with `03`)|Scripts had diverged defaults|Aligned both `02` and `03` to default `docker`|

> [!warning] CI_USER env var persistence If `CI_USER` is set in your shell session it will silently override script defaults. Always `unset CI_USER` before running, or use inline override: `CI_USER=docker bash 02-provision-vm.sh ...`

---

## ✨ Features Added

### `02-provision-vm.sh` — Per-VM hardware sizing flags

New optional flags for provision time:

```bash
--memory   <MB>       # Default: 2048
--cores    <n>        # Default: 2
--disk-size <GB>      # Default: 20
```

**Usage example:**

```bash
bash /root/02-provision-vm.sh \
  --vmid 201 \
  --hostname traefik-dmz-01 \
  --vlan 80 \
  --nic2-vlan 60 \
  --memory 1024 \
  --cores 2 \
  --disk-size 10
```

This avoids needing separate templates per VM size — the base template stays generic and sizing is applied at provision time via `qm set`.

---

## 📦 Stack Updates

### `stack-portainer.yml` — Rewrote to proper Swarm agent mode

**Before:** Standalone socket mode (`/var/run/docker.sock` volume mount directly on the Portainer service). This works for single-node Docker but is not Swarm-aware — it cannot see other nodes or manage stacks properly.

**After:** Portainer server + `portainer_agent` as a `global` service (runs on every Swarm node automatically). Portainer connects to agents via the overlay network.

**Key design decisions:**

- `portainer_agent` deployed as `mode: global` — no manual scaling needed as nodes join
- Direct port `9443` exposed on the manager so Portainer is reachable _before_ Traefik exists
- Both services in the same stack file for single-deploy convenience

```yaml
# stack-portainer.yml (structure summary)
services:
  portainer:
    image: portainer/portainer-ce:latest
    ports:
      - "9443:9443"
    volumes:
      - /mnt/docker-data/portainer:/data
    deploy:
      placement:
        constraints: [node.role == manager]

  portainer_agent:
    image: portainer/agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    deploy:
      mode: global
```

> [!note] Why global mode for the agent? `mode: global` means Docker Swarm automatically schedules one agent replica per node — including nodes added later. No manual intervention needed when you provision `worker-media-01` etc.

---

## 📚 Ops Learnings

### SSH key passphrase breaks unattended automation

`BatchMode=yes` in `03-post-boot.sh` causes SSH to immediately fail if the key has a passphrase (it can't prompt). The automation key at `/root/.ssh/id_rsa` must have no passphrase.

```bash
# Remove passphrase from existing key
ssh-keygen -p -f /root/.ssh/id_rsa
# Enter current passphrase, leave new passphrase blank
```

> [!tip] Separate keys for humans vs automation Keep your personal SSH key (with passphrase) for interactive Proxmox access. Use a dedicated passphraseless key baked into VMs for script automation.

---

### virtiofs bind mount directories — ownership matters

Directories on virtiofs mounts inherit whatever ownership they were created with. If the `admin` user created a service directory during setup, Docker containers (running as root) can still be blocked by the DAC check at the mount layer.

**Fix — all service dirs under `/mnt/docker-data` must be root-owned:**

```bash
chown root:root /mnt/docker-data/<service-dir>
chmod 755 /mnt/docker-data/<service-dir>
```

> [!warning] This applies to every service directory When running `03-post-boot.sh`, verify ownership after the mount step before starting any containers.

---

### `acme.json` must pre-exist with correct permissions

Traefik will fail to start if `acme.json` doesn't exist or has wrong permissions. Traefik won't create it — it expects the file to already be there.

```bash
# Run this before deploying the Traefik stack
touch /mnt/docker-data/traefik/acme.json
chmod 600 /mnt/docker-data/traefik/acme.json
```

> [!danger] 600 is required, not optional Traefik explicitly checks the permission bits and refuses to use `acme.json` if they are too open. The ACME cert will silently fail to be stored.

---

## 📊 Current Swarm State

|Node|VMID|VLAN|Status|
|---|---|---|---|
|`manager-01`|200|60|✅ Provisioned, Swarm initialised|
|`traefik-dmz-01`|201|80+60|✅ Provisioned, joined Swarm|
|`worker-media-01`|—|50|⏳ Pending|
|`worker-controller-01`|—|60+20|⏳ Pending|
|`worker-general-01`|—|40|⏳ Pending|

|Service|Status|
|---|---|
|Portainer (+ agent)|🔄 Deploying|
|Traefik v3|🔜 Next up|
|ZFS tuning + snapshot cron|⏳ Pending|

---

## Session Outcome

> [!success] Closed — 2026-03-31
> manager-01 and traefik-dmz-01 provisioned. Portainer stack deployed in Swarm agent mode.
>
> All outstanding tasks promoted to [[Swarm Topology]] and [[Docker Swarm Infrastructure Runbook]].
> → Current status: [[Swarm Topology]]


## 🔗 Related Notes

- [[Docker Swarm Infrastructure Runbook]]
- [[Session Notes — 2026-03-19 — VM Template Debugging]]
- [[Session Notes — 2026-03-23 — DataSourceNoCloud Template Fix]]
- [[Traefik Setup]]
- [[ZFS Configuration and Setup]]
