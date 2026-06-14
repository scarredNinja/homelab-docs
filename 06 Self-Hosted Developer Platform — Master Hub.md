---
project_id: DevPlatform-2026
status: In Progress
created_date: '2026-06-02'
tags:
  - project
  - dev-platform
  - homelab
phase: Developer Platform
---
# 🛠️ Self-Hosted Developer Platform — Master Hub

> [!NOTE]
> This master project hub tracks the deployment, configuration, and security audits of self-hosted software development, container management, and continuous integration services on the Swarm cluster.

## Project Details
- **Project ID:** `DevPlatform-2026`
- **Status:** 🔵 Planned
- **VLAN Focus:** VLAN 40 (Developer Isolation) & VLAN 60 (Management Ingress)

---

## 🎯 Scope & Objectives
1. Provide a local continuous delivery ecosystem bypassable from external public clouds.
2. Establish a secure Git Repository host utilizing custom database storage.
3. Deploy an internal container registry to host custom microservice images.
4. Support browser-based remote workspace IDE setups.

---

## 📋 Deferred Project Tasklist

### 🔹 Stage 1: Git Service & Continuous Integration
- [/] **Forgejo/Gitea Setup:** Deploy Forgejo Swarm service with persistent ZFS storage and external Postgres hook (Stack deployed manually).
- [/] **Woodpecker CI:** Deploy Woodpecker manager and Swarm workers to automate formatting and linting tasks (Agent VM provisioned and joined to Swarm).

### 🔹 Stage 2: Container Registry & Artifact Storage
- [ ] **Harbor Registry:** Deconstruct Harbor into Swarm-compliant ingress paths, using Redis caching and Postgres storage.

### 🔹 Stage 3: Developer Ingress & Automation
- [ ] **code-server:** Deploy remote browser-based VS Code server sandboxed behind Traefik Authentication.
- [ ] **n8n Automation:** Deploy n8n workflow engine to hook Git commit activities to discord alerts.

- [ ] look at some other MCP tool sets and some further automation for obsidian vault where possible. 
- [ ] build out some skills and rule books for obsidian MCP and continue/vscode

---

## 🔗 Related Resources
- [[Docker Swarm Details]]
- [[VM - dev-node-01]]
- [[VM - dev-runner-01]]
- [[Homelab Next Phase — Review & Roadmap]]
