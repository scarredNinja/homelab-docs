---
project_id: Homelab-2025
status: Reference
phase: 'Phase 5: Docker Swarm'
tags:
  - reference
  - architecture
---
# NetBox Migration and Compose‑VPN Blind Spot Review Plan

## Goal Description

Research and produce detailed migration and remediation reports for two critical infrastructure tasks in the **docker‑swarm‑home** repository:

1. **NetBox Migration** – Move the stateful PostgreSQL and Redis components of the NetBox stack from `manager‑01` to a dedicated worker node, preserving ZFS storage datasets and updating placement constraints.
2. **Compose‑VPN Blind Spot** – Evaluate the current `systemd`‑managed `compose‑vpn` service on `worker‑mediamanagement‑01`, and propose an architecture that ensures automatic restoration on node reprovision (e.g., Swarm service, GitOps boot‑strap, Portainer hook).

Both reports will be stored as artifacts (`netbox_migration_review.md` and `vpn_blindspot_review.md`) and linked from this note.

## User Review Required

> [!IMPORTANT]
> No immediate decisions needed – the plan assumes read‑only access to the repository and vault. Let me know if you prefer a different format.

## Open Questions

> [!WARNING]
> *None at this time.*

## Proposed Research Steps

### NetBox Migration
- Locate NetBox stack definition files (e.g., `stack‑netbox.yml`) in the repository.
- Identify current placement constraints for PostgreSQL and Redis services.
- Verify ZFS dataset paths for persistent storage.
- Review existing Obsidian documentation (`Service - NetBox.md`, `VM - manager‑01.md`).
- Draft migration steps:
  * Provision a worker node with appropriate labels.
  * Snapshot/replicate ZFS datasets.
  * Update stack file with new constraints and volume options.
  * Perform zero‑downtime data migration (pg_dump/restore, Redis replication).
  * Validate services and adjust firewall/VLAN rules.

### Compose‑VPN Blind Spot
- Open `systemd/compose‑vpn.service` and `compose‑vpn.yml`.
- Determine why `systemd` is required (NET_ADMIN, capabilities).
- Explore alternatives:
  * Convert to a Docker Swarm service with needed capabilities.
  * Automate `systemd` unit deployment via `03‑post‑boot.sh`.
  * Use Portainer compose hooks or webhook for auto‑restore.
- Evaluate security and reliability trade‑offs.
- Produce a concise remediation plan.

## Verification Plan
- Use `grep_search` to confirm presence of relevant files.
- Open files via `view_file` to extract snippets.
- Write the two markdown artifacts (`netbox_migration_review.md`, `vpn_blindspot_review.md`).

---
### Linked Tasks
- [[Homelab Next Phase — Review & Roadmap#NetBox on manager node]]
- [[Homelab Next Phase — Review & Roadmap#compose‑vpn is a Swarm blind spot]]

*Note: Adjust the task note titles if they differ in your vault.*
