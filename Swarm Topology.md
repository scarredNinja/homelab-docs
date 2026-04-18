---

project_id: Homelab-2025 
phase: "Phase 5: Docker Swarm" 
tags:

- DockerSwarm
- Topology
- Dashboard date: 2026-04-05

---

# Swarm Topology — Living Dashboard

> [!note] This file is the current state snapshot. Updated at the end of every session. Reference docs: [[Docker Swarm Infrastructure Runbook]] · [[Traefik Routing Architecture]] · [[ZFS Configuration and Setup]]

---

## 🚨 Active Blockers

|#|Blocker|Detail|
|---|---|---|
|1|**Traefik not yet deployed**|Stack and config ready. Deploy via Portainer next session.|
|2|**Worker token handling in 02-provision-vm.sh**|traefik-dmz-01 joined as manager instead of worker — stale token suspected. Verify script reads fresh token correctly before provisioning remaining workers.|

---

## Next Actions (in order)

- [ ] Deploy Traefik stack via Portainer UI
- [ ] Verify both entrypoints active (`:443` websecure, `:8443` internal)
- [ ] Check Cloudflare DNS challenge completes and cert issues
- [ ] Verify Portainer accessible via Traefik internal entrypoint
- [ ] Check `02-provision-vm.sh` worker token handling — fix if stale token issue confirmed
- [ ] Provision `worker-media-01`, `worker-controller-01`, `worker-general-01`
- [ ] Deploy monitoring stack (Grafana, Prometheus, InfluxDB)
- [ ] ZFS snapshot cron — verify backup pool target once old pool repurposed

---

## VM Status

|VM|VMID|VLAN(s)|IP|Status|Notes|
|---|---|---|---|---|---|
|`manager-01`|200|60|10.0.60.30|✅ Running|Swarm Leader. Portainer deployed.|
|`traefik-dmz-01`|201|60 + 80|TBC|✅ Running|Joined Swarm as worker. Traefik pending.|
|`worker-media-01`|—|50|—|Not provisioned|—|
|`worker-controller-01`|—|60 + 20|—|Not provisioned|—|
|`worker-general-01`|—|40|—|Not provisioned|—|

---

## Service Status

|Service|VM|Status|Notes|
|---|---|---|---|
|Portainer|manager-01|✅ Running|`https://manager-01:9443` direct access|
|portainer_agent|all nodes|✅ Running|global mode — on all current nodes|
|Traefik v3|traefik-dmz-01|🔜 Next session|Stack ready, deploy via Portainer|
|Grafana|manager-01|⏳ Pending|After Traefik|
|Prometheus|manager-01|⏳ Pending|After Traefik|
|InfluxDB|manager-01|⏳ Pending|After Traefik|
|Plex|worker-media-01|⏳ Pending|Waiting on worker|
|Sonarr / Radarr|worker-media-01|⏳ Pending|—|
|Transmission|worker-media-01|⏳ Pending|—|
|Home Assistant|worker-controller-01|⏳ Pending|—|
|UniFi|worker-controller-01|⏳ Pending|—|

---

## Phase 0 Status

|Step|Status|
|---|---|
|0.0 Template prep (NBD)|✅ Done|
|0.1 Verify PBS|⏳ Later — after PBS setup|
|0.2 ZFS tuning + snapshot cron|✅ Done|
|0.3 PBS backup jobs|⏳ Later|
|0.4 NFS exports|✅ Done|
|0.5 Placement constraint discipline|✅ Done|

---

## Infrastructure Reference

**Overlay network:** `traefik-public` — all services attach to this **Stack files:** `/mnt/docker-swarm/stacks/<service>/stack.yml` **Runtime config:** `/mnt/docker-data/<service>/data/` **Repo:** `/root/proxmox-swarm` (SSH via port 443) **Swarm worker token:** `/root/swarm_worker_token.txt` — update after any manager reprovision

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