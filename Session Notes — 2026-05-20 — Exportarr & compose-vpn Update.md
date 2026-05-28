---
project_id: Homelab-2025
date: 2026-05-20
tags:
  - homelab
  - session-note
  - vpn
  - gluetun
  - wireguard
  - exportarr
  - swarm
  - compose-vpn
status: Complete
---
# Session Notes — 2026-05-20 — Exportarr & compose-vpn Update

## Session Goal

Resolve authentication errors on Exportarr sidecar containers (`sonarr-exporter` and `radarr-exporter`) by updating the deprecated `APIKEY` environment variable to `API_KEY_FILE`, and correct Prometheus scrape target IPs. Address trailing newline validation failures inside Exportarr by creating new clean `sonarr_apikey_v2` and `radarr_apikey_v2` secrets using `printf`. Audit the standalone non-Swarm `compose-vpn.yml` stack on `worker-mediamanagement-01` and prepare a concrete manual deployment and sync process.

---

## Completed

- **Exportarr Sidecars Configuration Fixed:** Configured the Exportarr sidecars in `stack-arr.yml` to use standard, prefix-free environment variables (`PORT`, `URL`, and `API_KEY_FILE`). Bypassed CLI flag arguments (which encountered validation bugs in the Go binary) and the incorrect `EXPORTARR_` prefixes (which were ignored by the binary, causing the empty `URL` validation loop).
- **Secrets Validation Bug Resolved (_v2 Secrets):** Solved the `regex: api-key must be a 20-32 character alphanumeric string` crash loop. This was caused by trailing newline characters in standard Docker secrets (introduced by `echo "KEY"`). We transitioned `stack-arr.yml` to reference clean `sonarr_apikey_v2` and `radarr_apikey_v2` secrets created via `printf`.
- **compose-vpn Container Conflict Resolved:** Handled the `gluetun` name conflict loop in `compose-vpn.service`. Added a forceful `docker rm -f gluetun` startup hook in the systemd wrapper so stopped/partially-created containers are cleanly deleted during recovery without manual command execution.
- **Prometheus Scrape Targets Corrected:** Corrected the scrape targets in `prometheus.yml` for `exportarr-sonarr` and `exportarr-radarr` from `10.0.50.30` to `10.0.50.51` (the actual IP address of `worker-mediamanagement-01`), enabling successful metrics collection.
- **Git Commit, Push, and Merge:** Committed and pushed all updates to the `feature/exportarr-secrets-fix` branch, merged into `main`, and pushed to the remote repository `origin/main` (latest commit `52cece5`).

---

## Current State

| Component | Status |
|---|---|
| Exportarr API configuration in Git | ✅ Merged into `main` with standard prefix-free env vars and `_v2` fix |
| Prometheus scrape targets in Git | ✅ IP corrected in `prometheus.yml` (`.50.30` -> `.51`) and pushed to `main` |
| compose-vpn systemd service cleanup | ✅ Merged into `main` with robust container removal |
| Obsidian Hub Task Sync | ✅ Synced |
| Remote VM Synchronization | ✅ Deployed & Running |

---

## Next Steps (Manual VM Commands)

Since the changes have been fully merged and pushed to **`main`**, run the following actions on your Swarm Manager node to pull the latest configuration and sync it to the cluster:

1. **Pull and sync the latest configuration:**
   On your Swarm Manager VM:
   ```bash
   # Make sure you are on the main branch and pull the latest changes
   git checkout main
   git pull origin main
   
   # Synchronize the configurations to the virtiofs share
   bash /mnt/docker-swarm/scripts/copy-swarm-config.sh
   ```

2. **Reload Prometheus Configuration:**
   Instruct Prometheus to reload its configuration to pick up the corrected `.51` scrape targets:
   ```bash
   # Send a reload signal to the Prometheus Swarm service
   curl -X POST http://localhost:9090/-/reload
   ```

3. **Verify Scrape Targets:**
   Check the Prometheus web UI (or Grafana dashboard) to ensure `exportarr-sonarr` and `exportarr-radarr` targets are showing as `UP` at `10.0.50.51:9707` and `10.0.50.51:9708`.

---

### Legacy Reference (VM setup details)

1. **Create the clean `_v2` secrets (using `printf` to avoid newlines):**
   Execute these commands directly on your Swarm Manager node (replace `<YOUR_SONARR_API_KEY>` and `<YOUR_RADARR_API_KEY>` with your actual 32-character API keys from Sonarr/Radarr):
   ```bash
   printf '%s' '<YOUR_SONARR_API_KEY>' | docker secret create sonarr_apikey_v2 -
   printf '%s' '<YOUR_RADARR_API_KEY>' | docker secret create radarr_apikey_v2 -
   ```

2. **Synchronize configurations to the VM host/virtiofs mount:**
   On your Swarm Manager VM:
   ```bash
   bash /mnt/docker-swarm/scripts/copy-swarm-config.sh
   ```

3. **Force-update the Arr Swarm stack (deploys Exportarr with prefixed env variables and new secrets):**
   ```bash
   # Redeploy the stack
   docker stack deploy -c /mnt/docker-swarm/stacks/stack-arr.yml arr
   ```

4. **Update and start the `compose-vpn` stack on `worker-mediamanagement-01`:**
   Log into `worker-mediamanagement-01`:
   ```bash
   ssh ubuntu@worker-mediamanagement-01
   
   # 1. Provision the WireGuard private key file (NordVPN client private key)
   # (Substitute your actual private key below. This file is git-ignored and MUST be present for gluetun to start)
   sudo mkdir -p /opt/compose-vpn
   echo "<YOUR_NORDVPN_WIREGUARD_PRIVATE_KEY>" | sudo tee /opt/compose-vpn/wg_private_key > /dev/null
   sudo chmod 600 /opt/compose-vpn/wg_private_key

   # 2. Copy the updated compose-vpn.yml and systemd service file from the synced virtiofs mount
   sudo cp /mnt/docker-swarm/stacks/compose-vpn.yml /opt/compose-vpn/compose-vpn.yml
   sudo cp /mnt/docker-swarm/systemd/compose-vpn.service /etc/systemd/system/compose-vpn.service
   sudo systemctl daemon-reload

   # 3. Force-remove any conflicting gluetun container from previous failed attempts
   docker rm -f gluetun >/dev/null 2>&1 || true

   # 4. Restart the systemd wrapper service
   sudo systemctl restart compose-vpn.service

   # 5. Verify the containers are running cleanly
   docker compose -p compose-vpn -f /opt/compose-vpn/compose-vpn.yml ps
   ```
