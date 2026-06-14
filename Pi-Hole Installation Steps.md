---
project_id: Homelab-2025
phase: 'Phase 3: Network Config'
tags:
  - pihole
status: Reference
---

## Tasks

- [x] - Review Pi hole setup with two pihole instances on raspberry pi. [priority:: 1] ✅ 2026-02-17
	- Status: Currently not HA, only on one 
 - [x] format both pis ✅ 2026-02-18
	 - [x] pi 1 ✅ 2026-02-18
	 - [x] p2 ✅ 2026-02-18
- [x] Pi 1 - Set ip to VLAN 60 ✅ 2026-02-24
	- Update the ports VLAN 
- [x] Configure Gravity Sync between pihole1 and pihole2 ✅ 2026-05-22
- [ ] Traefik — add routing for pihole1 (10.0.60.20) and pihole2 (10.0.60.21)
- [x] Deploy node_exporter on pihole1 and pihole2 for Prometheus scraping ✅ 2026-05-28
- [x] Deploy pihole-exporter (eko/pihole-exporter) in monitoring stack for DNS/ad-block metrics ✅ 2026-05-28

**Core Components:**

- 2x Raspberry Pis (`pihole1` at `10.0.60.20`, `pihole2` at `10.0.60.21`) — both on VLAN 60
- Pi-hole on each node
- Gravity Sync (automatic Pi-hole database synchronization over SSH)
- DNS load-balancing via pfSense DHCP (both IPs listed as DNS servers per VLAN)

---

## Step-by-Step Setup

### 1 — Prepare the Pis

On both Raspberry Pis:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git unzip -y
hostnamectl set-hostname pihole1  # pihole2 on the other
```

---

### 2 — Install Pi-hole on Each

```bash
curl -sSL https://install.pi-hole.net | bash
```

When prompted:

- Interface: `eth0`
- Static IPs:
  - pihole1 → `10.0.60.20`
  - pihole2 → `10.0.60.21`
- Upstream DNS: `1.1.1.1`

Confirm after install:

```bash
pihole -v
```

Web interfaces: `http://10.0.60.20/admin` and `http://10.0.60.21/admin`

---

## Gravity Sync Setup

Gravity Sync 4.x replicates the Pi-hole SQLite databases (blocklists, allow/deny lists, custom DNS records, client groups) from pihole1 (primary) to pihole2 (secondary) over SSH.

### Step 1 — SSH Key Setup (run on pihole2)

```bash
ssh-keygen -t ed25519 -C "gravity-sync" -f ~/.ssh/gravity_sync -N ""
ssh-copy-id -i ~/.ssh/gravity_sync.pub admin@10.0.60.20
# Verify:
ssh -i ~/.ssh/gravity_sync admin@10.0.60.20 "echo SSH OK"
```

### Step 2 — Install Gravity Sync (run on BOTH pihole1 and pihole2)

Clone the repo and copy the actual binary (`gravity-sync.sh` in the repo root is the v3→v4 migration utility — do not use it):

```bash
git clone https://github.com/vmstan/gravity-sync
sudo cp ~/gravity-sync/gravity-sync /usr/local/bin/gravity-sync
sudo chmod +x /usr/local/bin/gravity-sync
sudo mkdir -p /etc/gravity-sync/.gs/templates
sudo cp ~/gravity-sync/templates/* /etc/gravity-sync/.gs/templates/
```

### Step 3 — Configure Gravity Sync (run on pihole2 only)

```bash
gravity-sync config
```

This runs an interactive setup wizard that:
- Registers the SSH key to pihole1 (10.0.60.20)
- Detects the local and remote Pi-hole installations automatically
- Writes `/etc/gravity-sync/gravity-sync.conf`

No config step needed on pihole1 — it is the primary.

### Step 4 — Run Initial Push and Verify (run on pihole2)

```bash
gravity-sync push   # pushes pihole1's full config down to pihole2
gravity-sync logs   # confirm success
```

Then verify pihole2 web UI (`http://10.0.60.21/admin`) matches pihole1.

### Step 5 — Enable Auto-Sync (run on pihole2)

```bash
gravity-sync auto   # sets up systemd timer for ongoing sync
```

---

## Verification

1. Make a change on pihole1 (add a blocklist or custom DNS entry)
2. Wait for the cron interval, or run `gravity-sync pull` manually on pihole2
3. Confirm the change appears in pihole2's web UI (`http://10.0.60.21/admin`)
4. Run `gravity-sync logs` — look for `Sync completed successfully`
5. Confirm pfSense DHCP for each VLAN lists both `10.0.60.20` and `10.0.60.21` as DNS servers

---

## Notes

- No Nebula needed — both Pis are on VLAN 60 (`10.0.60.x`) and have direct L2 reachability
- pihole1 is always primary; do not make config changes on pihole2 directly — they will be overwritten on next sync
- Gravity Sync is one-way (pihole1 → pihole2); use `gravity-sync push` from pihole2 to force an immediate sync
- `gravity-sync.sh` in the repo root is the v3→v4 migration utility — do not copy it; the actual v4 binary is the `gravity-sync` file (no extension)
- Templates must be copied to `/etc/gravity-sync/.gs/templates/` before running `gravity-sync config`
- Auto-sync uses systemd timers, not crontab

## Local DNS Records

| Record | Value | Purpose |
|--------|-------|---------|
| `*.home.purvishome.com` | `10.0.60.40` | Traefik LAN IP — routes all homelab subdomains through Traefik |

- Added via Pi-hole UI → Local DNS → DNS Records on 2026-05-26
- If Pi-hole is rebuilt or Gravity Sync resets, this record must be re-added manually on pihole1 (Gravity Sync will then replicate it to pihole2)

---

## Monitoring Setup

Two layers of monitoring are in place:

### 1 — System-Level Metrics: `node_exporter`

Installed natively on both Pis via `apt` so Prometheus can scrape CPU, memory, disk, and network stats.

```bash
# Run on BOTH pihole1 and pihole2:
sudo apt update && sudo apt install -y prometheus-node-exporter
sudo systemctl enable --now prometheus-node-exporter

# Verify:
sudo systemctl status prometheus-node-exporter
curl -s http://localhost:9100/metrics | head -20
```

Default port: `9100`. Both hosts are on VLAN 60, same as the Swarm monitoring VM — no pfSense inter-VLAN rule needed.

Prometheus scrape targets (in `config/monitoring/prometheus.yml`):

```yaml
- job_name: node_exporter_external
  static_configs:
    - targets:
        - '10.0.60.20:9100'   # pihole1
        - '10.0.60.21:9100'   # pihole2
      labels:
        vlan: 'vlan60'
```

### 2 — Pi-hole DNS Metrics: `pihole-exporter`

Deployed as a Swarm service in `stack-monitoring.yml` using `ekofr/pihole-exporter:v0.4.0`.
Exposes Pi-hole-specific metrics (blocked queries, top clients, query types) on port `9617`.

**Pre-requisite — create Docker Secret (run once on a Swarm manager):**

```bash
# Use your Pi-hole web admin password
printf 'your-pihole-password' | docker secret create pihole_password -
```

The service is pinned to `zone=monitoring` and monitors both instances using comma-separated hostnames:

```yaml
PIHOLE_HOSTNAME: "10.0.60.20,10.0.60.21"
```

Prometheus scrape job:

```yaml
- job_name: pihole-exporter
  static_configs:
    - targets: ['tasks.monitoring_pihole-exporter:9617']
```

**Verification:**

```bash
# From a Swarm node
curl -s http://<monitoring-vm-ip>:9617/metrics | grep pihole_
```

Grafana dashboard: import ID **10176** (Pi-hole Exporter dashboard from Grafana Labs).
