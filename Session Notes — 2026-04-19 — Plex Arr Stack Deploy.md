---
date: 2026-04-19
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm — Media Stack Deploy"
session_type: Planning + Deploy
status: Complete
tags:
  - SessionNotes
  - DockerSwarm
  - Traefik
  - ACME
  - Plex
  - Arr
  - NFS
---

# Session Notes — 2026-04-19

## 🎯 Session Goal

Verify Traefik cert resolver, deploy Plex and Arr stacks, validate NFS performance, begin testing media stack end-to-end.

---

## ✅ Traefik Cert Resolver — Verified Working

### What was confirmed

- `cloudflare.acme` provider loading correctly on startup
- All 10 domains challenged and certs issued successfully within ~60 seconds
- `delayBeforeCheck: "10s"` working — `Waiting for DNS record propagation` visible in logs
- DNS-01 challenge completing via Cloudflare API
- `internal` entrypoint listening on `0.0.0.0:8443` ✅
- `websecure` entrypoint listening on `0.0.0.0:443` ✅

### Correct traefik.yaml cert resolver config

```yaml
certificatesResolvers:
  cloudflare:
    acme:
      email: scarredninja360@gmail.com
      storage: /etc/traefik/acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "1.0.0.1:53"
        delayBeforeCheck: "10s"
```

### Issues found and fixed this session

|Issue|Fix|
|---|---|
|`storage:` field missing from cert resolver|Added `storage: /etc/traefik/acme.json`|
|`acme.json` permissions wrong (`755`)|`chmod 600` on Proxmox host-side path|
|Cloudflare secret name mismatch|Corrected secret name in stack file|
|Crash loop due to above|Resolved after all three fixes applied|

> [!warning] acme.json must be chmod 600 on Proxmox host virtiofs exposes host-side permissions directly. Must run on Proxmox host: `chmod 600 /mnt/docker-data/traefik/data/acme.json` This is Gotcha #11 in the runbook.

### Let's Encrypt Rate Limit Hit

Due to crash loop re-challenging repeatedly, rate limit was hit on production LE.

- Rate limit resets: ~**2026-04-26**
- Fix: use staging CA during debugging in future

```yaml
# Staging CA — add this during debugging, remove for production
caServer: "https://acme-staging-v02.api.letsencrypt.org/directory"
```

---

## ⚠️ Security Finding — internal-only Allowlist Stale Entries

Middleware `internal-only` contains two subnets not matching any defined VLAN:

- `10.0.4.0/24`
- `10.0.10.0/24`

These silently expand the trust boundary. Review and remove if legacy.

## ⚠️ InfluxDB Missing internal-only Middleware

InfluxDB router on `websecure` has no IP restriction — publicly routable if pfSense forward is open. Add label to `stack-monitoring.yml`:

```yaml
- "traefik.http.routers.influxdb.middlewares=internal-only@file"
```

## ⚠️ Stale Overlay Network

```
Network not found, id: yq8l0hfvtbekyh2hxqqkugwyi
```

Run on manager-01 to clean up:

```bash
docker network prune
```

---

## ✅ NFS Performance Validated

Tested cold read from Synology NAS on `worker-media-01`:

```bash
dd if=/mnt/media/Movies/"Sonic the Hedgehog (2020)"/"Sonic the Hedgehog (2020).mkv" \
  of=/dev/null bs=1M count=1000 status=progress
```

**Result: 128 MB/s** — well above 80 MB/s target. Direct VLAN 100 path confirmed working.

---

## ✅ Plex Stack Deployed

- Deployed via Portainer from `stacks/plex/stack-plex.yml`
- Config rsync'd from old server — existing libraries intact, no claim token needed
- NFS read-only mount confirmed at `/mnt/media`
- Plex ownership chown pass completed on startup

---

## ✅ Arr Stack Deployed

- Deployed via Portainer from `stacks/arr/stack-arr.yml`
- NordVPN WireGuard key generated via NordVPN API and stored as Docker secret
- Secrets created:
    
    ```bash
    printf '%s' 'nordvpn'        | docker secret create vpn_service_provider -printf '%s' 'wireguard'      | docker secret create vpn_type -printf '%s' '<KEY>'          | docker secret create wireguard_private_key -
    ```
    

### NordVPN WireGuard key retrieval

```powershell
$token = "<ACCESS_TOKEN>"
$headers = @{
    Authorization = "Basic " + [Convert]::ToBase64String(
      [Text.Encoding]::ASCII.GetBytes("token:$token"))
}
$creds = Invoke-RestMethod `
  -Uri "https://api.nordvpn.com/v1/users/services/credentials" `
  -Headers $headers
$creds.nordlynx_private_key
```

---

## 🐛 Sonarr Not Binding on Port 8989

### Symptoms

- `ss -tlnp | grep 8989` returns nothing on `worker-mediamanagement-01`
- Traefik returns `502` for `sonarr.home.purvishome.com`
- Sonarr logs show RSS sync running — app itself is alive

### config.xml looks correct

```xml
<Port>8989</Port>
<UrlBase></UrlBase>
<BindAddress>*</BindAddress>
<EnableSsl>False</EnableSsl>
<AuthenticationMethod>Basic</AuthenticationMethod>
```

### Suspected cause

`AuthenticationMethod: Basic` is deprecated in Sonarr v4 — new versions require `Forms`. This may prevent the web server from binding on startup.

### Next session fix

```bash
# Get container ID
ssh docker@worker-mediamanagement-01 "docker ps | grep sonarr"

# Check what's listening inside container
ssh docker@worker-mediamanagement-01 "docker exec <id> ss -tlnp"

# Check full startup logs
docker service logs arr_sonarr 2>&1 | head -50

# If AuthenticationMethod is the cause, edit config.xml:
# Change <AuthenticationMethod>Basic</AuthenticationMethod>
# To:    <AuthenticationMethod>Forms</AuthenticationMethod>
# Then restart service
```

---

## 🐛 Sonarr → Transmission Connection Failing

Sonarr logs show:

```
[Warn] Unable to retrieve queue and history items from Transmission
```

Transmission runs behind Gluetun VPN sidecar. Sonarr must connect to the **Gluetun service name** not `transmission` directly.

### Next session fix

In Sonarr → Settings → Download Clients → Transmission:

- Host: `gluetun` (or whatever the Gluetun service name is in the stack)
- Port: `9091`

---

## 📊 Current State (End of Session)

### Swarm Nodes

|Node|Role|Status|VLAN|
|---|---|---|---|
|`manager-01` (10.0.60.30)|Leader|Ready|60|
|`traefik-dmz-01` (10.0.60.40)|Worker|Ready|80+60|
|`worker-monitoring-01` (10.0.60.41)|Worker|Ready|60|
|`worker-media-01`|Worker|Ready|50+100|
|`worker-mediamanagement-01`|Worker|Ready|50+100|

### Stacks

|Stack|Status|Notes|
|---|---|---|
|`traefik`|✅ Running|Certs rate limited until ~Apr 26|
|`portainer`|✅ Running||
|`monitoring`|✅ Running|Grafana config pending|
|`plex`|✅ Running|Libraries intact|
|`arr`|⚠️ Partial|Sonarr not binding on 8989|

---

## ➡️ Next Session Priorities

1. **Fix Sonarr web server not binding** — investigate `AuthenticationMethod: Basic` vs `Forms` in v4, check full startup logs, check container-side `ss -tlnp`
2. **Fix Transmission connection** — point Sonarr/Radarr download client at Gluetun service name
3. **Cert rate limit clears ~Apr 26** — force Traefik stack update, confirm all certs issue across all domains, verify no `unrecognized name` TLS errors
4. **Plex external access** — once certs confirmed, test `https://plex.purvishome.com` end-to-end through Cloudflare → pfSense → Traefik → Plex
5. **Add `internal-only` middleware to InfluxDB** router in `stack-monitoring.yml`
6. **Final rsync** — after arr stack fully validated end-to-end
7. **Grafana data sources + dashboards** — still pending from previous session

---

## 🔗 Related Notes

- [[Docker Swarm Infrastructure Runbook]]
- [[Traefik Setup]]
- [[Session Notes — 2026-04-12 — Monitoring Deploy]]