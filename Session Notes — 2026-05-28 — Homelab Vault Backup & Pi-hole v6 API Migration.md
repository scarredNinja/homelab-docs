---
date: 2026-05-28
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm — Dev Sandboxes & Ingestion Setup"
session_type: "Backup & Config"
status: "Complete"
tags:
  - SessionNotes
  - Backup
  - Pihole
  - Homepage
---

# Session Notes — 2026-05-28 — Homelab Vault Backup & Pi-hole v6 API Migration

## 🎯 Session Goal
Perform a full session backup of all local documentation and automation scripts to GitHub, and resolve the `API Error: Bad Request` roadblock on the Homepage dashboard for the Pi-hole DNS resolvers.

---

## ✅ Completed

### 1. Full Workspace Backup [SUCCESS 🟢]
*   **Obsidian Notes Repo (`homelab-docs`)**: Resolved SSH publickey authentication failures by migrating the git remote URL to HTTPS. Successfully staged, committed, and pushed the complete backlog of Obsidian session notes, VM definitions, and Phase 5 checklist updates to the GitHub `main` branch.
*   **Infrastructure Repo (`docker-swarm-home`)**: Successfully staged, committed, and pushed local PowerShell automation scripts (`scheduled-changelog.ps1`, `scheduled-progress.ps1`) and the `.claude/agents` configuration files to `main`.

### 2. Pi-hole Widget Roadblock Diagnosed & Solved [SUCCESS 🟢]
*   **v6 REST API Diagnosis**: Identified that the Pi-holes are running **Pi-hole v6**. Successfully verified that Pi-hole v6 completely deprecates the old `/admin/api.php` endpoint and exposes a secure REST API under `/api`.
*   **Homepage Configuration Migration**: Migrated `services.yaml` to include `version: 6` under both `pihole` widgets and defined a placeholder for the Pi-hole v6 App Password (`key`). Committed and pushed directly to the `main` branch.

---

## ➡️ Next Steps
- [ ] **Generate Pi-hole App Passwords**:
  1. Log into your Pi-holes at `http://10.0.60.20/admin` and `http://10.0.60.21/admin`.
  2. Navigate to **System > Settings > Web interface/API**.
  3. Toggle the view mode from **Basic** to **Expert**.
  4. Generate a new **App Password** for the Homepage dashboard under the **Configure app password** section.
- [ ] **Update key in `services.yaml`**: Replace `<api_token_or_app_password>` in `proxmox-swarm/stacks/homepage/services.yaml` with the generated app passwords.
- [ ] **Sync Swarm Config**:
  Run the copy script on the Proxmox host to apply the configuration:
  ```bash
  git pull && sudo bash proxmox-swarm/scripts/copy-swarm-config.sh
  ```
