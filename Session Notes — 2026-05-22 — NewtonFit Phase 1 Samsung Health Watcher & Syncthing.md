---
phase: "Phase 1: Ingestion & Target Sync"
project_id: FitnessDev-2026
date: 2026-05-22
status: In Progress
tags:
  - newtonfit
  - fitness-app
  - samsung-health
  - syncthing
  - session-notes
---

# 📝 Session Notes — 2026-05-22 — NewtonFit Phase 1: Samsung Health Watcher & Syncthing

> **Phase**: Phase 1 — Ingestion & Target Sync
> **Session Date**: 2026-05-22
> **Hub**: [[05 Personal Fitness & Health - Phase 1 Ingestion & Target Sync Hub]]

---

## ✅ Accomplished This Session

### 1. Samsung Health Auto-Watcher Implemented

- Added a **chokidar file watcher** for the `samsung_health` imports folder in `server.js`.
- **Drop path**: `data/imports/samsung_health/` (in the `friendly-newton` project root).
- Supports both **CSV** and **JSON** file formats.
- On file drop: auto-ingests biometric data, merges with the database, and archives the file.
- **No manual intervention required** — any Health Sync export dropped into the folder is processed immediately.

### 2. Option B (Health Sync Local File) Chosen

- The phone-side **Tasker Health Connect plugin** was not visible/installed on the device.
- Switched to **Option B**: using the **Health Sync** app to export Samsung Health data to a local directory on the phone.
- Health Sync writes `weight.json` and `resting_heart_rate.json` to a local folder (e.g. `/storage/emulated/0/Download/HealthSync/`).
- This folder is to be synced to the PC via **Syncthing** → NewtonFit auto-ingests on arrival.

### 3. Android Automation Setup Guide Restructured

- The guide at [[Android Automation Setup]] was reorganised into **two clear stages**:
  - **Stage 1: Local PC Ingestion** *(current / active)* — Syncthing mirrors to PC, Express watcher ingests.
  - **Stage 2: Production Homelab Deployment** *(future)* — Transition Syncthing targets to Docker Swarm stack.
- Both Pathway A (Syncthing) and Pathway B (Tasker HTTP POST) are fully documented.

### 4. Syncthing Phone-to-PC Connection — Blocked 🔴

- Attempted to connect Syncthing on Android to the Windows PC at `10.0.4.85:22000`.
- Connection could **not be established**. Steps tried:
  - Changed Windows network profile to **Private** (to allow device discovery).
  - Reviewed **NordVPN** settings and kill-switch rules.
- **Root cause suspected**: NordVPN routing or VLAN/subnet isolation is blocking inbound connections on port `22000`.
- **Status**: Unresolved — carry-over to next session.

---

## 🔴 Outstanding Tasks — Next Session

### 🔌 1. Debug Syncthing Phone-to-PC Connection

**Goal**: Establish a working Syncthing sync between the Android phone and the Windows PC on the LAN.

- [ ] Try setting a **static listen address** in PC Syncthing options: `tcp://10.0.4.85:22000` [priority:: 1] #Syncthing
- [ ] Check if the Android phone is on a **guest/IoT VLAN** (separate subnet that blocks device-to-device traffic) [priority:: 1] #Syncthing
- [ ] Review **NordVPN kill-switch** settings — disable kill-switch temporarily and test Syncthing connectivity [priority:: 1] #Syncthing
- [ ] Check Windows Defender Firewall — ensure **inbound rule for port 22000** is allowed for Private networks [priority:: 2] #Syncthing
- [ ] Confirm both devices are on the same subnet (phone `10.0.x.x` vs PC `10.0.4.85`) [priority:: 2] #Syncthing

**Key network details**:
| Field | Value |
| :--- | :--- |
| PC LAN IP | `10.0.4.85` |
| Syncthing port | `22000` (TCP) |
| Syncthing Web UI | `http://localhost:8384` |
| Samsung Health drop path | `data/imports/samsung_health/` |
| Health Sync phone folder | `/storage/emulated/0/Download/HealthSync/` |

### ⏰ 2. Schedule Obsidian Sync PowerShell Script

- [ ] Register the daily sync script as a Windows Task Scheduler job via `Register-ScheduledTask` [priority:: 2] #Automation
- [ ] Script location: `automation/sync-obsidian-goals.ps1` in `docker-swarm-home` repo
- [ ] Set trigger: **Daily at 08:00 AM** (or preferred morning time)
- [ ] Verify the task runs correctly and updates [[Physical Goals & Biomarkers]] frontmatter

### 🚀 3. Deploy Production Updates to Docker Swarm

- [ ] Once local ingestion tests pass, deploy updated `server.js` (with samsung_health watcher) to `scarredNinja/docker-swarm-home` stack [priority:: 3] #Swarm
- [ ] Ensure the persistent volume mount at `/mnt/docker-data/newtonfit/imports/samsung_health/` is present on the Swarm
- [ ] Update Syncthing targets on Android to point to homelab Syncthing (Stage 2 transition)

---

## 🔧 Technical Reference

### NewtonFit Local Dev Server

| Field | Value |
| :--- | :--- |
| Server URL | `http://10.0.4.85:8080` |
| Project root | `C:\Users\DJ\Documents\antigravity\friendly-newton\` |
| FitNotes import path | `data/imports/fitnotes/` |
| Samsung Health import path | `data/imports/samsung_health/` |

### Production

| Field | Value |
| :--- | :--- |
| Production URL | `https://newtonfit.home.purvishome.com` |
| Swarm import path (FitNotes) | `/mnt/docker-data/newtonfit/imports/fitnotes/` |
| Swarm import path (Samsung Health) | `/mnt/docker-data/newtonfit/imports/samsung_health/` |

### Health Sync App Setup (Option B)

- **Source**: Samsung Health
- **Destination**: Local File / Directory
- **Data types**: Weight, Resting Heart Rate
- **Export format**: JSON
- **Phone export path**: `/storage/emulated/0/Download/HealthSync/`
- **Output files**: `weight.json`, `resting_heart_rate.json`

---

## 🔗 Related Notes

- [[05 Personal Fitness & Health - Phase 1 Ingestion & Target Sync Hub]]
- [[Android Automation Setup]]
- [[Physical Goals & Biomarkers]]
- [[NewtonFit - Data Ingestion & Target Sync Strategy]]
- [[Service - newtonfit]]
