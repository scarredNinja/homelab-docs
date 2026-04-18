---
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - Traefik
  - DockerSwarm
  - Networking
  - DMZ
  - Security
date: 2026-04-02
---

# Traefik Routing Architecture

> [!abstract] Overview
> Single Traefik instance on `traefik-dmz-01` handles all ingress — both external
> (internet-facing) and internal (VLAN 60 only). Separation is enforced by
> entrypoints bound to separate NICs, not by running multiple Traefik instances.
> External access beyond Plex goes through pfSense WireGuard VPN — not through
> the DMZ. This minimises public attack surface to a single port.

---

## Network Position

```
Internet
    │
    ├── Port 443 (Cloudflare IPs only)
    │       └── pfSense NAT → traefik-dmz-01 VLAN 80 NIC
    │                               └── websecure entrypoint
    │                                       └── Plex (only public service)
    │
    └── VPN port (WireGuard on pfSense)
            └── VPN client gets VLAN 70 equivalent access
                    └── traefik-dmz-01 VLAN 60 NIC
                            └── internal entrypoint
                                    └── All other services
```

`traefik-dmz-01` is dual-homed:
- `eth0` — VLAN 60 (Management/Infrastructure) — internal entrypoint
- `eth1` — VLAN 80 (DMZ) — websecure entrypoint, internet-facing

---

## Why One Instance, Not Two

Running two Traefik containers (one internal, one external) adds complexity
and a second attack surface with no security benefit at homelab scale.

The security boundary is enforced by:
1. pfSense firewall rules — port 443 restricted to Cloudflare IPs only
2. VLAN 80 rules — DMZ cannot initiate connections to RFC1918 addresses
3. Entrypoints bound to specific NIC IPs — not `0.0.0.0`

One instance, two entrypoints, two NICs. Clean and auditable.

---

## Entrypoint Definitions

Add to `traefik.yml` — bind each entrypoint to its specific NIC IP to prevent
cross-contamination between the two contexts:

```yaml
entryPoints:
  # External — VLAN 80 NIC only, internet-facing
  websecure:
    address: '10.0.80.x:443'
    forwardedHeaders:
      trustedIPs:
        # Cloudflare IP ranges — update from https://www.cloudflare.com/ips/
        - 173.245.48.0/20
        - 103.21.244.0/22
        - 103.22.200.0/22
        - 103.31.4.0/22
        - 141.101.64.0/18
        - 108.162.192.0/18
        - 190.93.240.0/20
        - 188.114.96.0/20
        - 197.234.240.0/22
        - 198.41.128.0/17
        - 162.158.0.0/15
        - 104.16.0.0/13
        - 104.24.0.0/14
        - 172.64.0.0/13
        - 131.0.72.0/22

  # Internal — VLAN 60 NIC only, LAN/VPN access
  internal:
    address: '10.0.60.x:8443'
```

> [!important] Replace `10.0.80.x` and `10.0.60.x` with the actual static IPs
> assigned to `traefik-dmz-01` on each VLAN. Check pfSense DHCP reservations.

> [!note] Why port 8443 for internal?
> Port 443 is reserved for the external entrypoint. Using 8443 internally
> prevents any ambiguity and makes firewall rules explicit. VPN clients
> reaching `service.home.yourdomain.com:8443` is fine — browsers handle
> non-standard ports transparently when Traefik sets the correct headers.
> Alternatively use 443 on the internal NIC if you want clean URLs — just
> ensure pfSense rules only forward 443 from WAN to the VLAN 80 IP, not
> the VLAN 60 IP.

---

## Service Routing Split

### External — `websecure` entrypoint

Internet-facing only. Requires Cloudflare DNS A record pointing to pfSense WAN IP.

| Service | Notes |
|---|---|
| **Plex** | Only service that requires inbound internet connection. Plex remote access and account auth need it. |
| **Public website** | If you ever host one — no auth needed, just a `websecure` router |
| **Authelia** | Auth portal — needs `websecure` so the callback URL is reachable from internet during external auth flows |
| **Any future public service** | Deploy on appropriate worker, add `websecure` router labels |

> [!warning] Keep this list short
> Every service on `websecure` is a public attack surface. Default answer is
> always `internal`. Only add to `websecure` when internet access is a hard
> requirement, not a convenience.

---

### Internal — `internal` entrypoint

Reachable only from VLAN 60, VLAN 70, and VPN clients. No public exposure.

| Service | Worker | Port |
|---|---|---|
| Portainer | manager-01 | 9000 |
| Grafana | manager-01 | 3000 |
| Prometheus | manager-01 | 9090 |
| InfluxDB | manager-01 | 8086 |
| Netbox | manager-01 | 8080 |
| Sonarr | worker-media-01 | 8989 |
| Radarr | worker-media-01 | 7878 |
| Prowlarr | worker-media-01 | 9696 |
| Tautulli | worker-media-01 | 8181 |
| Transmission | worker-media-01 | 9091 |
| Home Assistant | worker-controller-01 | 8123 |
| UniFi | worker-controller-01 | 8443 |
| Traefik dashboard | traefik-dmz-01 | 8080 |

**Non-Docker services** (Proxmox, PiHole) — cannot use stack labels.
Configure via Traefik file provider in `dynamic.yml`:

```yaml
# dynamic.yml — file provider for non-Docker services
http:
  routers:
    proxmox-int:
      rule: "Host(`proxmox.home.yourdomain.com`)"
      entryPoints:
        - internal
      tls: {}
      service: proxmox-svc

    pihole-int:
      rule: "Host(`pihole.home.yourdomain.com`)"
      entryPoints:
        - internal
      tls: {}
      service: pihole-svc

  services:
    proxmox-svc:
      loadBalancer:
        servers:
          - url: "https://10.0.60.50:8006"
        passHostHeader: true

    pihole-svc:
      loadBalancer:
        servers:
          - url: "http://10.0.60.x:80"
```

---

## Stack Label Conventions

### Internal service (standard pattern)

```yaml
deploy:
  labels:
    - traefik.enable=true
    - traefik.http.routers.portainer-int.rule=Host(`portainer.home.yourdomain.com`)
    - traefik.http.routers.portainer-int.entrypoints=internal
    - traefik.http.routers.portainer-int.tls=true
    - traefik.http.services.portainer-int.loadbalancer.server.port=9000
```

### External service — Plex

```yaml
deploy:
  labels:
    - traefik.enable=true
    - traefik.http.routers.plex-ext.rule=Host(`plex.yourdomain.com`)
    - traefik.http.routers.plex-ext.entrypoints=websecure
    - traefik.http.routers.plex-ext.tls=true
    - traefik.http.routers.plex-ext.tls.certresolver=cloudflare-origin
    - traefik.http.routers.plex-ext.middlewares=cloudflare-ips@file
    - traefik.http.services.plex-ext.loadbalancer.server.port=32400
```

### External service with Authelia (future pattern)

```yaml
deploy:
  labels:
    - traefik.enable=true
    - traefik.http.routers.myservice-ext.rule=Host(`myservice.yourdomain.com`)
    - traefik.http.routers.myservice-ext.entrypoints=websecure
    - traefik.http.routers.myservice-ext.tls=true
    - traefik.http.routers.myservice-ext.tls.certresolver=cloudflare-origin
    - traefik.http.routers.myservice-ext.middlewares=authelia@docker
    - traefik.http.services.myservice-ext.loadbalancer.server.port=XXXX
```

> [!note] Router naming convention
> Suffix `-int` for internal routers, `-ext` for external. Prevents name
> collisions and makes intent obvious when reading stack files. If a service
> genuinely needs both (e.g. Authelia itself), it gets two routers:
> `authelia-int` and `authelia-ext`.

---

## External Access Decision Framework

```
Does this service need to be reachable without VPN?
    │
    No ──→ internal entrypoint only. Done.
    │
    Yes
    │
    ├── Public with no auth? (website, public API, Plex)
    │       └── websecure, no middleware (Plex has its own auth)
    │
    ├── Personal/admin service needing remote access?
    │       └── websecure + Authelia middleware
    │           (deploy Authelia first — see section below)
    │
    └── Something that only needs occasional external access?
            └── Consider VPN instead — simpler, no extra exposure
```

---

## VPN External Access (Primary Path for Everything Except Plex)

pfSense WireGuard VPN gives connected clients VLAN 70 equivalent access.
VLAN 70 has access to VLAN 60 services. This means:

**VPN connected → Traefik internal entrypoint → all internal services**

No additional firewall rules needed beyond existing VLAN 70 rules.
No Authelia needed on the VPN path — WireGuard authentication is sufficient.

This is the correct path for:
- Remote access to Portainer, Grafana, HA, Proxmox UI
- Accessing Sonarr/Radarr/Prowlarr remotely
- Any admin task done away from home

> [!tip] NordVPN vs homelab VPN
> The NordVPN config in pfSense is outbound-only (Transmission kill switch).
> The homelab VPN for remote access is a separate inbound WireGuard instance
> on pfSense — different interface, different purpose. Do not confuse them.

---

## Adding Authelia (Future)

Only needed when you want to expose personal services externally without
requiring VPN. Not needed if VPN covers all your remote access needs.

**Placement:** `manager-01` — internal service, `node.role=manager` constraint.

**Authelia needs two routers:**
```yaml
# Internal — so you can access the auth portal from inside the network
- traefik.http.routers.authelia-int.entrypoints=internal
- traefik.http.routers.authelia-int.rule=Host(`auth.home.yourdomain.com`)

# External — so the callback URL works during external auth flows
- traefik.http.routers.authelia-ext.entrypoints=websecure
- traefik.http.routers.authelia-ext.rule=Host(`auth.yourdomain.com`)
- traefik.http.routers.authelia-ext.tls.certresolver=cloudflare-origin
```

**Dependencies before deploying Authelia:**
- Postgres or MySQL instance (use `rpool/docker-db` dataset)
- Redis for session storage
- SMTP relay for 2FA emails (or use a local mail relay)

> [!warning] Do not deploy Authelia speculatively
> It has real infrastructure dependencies and ongoing maintenance. Deploy it
> only when you have a concrete use case — i.e. a specific service you want
> externally accessible without VPN. Until then, VPN is simpler and equally
> secure.

---

## Adding a Public Website or New External Service

No VM layout changes needed. The process is:

1. Deploy the container on `worker-general-01` (zone=homelab, VLAN 40)
2. Add `websecure` router labels to the stack
3. Add Cloudflare DNS A record → pfSense WAN IP
4. pfSense already forwards port 443 to `traefik-dmz-01` — nothing changes

The Swarm overlay network handles routing from `traefik-dmz-01` to
`worker-general-01` transparently. Traefik doesn't need to be on the same
node as the service it proxies.

> [!note] VLAN 80 → VLAN 40 routing
> Check pfSense rules allow `traefik-dmz-01` (VLAN 80) to reach the overlay
> network ports on `worker-general-01` (VLAN 40). Swarm overlay uses:
> - TCP/UDP 7946 (node discovery)
> - UDP 4789 (VXLAN overlay traffic)
> These need to be open between all Swarm nodes regardless of VLAN.

---

## Required pfSense Rules Summary

| Rule | Interface | Source | Destination | Port | Action |
|---|---|---|---|---|---|
| Cloudflare inbound | WAN | Cloudflare IPs alias | traefik-dmz-01 VLAN80 IP | 443 | Allow |
| Block all other 443 | WAN | any | any | 443 | Block |
| VLAN60 → Traefik internal | VLAN60 | 10.0.60.0/25 | traefik-dmz-01 VLAN60 IP | 8443 | Allow |
| VLAN70 → Traefik internal | VLAN70 | 10.0.70.0/25 | traefik-dmz-01 VLAN60 IP | 8443 | Allow |
| VPN clients → Traefik internal | VPN interface | VPN pool | traefik-dmz-01 VLAN60 IP | 8443 | Allow |
| DMZ → Swarm overlay | VLAN80 | traefik-dmz-01 | all Swarm node IPs | 7946, 4789 | Allow |
| DMZ → service ports | VLAN80 | traefik-dmz-01 | worker IPs | service ports | Allow |

> [!warning] DMZ block rule order matters
> The existing VLAN 80 rule `Block VLAN80 → RFC1918` must come AFTER the
> explicit allow rules above, or Traefik cannot reach backend services.
> pfSense rules are evaluated top-to-bottom, first match wins.

---

## TLS Strategy

**External (`websecure`):** Cloudflare Origin Certificate
- Free, 15-year validity, no ACME renewal needed
- Requires Cloudflare SSL mode: Full (strict)
- Generate at: Cloudflare Dashboard → SSL/TLS → Origin Server → Create Certificate
- Store at `/mnt/docker-swarm/traefik/certs/`

**Internal (`internal`):** Self-signed wildcard or local CA
- `*.home.yourdomain.com` wildcard covers all internal services
- Current config in `tls.yaml` uses `/certs/internal-wildcard.crt`
- Browsers will warn unless you import the CA cert — do this on your devices
  and VPN clients

---

## Related

- [[Docker Swarm Infrastructure Runbook]] — Phase 1 Traefik migration steps
- [[pfSense Firewall Rules]] — VLAN 80 DMZ rules
- [[VLAN and Subnet Summary Sheet]]
- [[Swarm Topology]]
- [[pfSense VPN Policy Routing & Kill Switch (NordVPN)]]
