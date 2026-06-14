---
date: 2026-03-19
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
session_type: Diagnose + Fix
status: Completed
tags:
  - SessionNotes
  - DockerSwarm
  - Proxmox
  - CloudInit
  - SSH
  - Docker
  - Debugging
  - VMTemplate
---

# Session Notes — VM Template Debugging & Fixes

> [!abstract] Overview This session covered SSH key setup on Windows, rebuilding the VM template and provisioning scripts, and a full debugging run of a cloned VM from first boot through to Docker running correctly. Several non-obvious issues were uncovered and fixed.

---

## SSH Key Setup — Windows to Proxmox

### Generate key on Windows

```powershell
ssh-keygen -t ed25519 -C "homelab" -f "$env:USERPROFILE\.ssh\homelab_ed25519"
Get-Content "$env:USERPROFILE\.ssh\homelab_ed25519.pub" | clip
```

### Two separate destinations for the public key

|Destination|Purpose|
|---|---|
|`/root/.ssh/authorized_keys` on Proxmox host|SSH into Proxmox itself|
|`/root/.ssh/homelab_ed25519.pub` on Proxmox host|Used by `--sshkeys` flag in template script to inject into VMs via cloud-init|

> [!warning] These serve different purposes `authorized_keys` on Proxmox only controls access to Proxmox itself. The `.pub` file is what gets injected into cloned VMs. Both are needed but for different reasons.

### Copy public key to Proxmox host

```powershell
scp $env:USERPROFILE\.ssh\homelab_ed25519.pub root@<proxmox-ip>:/root/.ssh/homelab_ed25519.pub
```

### Terminus setup

- Import private key (`homelab_ed25519`, not `.pub`) into a Terminus keychain entry
- Set SSH profile authentication to use the keychain entry
- SSH user for VMs is `docker`, not `ubuntu` or `root`

---

## Issues Found in Original Scripts

### `create-swarm-template.sh`

|Issue|Fix|
|---|---|
|SSH key path hardcoded to `/root/.ssh/id_rsa.pub`|Changed to `/root/.ssh/homelab_ed25519.pub` with named variable + preflight check|
|`docker.io` from Ubuntu apt (version 20.x)|Replaced with Docker Engine from official repo (version 29.x)|
|Missing `qm template` call at end|Added — without this the VM exists but isn't actually a template|
|`fuse-overlayfs` not installed|Added to vendor packages|
|`99-pve.cfg` not baked in|Added to runcmd so every clone gets correct datasource order|

### `provision-swarm-worker.sh`

|Issue|Fix|
|---|---|
|Samba installed on every VM|Removed — no place on a Swarm node|
|Traefik profile comment showed `--vlan 10`|Fixed to `--vlan 60 --nic2 80`|
|`chown` used `${MAIN_UID:-$MAIN_USER}` for group|Fixed to always use `$MAIN_USER` — chown needs username not UID|
|`cicustom` only set `user=` — overwrote vendor|Fixed to include both: `user=...,vendor=local:snippets/base-vendor.yaml`|
|`storage-driver: overlay2` in daemon.json|Fixed to `fuse-overlayfs` — overlay2 doesn't work on virtiofs|
|No hostname set in cloud-init yaml|Added `hostname`, `fqdn`, `manage_etc_hosts: true`|
|No guard against DMZ VLAN as primary NIC|Added VLAN 80 warning + confirmation prompt|

---

## Debugging Run — traefik-dmz-01

### Issue 1 — Wrong VLAN on primary NIC

**Symptom:** `connect (101: Network is unreachable)` during cloud-init apt install.

**Cause:** VM provisioned with `--vlan 80 --nic2 60` — primary NIC landed on VLAN 80 (DMZ). pfSense blocks outbound internet from DMZ by design.

**Fix:** Swap the flags. Primary NIC must always be a management/internal VLAN with outbound internet access.

```bash
# Wrong
./provision-swarm-worker.sh --vlan 80 --nic2 60 ...

# Correct
./provision-swarm-worker.sh --vlan 60 --nic2 80 ...
```

> [!tip] Rule `--vlan` = management VLAN (60) — cloud-init, Swarm traffic, apt installs run here. `--nic2` = secondary purpose VLAN (80 DMZ, 20 IoT) — service-specific traffic only.

---

### Issue 2 — DNS not resolving

**Symptom:** `Temporary failure resolving 'archive.ubuntu.com'`

**Cause:** VM got a DHCP address but no DNS server was assigned, or `systemd-resolved` not configured correctly on first boot.

**Fix:** Resolved by correcting the VLAN issue — once on VLAN 60, DHCP handed out the correct DNS server (PiHole).

---

### Issue 3 — Docker and qemu-guest-agent not installed after cloud-init

**Symptom:** `docker: command not found`, `Unit docker.service could not be found`

**Cause:** `cicustom` in the provisioning script only set `user=` — this overwrote the `vendor=` config that the template set. Without the vendor config, the Docker Engine install runcmd never ran.

**Fix:**

```bash
# Broken — overwrites vendor config from template
qm set $VMID --cicustom "user=local:snippets/${VM_NAME}-cloud-init.yaml"

# Fixed — preserves both
qm set $VMID --cicustom "user=local:snippets/${VM_NAME}-cloud-init.yaml,vendor=local:snippets/base-vendor.yaml"
```

> [!warning] Cloud-init cicustom behaviour Setting `--cicustom` replaces the entire cicustom value. If you only set `user=`, any existing `vendor=` is silently dropped. Always include both in the same `--cicustom` call.

---

### Issue 4 — DataSourceNoCloud [seed=/dev/sr0]

**Symptom:** `cloud-init status --long` showed `detail: DataSourceNoCloud [seed=/dev/sr0]`. Vendor runcmd never executed even when cicustom was correctly set.

**Cause:** Ubuntu 24.04 cloud images default datasource order prefers NoCloud. Cloud-init was reading from the CD-ROM drive (`/dev/sr0`) as a NoCloud seed instead of using the Proxmox ConfigDrive. Because it had a valid NoCloud seed it never looked at the Proxmox cicustom configs.

**Fix — immediate (on existing VM):**

```bash
sudo tee /etc/cloud/cloud.cfg.d/99-pve.cfg << 'EOF'
datasource_list:
  - ConfigDrive
  - NoCloud
  - None
EOF

sudo cloud-init clean --logs
sudo reboot
```

**Fix — permanent (baked into template):** Added to vendor cloud-init runcmd in `create-swarm-template.sh` so every cloned VM gets `99-pve.cfg` written on first boot before any subsequent boots occur.

> [!note] Why this matters Without this fix, every VM cloned from the template ignores the Proxmox cicustom `vendor=` config. Packages install (from the `packages:` block) but `runcmd` is skipped entirely. Docker, qemu-guest-agent, virtiofs mounts, daemon.json, and Swarm join all live in runcmd — so nothing works.

---

### Issue 5 — overlay2 storage driver not supported on virtiofs

**Symptom:** Docker installed but failed to start with:

```
error initializing graphdriver: driver not supported: overlay2
failed to mount overlay: invalid argument storage-driver=overlay2
```

**Cause:** overlay2 requires a filesystem that supports `d_type` (directory entry type). virtiofs does not provide this. Docker's data root was set to `/mnt/docker-data` which is a virtiofs mount.

**Fix:** Use `fuse-overlayfs` instead of `overlay2` — designed to work on FUSE and virtiofs filesystems.

```json
{
  "data-root": "/mnt/docker-data",
  "storage-driver": "fuse-overlayfs",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Also requires `fuse-overlayfs` to be installed:

```bash
sudo apt-get install -y fuse-overlayfs
```

> [!tip] virtiofs + Docker storage driver Always use `fuse-overlayfs` when Docker's data root is on a virtiofs mount. overlay2 will fail silently during install and only error at first start.

---

## Final Sanity Check — What a Clean VM Should Look Like

Run these after cloud-init completes (`sudo cloud-init status` returns `done`):

```bash
# Correct hostname
hostname
# Expected: vm-name matching --name flag

# Docker Engine version (not docker.io)
docker --version
# Expected: Docker version 29.x or newer

# Docker running
sudo systemctl status docker --no-pager
# Expected: active (running)

# Docker functional
docker run --rm hello-world
# Expected: Hello from Docker!

# qemu-guest-agent running
sudo systemctl status qemu-guest-agent --no-pager
# Expected: active (running)

# All 4 virtiofs mounts present
mount | grep virtiofs
# Expected: docker-data, docker-tsdb, docker-db, docker-swarm all mounted

# daemon.json correct
sudo cat /etc/docker/daemon.json
# Expected: data-root=/mnt/docker-data, storage-driver=fuse-overlayfs

# User in docker group
groups
# Expected: docker group present

# cloud-init datasource correct
sudo cloud-init query dsname
# Expected: ConfigDrive (not DataSourceNoCloud)

# cloud-init completed cleanly
sudo cloud-init status
# Expected: status: done
```

---

## Key Lessons

**1 — Primary NIC must always be management VLAN** cloud-init needs outbound internet access to install packages. DMZ and IoT VLANs have restricted outbound rules. Always use `--vlan 60` (or whichever VLAN has internet access) as primary, secondary VLANs via `--nic2`.

**2 — cicustom user= silently drops vendor=** The `--cicustom` flag replaces the entire value. Setting only `user=` removes any existing `vendor=`. Always set both in a single call.

**3 — Ubuntu 24.04 cloud images default to NoCloud datasource** The distro image prefers reading cloud-init config from a local CD-ROM seed over Proxmox's ConfigDrive. This causes cicustom vendor configs to be silently ignored. Fix by baking `99-pve.cfg` into the template with ConfigDrive first in the datasource list.

**4 — overlay2 does not work on virtiofs** virtiofs doesn't support the `d_type` feature overlay2 requires. Use `fuse-overlayfs` as the Docker storage driver whenever the data root is on a virtiofs mount. Install `fuse-overlayfs` package alongside Docker.

**5 — cloud-init runcmd is skipped without a valid datasource** The `packages:` block runs regardless of datasource. `runcmd:` does not — it requires a valid datasource to be identified. If runcmd appears to be silently skipped, check `cloud-init status --long` for the datasource detail line first.

---

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Docker Swarm — Expansion & Storage Planning Notes]]
- [[Proxmox ZFS Dataset Layout]]