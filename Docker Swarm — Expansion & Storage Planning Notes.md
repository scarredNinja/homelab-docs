---
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Proxmox
  - RaspberryPi
  - Storage
  - Expansion
---

# Docker Swarm — Expansion & Storage Planning Notes

> [!abstract] Overview Planning notes for expanding the Docker Swarm homelab beyond the initial single-Proxmox-node setup. Covers storage architecture decisions, backup gaps, Pi node failover design, and the target two-tier architecture.

---

## Current Setup (Stage 1 — Proxmox only)

- Single Proxmox host running all Swarm VMs
- ZFS datasets on Proxmox → virtiofs passthrough → mounted inside each VM at `/mnt/docker-data`, `/mnt/docker-tsdb`, `/mnt/docker-db`, `/mnt/docker-swarm`
- PBS not yet configured (mid-migration from old PVE node to new)
- Old PVE node still running Docker Compose stacks as source of truth
- Migration (Phase 2 ZFS send/receive) not yet run — data still on old node

### VM virtiofs mount reference

|Tag|Mount point|Purpose|
|---|---|---|
|`docker-data`|`/mnt/docker-data`|Docker data root|
|`docker-tsdb`|`/mnt/docker-tsdb`|Time-series DB storage|
|`docker-db`|`/mnt/docker-db`|Relational DB storage|
|`docker-swarm`|`/mnt/docker-swarm`|Shared configs, stack files|

---

## Key Decisions Before Migration

> [!warning] Do these before running Phase 2 ZFS send/receive — retrofitting after is more disruptive.

- [ ] Set up PBS pointed at new PVE node **before** migration so data is in scope from day one #Later 
- [ ] Add new PVE VMs to PBS backup jobs even before data lands on them #Later 
- [x] Decide: NFS export now, or virtiofs-only for now? #Later ✅ 2026-04-08
- [ ] Configure `zfs snapshot` cron job on new node for `docker-data`, `docker-tsdb`, `docker-db` #Later 
- [ ] Identify `zfs send` target (second pool, external disk, or NAS)  #Later 

---

## Critical: PBS Backup Gap

> [!danger] Verify this before relying on PBS as your only backup.

PBS backs up **VM disk images**. Docker data lives in ZFS datasets passed through via virtiofs — these are **host-side mounts, not inside the VM's virtual disk**. PBS may not be capturing your actual data.

```bash
# Verify where data actually lives
zfs list -r newpool/docker-data
zfs list -r newpool/docker-tsdb

# Check what PBS captured in last backup
proxmox-backup-client list
```

**Fix:** Add an independent ZFS snapshot + send job as a data-layer backup alongside PBS.

```bash
# Example — daily snapshot and send to second pool
SNAP="daily-$(date +%Y%m%d)"
for ds in docker-data docker-tsdb docker-db; do
  zfs snapshot newpool/${ds}@${SNAP}
  zfs send newpool/${ds}@${SNAP} | zfs receive backuppool/${ds}
done
```

---

## Storage Architecture — Scaling Assessment

### Why the current setup is node-local

virtiofs passthroughs are host-side — a second Proxmox node cannot see datasets on node 1. Raspberry Pis have zero visibility into ZFS datasets or virtiofs regardless.

### Three options for expansion

#### Option A — NFS export from Proxmox (recommended first step)

Export ZFS datasets as NFS. All VMs and Pis mount the share. Stateful containers read/write over NFS.

**Pros:** Simple, keeps PBS + ZFS snapshots as backup, works for Pis. **Cons:** Proxmox host is still a single point of failure for storage.

#### Option B — Local volumes + placement constraints

Keep node-local storage, use Swarm placement constraints to pin stateful services to the node where their data lives. Stateless services float freely.

**Pros:** No change to current setup, operationally simple. **Cons:** No HA for stateful services — failover requires manual data movement.

#### Option C — Distributed storage (GlusterFS / Longhorn)

Data replicated across nodes. Any container can run anywhere. True HA.

**Pros:** Full HA, maximum flexibility. **Cons:** High operational overhead, Pis too slow as storage nodes, non-trivial migration from ZFS.

### Recommended path

1. Export ZFS datasets as NFS before adding any new nodes
2. Add second Proxmox node — VMs mount NFS, Swarm works across both hosts
3. Add Pis — mount NFS for Proxmox-tier services; keep Pi-resident services on local Pi storage
4. Use placement constraints to keep SQLite services off Pi nodes

---

## Target Architecture — Two-Tier Design

> [!note] This is the long-term goal, not stage 1. Plex moves to a dedicated Proxmox VM. Pis run lightweight always-on services. Taking down Proxmox should leave core services up.

### Pi tier (always up)

|Service|Label|Storage|
|---|---|---|
|Traefik|`zone=pi-public`|Local Pi storage|
|Home Assistant|`zone=pi-controller`|Local Pi storage|
|Lightweight monitoring|`zone=pi-controller`|Local Pi storage|

- 3 Pi managers maintaining Swarm quorum
- Storage is local to each Pi — no dependency on Proxmox NFS
- All config: local bind mounts on Pi's own SSD/SD card

### Proxmox tier (up when Proxmox is up)

|Service|Label|Storage|
|---|---|---|
|Plex (dedicated VM)|`type=plex`|ZFS via virtiofs|
|Sonarr, Radarr, Transmission|`type=media`|ZFS via virtiofs|
|InfluxDB, Grafana, Prometheus|`role=manager`|ZFS via virtiofs|
|UniFi (if not on Pi)|`zone=controller`|ZFS via virtiofs|

- Plex stays hard-pinned — acceptable to be down when Proxmox is down
- PBS + ZFS snapshots handle backup for this tier

---

## Docker Swarm Manager Quorum — The Key Problem

> [!warning] If `manager-01` is a Proxmox VM and Proxmox goes down, the Swarm loses its manager. Services already running keep running, but nothing can be rescheduled or restarted if it crashes.

**Solution: 3 Swarm managers, at least 2 on Pis.**

```
manager-01        → Proxmox VM (current)
manager-pi-01     → Raspberry Pi
manager-pi-02     → Raspberry Pi
```

With 3 managers, taking down Proxmox leaves 2 Pi managers holding quorum. Traefik, HA, and Pi-pinned services keep running and can reschedule.

**Join Pis as managers from the start — don't add as workers and promote later.**

---

## Pi Node Failover Design

### How Swarm handles node failure

Swarm automatically reschedules services onto other nodes — but only if another node satisfies the placement constraints. If `zone=controller` only exists on one Pi and that Pi goes down, HA has nowhere to reschedule.

**Fix:** Share zone labels across multiple Pis so any eligible Pi can pick up the service.

### Pi label layout (3 nodes)

```
pi-01: role=pi-manager, zone=pi-public, zone=pi-controller
pi-02: role=pi-manager, zone=pi-public, zone=pi-controller
pi-03: role=pi-manager, zone=pi-public, zone=pi-controller
```

Services constrained to `zone=pi-controller` with `replicas: 1` — Swarm picks one node, reschedules to another if it fails.

### Placement constraint pattern — critical habit

```yaml
# Wrong — tied to one node forever
constraints:
  - node.hostname == pi-01

# Right — any Pi with this label can run it
constraints:
  - node.labels.zone == pi-controller
```

Apply this from day one. Adding nodes later is then just labelling them — no stack file changes needed.

### Storage options for Pi failover

|Option|How it works|Best for|
|---|---|---|
|Shared NFS from a dedicated storage Pi or NAS|All Pis mount the same share; failover picks up exact state|Databases, anything with active writes|
|Syncthing between Pis|Local copy on each Pi, kept in sync|Config files, Traefik static config, certs|
|Frequent rsync / snapshots|Local storage, synced periodically|Accept small state loss on failover|

**Recommended combination:**

- Syncthing for config files (Traefik config, HA config directory)
- Shared NFS or dedicated storage Pi for active databases (HA SQLite or external DB)

> [!tip] HA database option Move HA from SQLite to PostgreSQL/MariaDB on a reliable node. This separates HA's application state (Pi-local, survives failover) from its database (on a reliable host). Most people either accept some state loss on failover or do this.

---

## Singleton Service Reference (unchanged from runbook)

These services **must not run as multiple simultaneous replicas** — constrain to one node and treat failover as scheduled, not automatic.

|Service|Reason|
|---|---|
|Plex|GPU passthrough, local transcode cache|
|Sonarr / Radarr / Prowlarr / Tautulli|SQLite — two instances = corruption|
|Transmission|Active torrent state/resume files are local|
|Home Assistant|Single-instance state machine, IoT VLAN binding|
|UniFi Controller|MongoDB, device adoption tied to one controller|
|InfluxDB|Local TSDB, clustering is enterprise-only|
|Prometheus|Local TSDB|
|Traefik|Single ingress point|

---

## Stage-by-Stage Expansion Plan

### Stage 1 — Current (Proxmox only, single manager)

- [ ] Finish Phase 1 and Phase 2 migration per runbook  #Later 
- [ ] Set up PBS on new PVE node  #Later 
- [ ] Add ZFS snapshot cron job as independent data backup  #Later 
- [ ] Verify PBS is actually capturing Docker data (check virtiofs gap)  #Later 

### Stage 2 — Add second Proxmox node

- [ ] Export ZFS datasets as NFS from node 1 before adding node 2 #MuchLater
- [ ] Update VM mounts to use NFS (or keep virtiofs for existing VMs, NFS for new ones) #MuchLater
- [ ] Add node 2 to Proxmox cluster #MuchLater
- [ ] Join node 2 VMs to Swarm as workers #MuchLater
- [ ] Verify services can be rescheduled across both hosts #MuchLater

### Stage 3 — Add Raspberry Pis

- [ ] Join Pis to Swarm as managers from the start #MuchLater
- [ ] Apply shared zone labels (`zone=pi-controller`, `zone=pi-public`) #MuchLater
- [ ] Migrate Traefik to Pi tier with local Pi storage #MuchLater
- [ ] Migrate Home Assistant to Pi tier with local Pi storage #MuchLater
- [ ] Set up Syncthing for config sync between Pis #MuchLater
- [ ] Update stack files to use `zone=pi-*` constraints for Pi-resident services #MuchLater
- [ ] Verify Proxmox can be taken down with Pi tier remaining operational #MuchLater

---

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[Proxmox ZFS Dataset Layout]]
- [[Homelab Network — VLAN Reference]]