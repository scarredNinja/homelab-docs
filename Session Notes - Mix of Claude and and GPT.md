---

project_id: Homelab-2025 
phase: "Phase 5: Docker Swarm" 
tags:

- DockerSwarm
- Networking
- Proxmox
- CloudInit
- Docker
- Debugging
- VMTemplate date: 2026-03-24

---

# 🧠 Homelab Provisioning Session Summary

## 🎯 Objective
Build a fully automated provisioning pipeline for:
- Proxmox VMs
- Docker Swarm (managers + workers)
- VLAN-aware dual-NIC networking
- Hybrid storage (virtiofs + NFS + optional ZFS)

---

## 🏗️ Architecture Overview

### Compute
- Docker Swarm cluster (managers + workers)
- Role-based node labeling

### Networking
- VLAN segmentation per workload
- Dual NIC design:
  - Primary: internal / management
  - Secondary: DMZ / external / service-specific

### Storage
- `virtiofs` → fast local storage (backed up via Proxmox)
- NFS → shared storage across nodes
- ZFS → optional for stateful workloads (DBs)

---

## 💥 Issue Encountered

### Symptoms
- VM stuck during boot:
Waiting for network on eth0...  
Device "eth0" does not exist.

- Network interfaces observed:

ens18 | False  
ens19 | False

  
---  
  
## 🔍 Root Cause  
  
### 1. Interface Naming Assumption  
- Script assumed:  
- `eth0`, `eth1`  
- Actual system used:  
- `ens18`, `ens19`  
  
➡️ Netplan rename (`set-name: eth0`) **did not apply**  
  
---  
  
### 2. Netplan Applied Too Late  
- Config written via `write_files`  
- Applied during `runcmd`  
  
➡️ Network stage already failed before config applied  
  
---  
  
### 3. Hard Dependency on `eth0`  
Used in:  
- wait-online override  
- cloud-init network wait loop  
- Docker startup logic  
  
➡️ Since `eth0` didn’t exist → cascading failure  
  
---  
  
### 4. Secondary Factor  
- Both NICs were `DOWN`  
- Likely causes:  
- DHCP delay  
- VLAN / tagging mismatch  
- pfSense DHCP not reached  
  
---  
  
## 🛠️ Fixes Implemented  
  
### ✅ Remove Hardcoded Interface Names  
- Replace `eth0` usage with dynamic detection:  
```bash  
ip route | awk '/default/ {print $5}'
```
---

### ✅ Fix Netplan Timing

- Move application earlier in boot process:
    
    bootcmd:  
      - cloud-init-per once netplan-generate netplan generate  
      - cloud-init-per once netplan-apply netplan apply
    

---

### ✅ Optional Secondary NIC

- Prevent boot blocking:
    
    optional: true
    

---

### ✅ Fix NFS Mount Behavior

- Replace blocking mounts with:
    
    _netdev,x-systemd.automount,noatime
    
- Remove `mount -a` from boot

---

### ✅ Improve Network Wait Logic

- Wait for active interface instead of `eth0`:
    
    IFACE=$(ip route | awk '/default/ {print $5}' | head -n1)
    

---

## ⚠️ Key Lessons

### 🧠 Never assume interface names

- Modern Linux uses:
    - `ens18`, `enp0sX`, etc.
- Renaming is fragile and timing-sensitive

---

### 🧠 Cloud-init timing matters

- `runcmd` = too late for network fixes
- Use:
    - `bootcmd`
    - proper netplan generation

---

### 🧠 Multi-NIC requires explicit handling

- Secondary NICs must be:
    - optional
    - or properly configured

---

### 🧠 Avoid blocking boot dependencies

- NFS mounts
- wait-online
- network assumptions

---

## 🧱 Current State

### ✅ Working Design Includes:

- VLAN-aware networking
- Dual NIC architecture
- Swarm-ready provisioning
- Hybrid storage model
- Node labeling system

---

## Session Outcome

> [!success] Closed — 2026-03-24
> Interface naming (ens18 vs eth0), multi-NIC netplan timing, and NFS mount
> blocking boot issues documented. All fixes incorporated into provisioning scripts
> in subsequent sessions.
>
> All outstanding tasks promoted to [[Swarm Topology]] and [[Docker Swarm Infrastructure Runbook]].
> → Current status: [[Swarm Topology]]
