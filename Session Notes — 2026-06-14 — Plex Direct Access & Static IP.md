---
date: '2026-06-14T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
session_type: Network Optimisation
status: Completed
tags:
  - SessionNotes
  - DockerSwarm
  - Plex
  - Networking
  - pfSense
  - Traefik
  - CGNAT
---

# Session Notes — 2026-06-14 — Plex Direct Access & Static IP

## Session Goal

Resolve remote streaming slowness and buffering on Plex (`plex.purvishome.com`) by bypassing the bandwidth-throttled Cloudflare Tunnel and setting up a secure, direct WAN access path via pfSense.

---

## Diagnostic Path

| Observation | Conclusion |
|---|---|
| Plex remote streams buffer constantly and fall back to low quality | Cloudflare Free Tier tunnel actively throttles high-bandwidth media streams |
| Disabling the tunnel requires direct inbound routing | Inbound port forwarding is required |
| WAN interface originally showed IP `100.71.224.X` | Homelab is behind Carrier-Grade NAT (CGNAT) where inbound ports are blocked |
| ISP port filtering disable setting did not remove CGNAT | ISP port filtering only unblocks outbound ports; static IP is required to exit CGNAT |
| User purchased a Static IP ($10/month) | Dedicated public IP will be assigned directly to WAN, enabling port forwarding |

---

## Architecture Plan: Direct DMZ Ingress

To keep the management network secure, incoming external connections land strictly on Traefik's DMZ interface rather than its internal LAN interface:

```mermaid
graph TD
    Client[External Plex Client] -->|HTTPS Port 443| pfSense[pfSense Router (Static WAN IP)]
    pfSense -->|NAT Port Forward| Traefik[Traefik Ingress Node: 10.0.80.20]
    Traefik -->|Docker Overlay Network| Plex[Plex Service: 10.0.50.50]
```

---

## Completed

### 1. Stack Configuration Updates (`stack-plex.yml`)
* Removed the `cloudflared` tunnel service definition and its `cloudflare_tunnel_token` secret.
* Verified `ADVERTISE_IP` is set to `"https://plex.purvishome.com:443"`.
* Redeployed the `plex` stack via Portainer using the `feat/plex-direct-access` branch.

### 2. Cloudflare DNS Changes
* Set `plex.purvishome.com` as a CNAME pointing to `ddns.purvishome.com`.
* Switched Proxy status to **DNS-Only (Grey Cloud)** to bypass Cloudflare's proxy network completely.

### 3. pfSense NAT & Firewall Configurations
* Created a WAN Port Forward NAT rule:
  * **Protocol:** TCP
  * **Destination Port:** `443` (HTTPS)
  * **Redirect Target IP:** `10.0.80.20` (Traefik DMZ VLAN 80 IP)
  * **Redirect Target Port:** `443`
* Configured the associated WAN Firewall rule:
  * **Source:** `*` (Any client)
  * **Destination:** `10.0.80.20`
  * **Port:** `443`
  * **Action:** Pass

---

## Current State (Parked / Pending Activation)

| Subsystem | Status | Note |
|---|---|---|
| `Plex Stack` | ✅ Healthy | Running on `worker-media-01` without the `cloudflared` sidecar. |
| `Traefik` | ✅ Healthy | Listening on `0.0.0.0:443` (both `10.0.60.40` and `10.0.80.20` interfaces). |
| `pfsense Rules` | ✅ Configured | NAT forward target set to `10.0.80.20` and firewall rule applied. |
| `WAN IP` | ⏳ Pending | Currently still on `100.71.224.X` (CGNAT) pending ISP static IP activation. |

---

## Parked Follow-up Tasks

- [x] **Force DHCP WAN Renewal:** Once the ISP confirms static IP activation, go to **Interfaces** -> **WAN** in pfSense, release, and renew the lease (or reboot the ONT) to pull the new static IP. ✅ 2026-06-15
- [x] **Verify DDNS Update:** Ensure pfSense updates `ddns.purvishome.com` to the new static IP. ✅ 2026-06-15
- [x] **Validate Port Reachability:** Check if port 443 shows as reachable via external tools (`ifconfig.co/port/443`). ✅ 2026-06-15
- [x] **Remote Playback Test:** Stream from a mobile device (off-site/mobile data) and verify Direct Play / Remote connection status in Plex dashboard / Tautulli. ✅ 2026-06-15

---

## Related Notes

- [[Service - plex]]
- [[Service - traefik]]
- [[VM - traefik-dmz-01]]
- [[Traefik Routing Architecture]]
