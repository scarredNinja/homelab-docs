---
phase: 'Phase 5: Docker Swarm'
project_id: Homelab-2025
session_type: Code + Configuration
status: Completed
tags:
  - DockerSwarm
  - Network
  - ExternalAccess
---
# Session Notes - 2026-06-02 - Seerr External Access Configuration

Expose Seerr (`seerr.purvishome.com`) externally using the existing Cloudflare Tunnel (`cloudflared`) daemon while implementing Split-Horizon DNS for local performance.

## Session Goal
Configure secure, high-performance external and internal access routes for Seerr.

## Changes Completed

### 1. Codebase (Docker Swarm Stacks)
* **`proxmox-swarm/stacks/stack-arr.yml`**: Split the single Traefik router for Seerr into:
  - `seerr-internal` -> matching `seerr.home.purvishome.com` (for local LAN / Tailscale)
  - `seerr-external` -> matching `seerr.purvishome.com` (for external WAN via Split-Horizon DNS)
  - Validated Compose file syntax successfully: `docker compose -f proxmox-swarm/stacks/stack-arr.yml config --quiet`.

### 2. Obsidian Documentation Sync
* **`Service - seerr.md`**: Updated `service_status` to `"in-progress"` and defined `url_external` as `https://seerr.purvishome.com`.
* **`Network DNS Mapping.md`**: Added `Seerr` to the network DNS routing matrix with both internal and external domains.
* **`Docker Swarm Details.md`**: Updated Seerr status in the service matrix to `"In Progress"`.

---

## Action Items (Required from Operator / User)

> [!important]
> To complete the deployment, you must perform these manual steps in your Cloudflare and Pi-hole dashboards:

### Step 1: Configure Cloudflare Tunnel Ingress
1. Log into the **Cloudflare Zero Trust Dashboard**.
2. Navigate to **Networks** -> **Tunnels** and edit your active homelab tunnel.
3. In the **Public Hostnames** tab, click **Add a public hostname**:
   - **Subdomain**: `seerr`
   - **Domain**: `purvishome.com`
   - **Path**: *Leave blank*
   - **Service Type**: `HTTP`
   - **URL**: `seerr:5055` (points to the Seerr service container over overlay networks)
4. Save the hostname. Cloudflare will automatically provision the DNS CNAME record.

### Step 2: Configure Split-Horizon Local DNS
1. Log into your Pi-hole admin console(s).
2. Go to **Local DNS** -> **DNS Records**.
3. Add a local mapping:
   - **Domain**: `seerr.purvishome.com`
   - **IP Address**: `10.0.60.40` (Traefik Ingress VIP)
4. Verify syncing via Gravity Sync if applicable.

### Step 3: Deploy Stack & Configure App
1. Redeploy the `arr` stack to apply the new labels:
   ```bash
   docker stack deploy -c /mnt/docker-swarm/stacks/arr/stack-arr.yml arr
   ```
   *(Or click "Update Stack" in the Portainer Stacks page for `arr`).*
2. Log into Seerr internally (`https://seerr.home.purvishome.com`).
3. Navigate to **Settings** -> **General** and change the **Application URL** to `https://seerr.purvishome.com`.
4. Click **Save Changes**.

---

## Verification Tasks
- [ ] Verify local resolution: `nslookup seerr.purvishome.com` inside LAN must return `10.0.60.40`.
- [ ] Verify external resolution: `nslookup seerr.purvishome.com 8.8.8.8` must return Cloudflare edge IPs.
- [ ] Verify external access: Navigate to `https://seerr.purvishome.com` on a mobile cellular connection and log in.
- [ ] Verify internal fallback: Confirm `https://seerr.home.purvishome.com` still loads correctly inside the LAN.
