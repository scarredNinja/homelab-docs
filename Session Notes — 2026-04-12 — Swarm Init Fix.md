---
date: 2026-04-12
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - SessionNotes
  - DockerSwarm
  - Proxmox
  - virtiofs
  - BugFix
session_type: Debugging + Code
status: Complete
---

# Session Notes — 2026-04-12

## 🎯 Session Goal

Diagnose and fix Docker Swarm manager (`manager-01`) failing to rejoin as manager after reboot / server switch. Node was reporting `This node is not a swarm manager` on every restart.

---

## 🔍 Diagnosis

### Symptom
```
Swarm: pending
NodeID: 8659l42oy8mmualsetbo9t1jl
Is Manager: false
Node Address: 10.0.60.30
```

Node knew it was *in* a Swarm (had a NodeID) but could not read Raft state → demoted itself to non-manager.

### Investigation Steps

**Step 1 — Check host-side permissions on Proxmox:**
```bash
ls -la /mnt/docker-data/
ls -laR /mnt/docker-data/swarm/
```

Found several `drwx------` (700) directories and `rw-------` (600) files including `docker-state.json`.

**Step 2 — Read the state file:**
```bash
cat /mnt/docker-data/swarm/docker-state.json
```

Output:
```json
{
  "LocalAddr": "",
  "RemoteAddr": "10.0.60.30:2377",
  "ListenAddr": "0.0.0.0:2377",
  "AdvertiseAddr": "",
  "DataPathAddr": "",
  "DefaultAddressPool": null,
  "SubnetSize": 0,
  "DataPathPort": 0,
  "JoinInProgress": false
}
```

`AdvertiseAddr` was empty — this was a stale/incomplete join-in-progress state, not a healthy manager state. `chmod` alone could not fix it; the content itself was corrupt.

### Root Causes Identified

Three compounding bugs, all virtiofs + ZFS related:

| Bug | Description |
|-----|-------------|
| **#19** | Docker creates subdirs under `/mnt/docker-data/` with `700` permissions. virtiofs exposes host-side permissions directly (no UID remapping). Docker can't read Raft state DB on restart → demotes to non-manager. |
| **#5** | `docker-state.json` contained an incomplete join-in-progress state (`AdvertiseAddr: ""`). Persists across VM rebuilds/shutdowns on the shared ZFS dataset. Causes `Swarm: pending` on every restart. |
| **#6** | Portainer agent stack deployed before all worker nodes join causes scoped network scheduling failures. |

---

## 🔧 Fix Applied

### 1. Back up and clear stale Raft state (Proxmox host)

```bash
# Backup first
cp -r /mnt/docker-data/swarm /mnt/docker-data/swarm.bak.20260412

# Clear stale state
rm -rf /mnt/docker-data/swarm/*
rm -rf /mnt/docker-data/network/files/local-kv.db 2>/dev/null || true

# Fix permissions
chmod -R 755 /mnt/docker-data/
```

> [!important] This must run on the **Proxmox host**, not inside the VM. virtiofs reflects host permissions directly — VM-side fixes don't survive Docker restart.

### 2. Re-initialise Swarm (manager-01)

```bash
sudo systemctl restart docker
sudo docker swarm init --advertise-addr 10.0.60.30
sudo docker info | grep -A3 "Swarm"
sudo docker node ls
```

### 3. Re-join worker (traefik-dmz-01)

```bash
# On manager-01 — get fresh token
sudo docker swarm join-token worker

# On traefik-dmz-01
sudo docker swarm leave --force
# Paste join command from above
```

### Result

```
ID                            HOSTNAME         STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
tqowbnbe56go8i451cla97dvm *   manager-01       Ready     Active         Leader           29.4.0
ii7nk2izpsoop5bta5vmwpc5z    traefik-dmz-01   Ready     Active                          29.4.0
```

---

## 💻 Code Session — PR #7

Branch `claude/swarm-postboot-zfs-traefik-GaJd9` merged to main. Branch deleted.

### Files Changed

| File | Change |
|------|--------|
| `stacks/portainer.yml` | New — agent mode, port 9000, `/mnt/docker-swarm/portainer-data/`, encrypted overlay `agent-network` |
| `stacks/traefik.yml` | New — docker-socket-proxy, correct image tag (`traefik:v3`) |
| `config/traefik/` | New — swarm provider endpoint, ACME, dashboard router, IP allowlist (LAN3 + all internal VLANs) |
| `copy-traefik-config.sh` | Updated paths to `proxmox-swarm/config/traefik/` |
| `03-post-boot.sh` | Bug #5, #6, #19 fixes added |

### Portainer Stack (portainer.yml)

Key decisions vs old stack:
- Agent mode (`-H tcp://tasks.agent:9001 --tlsskipverify`) not socket mode
- Agent deployed `global` across all Linux nodes
- Portainer `replicas: 1` pinned to `node.role == manager` (CE doesn't support HA)
- Port `9000:9000` direct — no Traefik labels yet (Traefik not deployed)
- Persistent data to `/mnt/docker-swarm/portainer-data/`
- Overlay network `agent-network` encrypted, no external dependencies

### 03-post-boot.sh Fixes

**Bug #5 — Stale Swarm state DB** (runs on Proxmox host before Docker starts):
```bash
rm -rf /mnt/docker-data/swarm/*
rm -rf /mnt/docker-data/network/files/local-kv.db 2>/dev/null || true
```

**Bug #6 — Portainer agent timing:**
Guard comment added at Portainer deploy step — do not deploy until all intended worker nodes have joined.

**Bug #19 — virtiofs permissions** (runs on Proxmox host after Docker first starts):
```bash
chmod -R 755 /mnt/docker-data/
```

---

## ✅ Bugs Closed

- ✅ **Bug #5** — Stale Swarm state DB — cleared in `03-post-boot.sh` before re-init
- ✅ **Bug #6** — Portainer agent timing — guard added to `03-post-boot.sh`
- ✅ **Bug #19** — virtiofs permissions — `chmod -R 755` added to `03-post-boot.sh` post Docker start

---

## 📊 Current State

| Node | Role | Status | Availability |
|------|------|--------|--------------|
| `manager-01` (10.0.60.30) | Leader | Ready | Active |
| `traefik-dmz-01` | Worker | Ready | Active |

| Stack | Status |
|-------|--------|
| Portainer | Ready to deploy (not yet deployed) |
| Traefik | Ready to deploy (not yet deployed) |
| Monitoring | Planned — next session |

---

## ➡️ Next Session

> [!note] Deploy order is critical — follow this sequence

1. **Deploy Portainer** — `docker stack deploy -c /mnt/docker-swarm/stacks/portainer.yml portainer`
   - Verify agent connects on both nodes via Portainer UI (port 9000)
2. **Deploy Traefik** — requires `traefik-public` overlay network to exist first
   - Verify dashboard accessible, routing working
3. **Deploy Monitoring stack** — Prometheus + Grafana + cAdvisor + node-exporter
4. **Plan database migration** from old Docker Compose server via ZFS send/receive

---

## 🔗 Related Notes

- [[Docker Swarm Infrastructure Runbook]]
- [[Swarm Deployment Overview]]
- [[Session Notes — 2026-04-12 — Monitoring Deploy]]
