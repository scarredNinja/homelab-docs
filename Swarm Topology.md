---

project_id: Homelab-2025 
phase: "Phase 5: Docker Swarm" 
tags:

- DockerSwarm
- Topology

---

# Swarm Topology — Living Dashboard

> [!note] This file is the current state snapshot. Updated at the end of every session. Reference docs: [[Docker Swarm Infrastructure Runbook]] · [[Traefik Routing Architecture]] · [[ZFS Configuration and Setup]]

---

## 🚨 Active Blockers

|#|Blocker|Detail|
|---|---|---|
|2|**Verify UniFi Prometheus metrics**|✅ Resolved 2026-05-08 — data confirmed in Grafana dashboards 11311/11315/11312.|

---

## Next Actions (in order)

- [x] Verify HA and UniFi containers healthy on `worker-controller-01` ✅ 2026-04-28
- [x] Wire Sonarr/Radarr → Transmission download client ✅ 2026-04-30
- [x] Update Prowlarr → Sonarr/Radarr sync config ✅ 2026-05-01
- [x] Deploy restic off-site backup → Synology ✅ 2026-05-07 PR #17
- [x] Deploy VM backups (vzdump) ✅ 2026-05-07 PR #17
- [x] Grafana backup dashboard ✅ 2026-05-07 PR #18
- [x] Deploy Uptime Kuma via Portainer ✅ 2026-05-08
- [x] Fix Uptime Kuma monitors — Prometheus, Grafana, Transmission ✅ 2026-05-08 (PR `claude/exciting-brahmagupta-e8f897`)
- [x] Fix InfluxDB 502 Bad Gateway — added to `traefik-public` network ✅ 2026-05-08 (same PR)
- [x] Merge PR `claude/exciting-brahmagupta-e8f897` — uptime-kuma arr_arr_default + InfluxDB fix ✅ 2026-05-08
- [x] Fix arr stack — Sonarr and Radarr back up ✅ 2026-05-08
- [x] Investigate arr stack performance — resolved, VM performance confirmed stable ✅ 2026-05-08
- [x] Add Uptime Kuma monitors for arr services ✅ 2026-05-08
- [x] Investigate Tautulli — DNS entry issue resolved ✅ 2026-05-08
- [x] Verify all 4 backup jobs in Grafana after overnight cron run — check tomorrow [priority:: 1] ✅ 2026-05-12
- [x] look at error in restic job [prioritry::1]
- [x] Verify `zfs-snapshot.sh` duplicate log fix — check `/var/log/zfs-snapshot.log` after 02:15 cron run for single entries [priority:: 1] ✅ 2026-05-11
- [x] Verify `restic-backup.sh` — check log for `RESTIC:` prefixed stderr lines after current run completes [priority:: 1] ✅ 2026-05-11
- [x] Add pfSense rule: VLAN 60 → `10.0.90.50:9100` (Prometheus → Proxmox node_exporter) ✅ 2026-05-08
- [x] Tailscale routing resolved — deployed and running ✅ 2026-05-08
- [x] Fix UniFi 404 — resolved ✅ 2026-05-08
- [x] Verify UniFi Prometheus metrics in Grafana ✅ 2026-05-08
- [x] Allow Home VLAN (VLAN 10) → Home Assistant access — pfSense rule VLAN10 → `10.0.60.42:8123`; `internal-only` already had `10.0.10.0/24` ✅ 2026-05-08
- [x] Redback HACS integration — checked 2026-05-08; integration working on v0.3 (`juicejuice/homeassistant_redback`); `ActiveExportedPowerInstantaneouskW` KeyError is a non-breaking log warning, no fix available ✅ 2026-05-08
- [x] Disable Home Assistant analytics telemetry ✅ 2026-05-08
- [x] Homepage dashboard build — 🟡 In progress; branch `claude/elegant-lehmann-00bda3`, container healthy, Traefik routing untested, no PR yet. See `project_homepage_state.md` [priority:: 2] ✅ 2026-05-24
- [x] Update Grafana data sources to Swarm service DNS names ✅ 2026-05-08 — Prometheus + InfluxDB updated; Loki connected, no dashboards yet
- [x] Update pfSense VLAN 50 VPN policy routing alias — superseded; Gluetun handles NordVPN tunnel at container level, pfSense policy routing not required ✅ 2026-05-08
- [x] Permanent Traefik asymmetric routing fix (`use-routes: false` on `traefik-dmz-01`) ✅ 2026-04-29 — resolved by NIC binding + netplan
- [x] Correct Prometheus scrape targets for exportarr (.50.30 -> .51) and verify successful scrape ✅ 2026-05-20
- [x] Update stacks to deploy directly from Git repo — all Swarm stacks migrated to Portainer Git integration (scarredNinja/docker-swarm-home, main branch) ✅ 2026-05-08

---

## 🖥️ VM Status

```dataviewjs
const pages = dv.pages('"10 - Projects"')
  .where(p => p.type === "swarm-vm")
  .sort(p => p.vmid ?? 999, "asc");

const statusIcon = s => ({
  "not-created":       "⬜",
  "provisioned":       "🟡",
  "post-boot-complete":"🟠",
  "running":           "🟢",
}[s] ?? "❓");

dv.table(
  ["VM", "VMID", "VLAN(s)", "IP", "Status", "Post-Boot", "Swarm"],
  pages.map(p => [
    p.file.link,
    p.vmid ?? "—",
    [p.vlan_primary, p.vlan_secondary].filter(Boolean).join(" + "),
    p.ip_primary ?? "—",
    statusIcon(p.vm_status) + " " + (p.vm_status ?? "—"),
    p.post_boot_run ? "✅" : "🔲",
    p.swarm_joined  ? "✅" : "🔲",
  ])
);
```

---

## 🐳 Service Status

```dataviewjs
const pages = dv.pages('"10 - Projects"')
  .where(p => p.type === "swarm-service")
  .sort(p => p.service_name, "asc");

const statusIcon = s => ({
  "pending":   "⬜",
  "deploying": "🟡",
  "deployed":  "🟢",
  "running":   "🟢",
  "degraded":  "🔴",
}[s] ?? "❓");

dv.table(
  ["Service", "VM", "Status", "Port", "External", "Dataset"],
  pages.map(p => [
    p.file.link,
    p.vm ?? "—",
    statusIcon(p.service_status) + " " + (p.service_status ?? "—"),
    p.port ?? "—",
    p.external_access ? "🌐 Yes" : "🔒 No",
    p.zfs_dataset ?? "—",
  ])
);
```

---

## Stack Deployment Status

| Stack       | File                                                   | Status                                                                                                                   | VM                        |
| ----------- | ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ | ------------------------- |
| portainer   | `proxmox-swarm/stacks/stack-portainer.yml`             | ✅ Deployed — Git integration not applicable (self-managed)                                                               | manager-01                |
| traefik     | `proxmox-swarm/stacks/stack-traefik.yml`               | ✅ Deployed — Portainer Git ✅ 2026-05-08                                                                                  | traefik-dmz-01            |
| monitoring  | `proxmox-swarm/stacks/stack-monitoring.yml`            | ✅ Deployed — Portainer Git ✅ 2026-05-08; Alertmanager + Discord alerting added PR #48 ✅ 2026-05-20                       | worker-monitoring-01      |
| plex        | `proxmox-swarm/stacks/stack-plex.yml`                  | ✅ Deployed — Portainer Git ✅ 2026-05-08; transcode mount + cloudflared token fixed PR #22                                | worker-media-01           |
| arr         | `proxmox-swarm/stacks/stack-arr.yml`                   | ✅ Deployed — Portainer Git ✅ 2026-05-08; Exportarr sidecars updated with prefix-free env vars & _v2 secrets ✅ 2026-05-20 | worker-mediamanagement-01 |
| compose-vpn | `proxmox-swarm/stacks/compose-vpn.yml`                 | ✅ Running (Docker Compose, not Swarm) — `compose-vpn.service` systemd unit (WireGuard/NordLynx) ✅ 2026-05-17             | worker-mediamanagement-01 |
| controller  | `proxmox-swarm/stacks/controller/stack-controller.yml` | ✅ Deployed — Portainer Git ✅ 2026-05-08; unpoller switched to virtiofs bind mount PR #19                                 | worker-controller-01      |
| restic      | `stacks/stack-restic.yml`                              | ✅ Deployed — Portainer Git ✅ 2026-05-08; backup script fixes PR #19                                                      | worker-monitoring-01      |
| uptime-kuma | `proxmox-swarm/stacks/stack-uptime-kuma.yml`           | ✅ Deployed — Portainer Git ✅ 2026-05-08                                                                                  | worker-monitoring-01      |
| homepage    | `proxmox-swarm/stacks/stack-homepage.yml`              | ✅ Deployed — Manually                                                                                                    | worker-monitoring-01      |

---

## Phase 0 Status

|Step|Status|
|---|---|
|0.0 Template prep (NBD)|✅ Done|
|0.1 Verify PBS|✅ Decided against 2026-05-07 — ZFS send + vzdump + restic sufficient|
|0.2 ZFS tuning + snapshot cron|✅ Done|
|0.3 PBS backup jobs|✅ Not required — see Step 0.1|
|0.4 NFS exports|✅ Done|
|0.5 Placement constraint discipline|✅ Done|
|0.6 VM backups (vzdump)|✅ Done 2026-05-07 — PR #17|
|0.7 Restic off-site → Synology|✅ Done 2026-05-07 — PR #17 (rest-server over NFS)|

---

## Infrastructure Reference

**Overlay networks:**
- `traefik-public` (`10.200.2.0/24`) — all Swarm services that need Traefik routing attach here
- `arr_arr_default` (`10.200.5.0/24`) — **attachable** overlay; compose containers on `worker-mediamanagement-01` join this to reach Traefik and each other

**Stack files:** `/mnt/docker-swarm/stacks/<stack>/stack-<name>.yml`
**Compose stacks (non-Swarm):** `/mnt/docker-swarm/stacks/arr/compose-vpn.yml` — run with `docker compose` on `worker-mediamanagement-01`
**Runtime config:** `/mnt/docker-data/<service>/` (virtiofs)
**Repo:** `/root/proxmox-swarm` on Proxmox host (SSH via port 443)
**Swarm worker token:** `/root/swarm_worker_token.txt` — update after any manager reprovision

**Inter-VM SSH:** `manager-01` `~/.ssh/id_ed25519` configured — passwordless SSH to all workers ✅ 2026-04-25
**VPN:** Transmission + Gluetun run as Docker Compose on `worker-mediamanagement-01`. NordVPN WireGuard (NordLynx) (successfully migrated on 2026-05-17 to eliminate single-threaded CPU scaling limits and UI latency). Traefik file-provider route `vpn-transmission.yml` → `transmission.home.purvishome.com`.
**Traefik cert resolver:** Cloudflare DNS-01 challenge active. `acme.json` chmod 600 ✅ 2026-04-25.

> [!tip] Overlay ping test — use nsenter, not host ping
> `docker network inspect` only shows local containers for non-attachable overlays.
> To test overlay reachability, enter the network namespace:
> ```bash
> # Get netns ID from `docker network ls`
> sudo nsenter --net=/var/run/docker/netns/1-<short-netid> ping <overlay-ip>
> ```

### Provisioning Commands

```bash
unset CI_USER

# manager-01
./02-provision-vm.sh --name manager-01 --vmid-min 200 \
  --role manager --first-manager --vlan 60 --memory 4096 --cores 2 --disk-size 20G

# traefik-dmz-01
./02-provision-vm.sh --name traefik-dmz-01 --vmid-min 201 \
  --vlan 60 --nic2-vlan 80 --memory 1024 --cores 2 --disk-size 10G

# worker-media-01
./02-provision-vm.sh --name worker-media-01 --vmid-min 202 \
  --vlan 50 --memory 8192 --cores 4 --disk-size 20G

# worker-controller-01
./02-provision-vm.sh --name worker-controller-01 --vmid-min 203 \
  --vlan 60 --nic2-vlan 20 --memory 4096 --cores 2 --disk-size 20G

# worker-general-01
./02-provision-vm.sh --name worker-general-01 --vmid-min 204 \
  --vlan 40 --memory 4096 --cores 2 --disk-size 20G
```

---

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Traefik Routing Architecture]]
- [[ZFS Configuration and Setup]]
- [[pfSense Firewall Rules]]
- [[VLAN and Subnet Summary Sheet]]