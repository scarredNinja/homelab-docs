---
date: 2026-03-27
tags: [Homelab, Phase5, DockerSwarm, Provisioning, Scripting]
project_id: Homelab-2025 
phase: "Phase 5: Docker Swarm"
---

# Session Notes — virtiofs Fix + Multi-NIC + Provision Flow

## Issue Fixed
02-provision-vm.sh was failing with virtiofs attach errors:
  virtiofs0: invalid format - format error
  virtiofs0.dirid: invalid configuration ID '/mnt/docker-data'

Root cause: script was passing path/tag directly to qm set.
Proxmox virtiofs requires Datacenter Directory Mappings — qm set takes mapping ID only.

Correct syntax: qm set $VMID --virtiofs0 "docker-data"

## Changes Made This Session
- 02: Fixed virtiofs attach to use mapping IDs
- 02: Added preflight check for all 4 directory mappings
- 02: Added multi-NIC support (--nic1-vlan, --nic2-vlan, --nic3-vlan)
- 02: Added pfSense DHCP prompt before VM start
- 02: Added prompt to chain into 03-post-boot.sh automatically
- 03: Fixed ZFS zvol format/mount (uses UUID in fstab, /mnt/local/<name>)
- 03: Added virtiofs fstab entries (idempotent)
- 01: Added check_or_create_mapping() to guarantee mappings exist

## Storage Decision — Plex Transcoding
Plex transcode temp should be a ZFS zvol (raw block device), not virtiofs.
virtiofs has FUSE overhead unsuitable for high-I/O transcode scratch.
Deploy pattern: --volume transcode:50G → /mnt/local/transcode inside VM

## ZFS Zvol Backup Gap
ZFS zvols created by 02 are NOT captured by PBS.
Must be added to ZFS snapshot cron manually after provisioning.
Template cron command printed in 02 summary output.

## Multi-NIC Design
--nic1-vlan: primary (management VLAN, cloud-init runs here) — never DMZ/IoT
--nic2-vlan: secondary (DMZ=80 for external services, IoT=20 for HA etc.)
--nic3-vlan: tertiary (reserved for storage VLAN future use)
Guard added: warns + prompts if nic1 VLAN is 80