---
project_id: AdvancedApps-2026
status: Planned
created_date: '2026-06-02'
tags:
  - homelab
  - advanced-apps
  - hosting
phase: Master Hub
---
# 🧠 Advanced Application Hosting — Master Hub

> [!NOTE]
> This master project hub tracks the deployment of resource-intensive, data-intensive, or security-sensitive application hosting services detached from the main cluster rebuild pipeline.

## Project Details
- **Project ID:** `AdvancedApps-2026`
- **Status:** 🔵 Planned
- **VLAN Focus:** VLAN 50 (Media plane) & VLAN 60 (Internal management plane)

---

## 🎯 Scope & Objectives
1. Enable secure password vault syncing across the homelab boundary.
2. Support photo backup, indexing, and machine-learning facial recognition stacks.
3. Deploy local S3 storage for distributed service backup replication.

---

## 📋 Deferred Project Tasklist

### 🔹 Immich (Self-Hosted Photos)
- [ ] **Immich Service Deployment:** Deploy Immich Swarm stack with dedicated postgresql database.
- [ ] **Machine Learning Pass-through:** Verify Immich-machine-learning service CPU routing constraints on Swarm.

### 🔹 Vaultwarden (Passwords)
- [ ] **Vaultwarden Deployment:** Deploy Vaultwarden stack behind Traefik proxy.
- [ ] **Database Encryption:** Configure daily backup and encrypted snapshotting of SQLCipher DB.

### 🔹 Object Storage & Backups
- [ ] **MinIO Object Storage:** Deploy MinIO distributed cluster on Swarm workers for local S3 targets.

---

## 🔗 Related Resources
- [[Docker Swarm Details]]
- [[Homelab Next Phase — Review & Roadmap]]
