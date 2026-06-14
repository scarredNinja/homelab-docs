---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
# Archive Index

Files in this folder are superseded by current docs. Kept for historical reference only — do not follow instructions or configs from these files without verifying against current state.

---

## Provisioning Scripts (superseded)

Current script: `proxmox-swarm/scripts/02-provision-vm.sh` in [docker-swarm-home](https://github.com/scarredNinja/docker-swarm-home) repo.

| File | Version | Why archived |
|---|---|---|
| `(Archive) - provision-docker-swarm-vm.md` | v1 | Initial draft, virtiofs not working |
| `(Archive) - provision-docker-swarm-vm v5.md` | v5 | Pre-multi-NIC support |
| `(Archive) - provision-docker-swarm-vm v6.md` | v6 | Pre-multi-NIC support |
| `(Archive) - provision-docker-swarm-vm v7.md` | v7 | Pre-virtiofs fix |
| `(Archive) - provision-docker-swarm-vm v8.md` | v8 | Pre-post-boot split |
| `(Archive) - provision-docker-swarm-vm v11.md` | v11 | Superseded by v12 |
| `(Archive) - provision-docker-swarm-vm v11 1.md` | v11a | Duplicate/variant |
| `(Archived) - provision-docker-swarm-vm v12.md` | v12 | Superseded by v14 |
| `(Archived) - provision-docker-swarm-vm v14.md` | v14 | Moved to git repo; notes-based copy archived |

---

## VM Templates & Checklists (superseded)

Replaced by automated provisioning pipeline in git repo.

| File | Why archived |
|---|---|
| `(Archive) - Docker Swarm Worker Template.md` | Manual checklist; replaced by script |
| `(Archive) - 101 - Docker Swarm Worker Template.md` | VM-specific checklist; replaced by script |
| `(Archive) - 201 - Plex Worker Test.md` | Early test VM checklist |
| `(Archive) - v2 - Plex Worker Test.md` | Iteration on above |
| `(Archive) - VM Provisioning Manager Checklist - Swam Manager 01.md` | Manual checklist; replaced by script |
| `(Archived) - VM Provisioning Manager Checklist - swarm-manager-01.md` | Duplicate of above |
| `(Archive) - Swarm-manager-01 Checklist.md` | Per-VM manual checklist |
| `(Archive) - Swarm-manager-02 Checklist.md` | Per-VM manual checklist |
| `(Archive) - Swarm-manager-03 Checklist.md` | Per-VM manual checklist |
| `(Archive) - Test Worker Provisioning v1.md` | Superseded by pipeline |
| `Docker Swarm Provisioning Template.md` | Early template; replaced by git scripts |
| `Worker Docker Swarm Provisioning Template.md` | Worker-specific variant; replaced |
| `v2 - Docker Swarm Provisioning Template.md` | Version 2; superseded |
| `v3 - Docker Swarm Provisioning Template.md` | Version 3; superseded |
| `v3 - Docker Swarm Provisioning Template 1.md` | Duplicate/variant of v3 |
| `VM Automation Script - Swarm.md` | Early planning doc for automation |
| `Update Swarm Worker Provisioning Script.md` | Change log for old scripts |

---

## Shell Scripts (versioned snapshots)

| File | Why archived |
|---|---|
| `(Archive) - swarm-init.sh.md` | Inline swarm init; moved to pipeline |
| `(Archive) - join-manager.sh.md` | Manual join script; automated in pipeline |
| `(Archive) - join-worker.sh.md` | Manual join script; automated in pipeline |
| `(Archive) - consul.yaml.md` | Early consul experiment; abandoned in favour of overlay networking |

---

## Other

| File | Why archived |
|---|---|
| `(Archive) - Ha Reverse Proxy Process - not right now.md` | Home Assistant reverse proxy planning; deferred to Phase 9 |
| `(Archive) - Network Correction.md` | Network fix notes from early rebuild; resolved |
| `Homelab Rebuild Master Checklist - (OLD - Archive).md` | Pre-phase-hub master checklist; replaced by phase hub system |
| `Updates to be made.md` | Old scratchpad; tasks absorbed into phase hubs |
