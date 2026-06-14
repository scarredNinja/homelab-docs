---
date: 2026-06-14
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm — Deploy Developer Platform"
session_type: Deploy
status: Complete
tags:
  - SessionNotes
  - DockerSwarm
  - Forgejo
  - Woodpecker
  - Proxmox
  - ZFS
---

# Session Notes — 2026-06-14 — Gitea Forgejo and Woodpecker CI Setup

## 🎯 Session Goal

Deploy a self-hosted developer platform (Forgejo and Woodpecker CI) on Docker Swarm, establishing an isolated VM for CI execution (`dev-runner-01`) on VLAN 40 to run untrusted build steps securely.

---

## ✅ Completed

### 1. Provisioning Scripts Modified
* Modified [[02-provision-vm.sh]] to support a new `--no-virtiofs` parameter. When passed, it skips attaching ZFS virtiofs shares and saves an empty `[]` array in the VM's JSON registry.
* Modified [[03-post-boot.sh]] to skip fstab mounts and Docker systemd drop-in dependencies if virtiofs list is empty.

### 2. VM Provisioned
* Provisioned `dev-runner-01` (VLAN 40, 4 vCPUs, 4GB RAM, 40G disk) with the `--no-virtiofs` option.
* Confirmed it has zero ZFS virtiofs host mounts, joined the Docker Swarm as a worker, and successfully labeled it `zone=runner`.

### 3. Stack Configurations Created and Fixed
* Created [[stack-forgejo.yml]] (Forgejo + Postgres) pinned to `zone=controller` (VLAN 60).
* Created [[stack-woodpecker.yml]] (Woodpecker Server + Agent) with the Server pinned to `zone=controller` and the Agent pinned to `zone=runner` (no-virtiofs).
* Resolved tag mismatch: Corrected Forgejo image tag from `codeberg.org/forgejo/forgejo:8-alpine` to `codeberg.org/forgejo/forgejo:8`.

### 4. Disk Space Cleanup
* Diagnosed disk full issue on `worker-controller-01` where `/` was at 100% capacity (19GB/19GB used).
* Ran `docker system prune -a -f` on `worker-controller-01` and reclaimed **8.82 GB** of disk space (reducing root usage to 48%).

### 5. Manually Deployed
* Manually deployed the Forgejo stack using the corrected stack definition on the `feature/dev-platform-setup` branch.

---

## 🔍 Diagnostic Path (Disk Space & Image Tag Issues)

| Observation | Conclusion | Action Taken |
|---|---|---|
| Deployment of `forgejo` fails with "No such image: codeberg.org/forgejo/forgejo:8-alpine" | Codeberg registry does not publish an Alpine-specific tag for major version 8 | Changed image to `codeberg.org/forgejo/forgejo:8` in stack config |
| Retrying pull fails with `no space left on device` on `worker-controller-01` | Root disk `/dev/sda1` on the controller VM is at 100% capacity | Ran remote `docker system prune -a -f` via SSH to free up space |
| Disk space check after prune shows 9.6GB free (48% usage) | Host has plenty of room to pull and run Forgejo | Successfully redeployed the manual stack |

---

## 🐛 Bug #19 — virtiofs Permissions Provisioning Hang (Resolved)
> [!bug] Root Cause
> A recursive `chmod -R` on the host side for virtiofs guest mount directories caused a kernel crash/hang in `manager-03`.
> **Resolution**: Removed the `-R` recursive option from host-side scripts, rebooted `manager-03`, and successfully force-updated Portainer and other services to restore Swarm health.

## 🐛 Bug #20 — No space left on device on worker-controller-01 (Resolved)
> [!bug] Root Cause
> Pulling container images failed due to the `/` partition being 100% full (19G/19G) on `worker-controller-01`.
> **Resolution**: Cleaned up the Docker cache by running `docker system prune -a -f` remotely on `worker-controller-01`, reclaiming **8.82 GB**.

---

## 📊 Current State (End of Session)

### Node Status

| Node | Zone | VLAN | Status | Swarm Joined | Virtiofs Mounts |
|---|---|---|---|---|---|
| `dev-runner-01` | `runner` | 40 | 🟢 Running | ✅ Yes (Worker) | 🔒 None (Secure Runner) |
| `worker-controller-01` | `controller` | 60 | 🟢 Running | ✅ Yes (Worker) | ✅ Yes |

### Stack Status

| Stack | Service | Status | Image | Placement Constraint |
|---|---|---|---|---|
| `forgejo` | `db` | 🟢 Running | `postgres:16-alpine` | `node.labels.zone == controller` |
| `forgejo` | `forgejo` | 🟢 Running | `codeberg.org/forgejo/forgejo:8` | `node.labels.zone == controller` |

---

## ➡️ Next Session Priorities

- [ ] **Configure Forgejo Admin**: Access `https://git.dev.home.purvishome.com:8444` and set up the initial admin account.
- [ ] **Establish OAuth App**: Create an OAuth application inside Forgejo for Woodpecker integration.
- [ ] **Configure Swarm Secrets**: Store Woodpecker OAuth credentials (`woodpecker_gitea_client_id` and `woodpecker_gitea_client_secret`) in Docker secrets.
- [ ] **Deploy Woodpecker Stack**: Deploy `stack-woodpecker.yml` to spin up the Woodpecker Server on VLAN 60 and the Woodpecker Agent on `dev-runner-01` (VLAN 40).
- [ ] **Verify CI Pipeline**: Trigger a test build to ensure the runner executes builds in isolation on `dev-runner-01`.

---

## 🔗 Related Notes

- [[06 Self-Hosted Developer Platform — Master Hub]]
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
- [[Docker Swarm Infrastructure Runbook]]
- [[VM - worker-controller-01]]
- [[VM - dev-runner-01]]
