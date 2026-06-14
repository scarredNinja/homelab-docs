---
project_id: Homelab-2025
phase: 'Phase 3: Network Config'
status: Reference
tags:
  - networking
  - pfsense
  - vpn
  - reference
---
> [!warning] Superseded
> This approach (pfSense policy routing + `VPN_Clients` alias) was replaced by **Gluetun** running as Docker Compose on `worker-mediamanagement-01`. Gluetun handles the NordVPN OpenVPN tunnel at the container level — no pfSense policy routing required. Tailscale handles admin VPN access (Phase 9). This doc is retained for reference only.

# pfSense VPN Policy Routing & Kill Switch (NordVPN)

## 1. Create an Alias (The "Clean" Way)

Don't use raw IP addresses in your firewall rules. Create an Alias in pfSense (`Firewall > Aliases`).

  

• Name: `VPN_Clients`

  

• Values: `10.0.50.12` (Transmission IP)

  

• Benefit: If you ever add another downloader, you just add the IP to this list, and the rules apply automatically.

  

## 2. Configure the NordVPN Interface

Ensure NordVPN is set up as an Interface in pfSense (usually `VPN_WAN`).

  

• Ensure Don't pull routes is checked in the OpenVPN/WireGuard client settings. This prevents NordVPN from taking over your entire internet connection the moment it connects.

  

## 3. The Firewall Rules (The "Kill Switch")

Go to `Firewall > Rules > VLAN50 (Media)`. You need two specific rules at the top of your list:

  

Why this works:

  

• The Kill Switch: The "!" (Invert) rule says: "If this traffic is trying to go to the internet (not local) via the default gateway, Kill it."

  

• The Tunnel: This rule tells pfSense: "If the source is Transmission, ignore the normal route and push it out the NordVPN tunnel."

  

## 4. Handling Port Forwarding (The "Messy" Part)

NordVPN does not support traditional port forwarding. This means:

  

• Your torrents will still work (via DHT/PEX), but you will be "Passive."

  

• You may see slower speeds or have trouble connecting to specific peers.

  

• Workaround: Ensure Prowlarr/Sonarr are NOT behind the VPN so they can reach trackers/indexers reliably; leave only the downloader (Transmission) behind the tunnel.

  

## 5. DNS Leak Protection

In your Transmission container settings (or the VM hosting it), manually set the DNS to NordVPN’s DNS servers (e.g., `103.86.96.100`). This ensures that even if the tunnel is up, your ISP doesn't see what you are searching for via DNS queries.

  

## Your Updated IP Strategy

| **Service**      | **VLAN** | **IP**     | **Gateway** | **Route**         |
| ---------------- | -------- | ---------- | ----------- | ----------------- |
| **Transmission** | 50       | 10.0.50.12 | 10.0.50.1   | Forced to NordVPN |
| **Prowlarr**     | 50       | 10.0.50.13 | 10.0.50.1   | Standard WAN      |
