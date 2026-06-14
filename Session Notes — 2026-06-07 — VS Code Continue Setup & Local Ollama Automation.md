---
tags:
  - session-note
  - homelab
  - automation
project_id: Homelab-2025
date: '2026-06-07'
---
# 🤖 Local LLM & Editor Automation Review (2026-06-07)

During this session, we migrated the homelab development workspace configurations and set up local offline-first LLM tooling on the new Linux host.

## 🛠️ Work Done

### 1. Repository Relocation
- Moved `docker-swarm-home` from `.gemini` scratch directory to its permanent repository folder at `/home/scarredninja/source/repos/docker-swarm-home/`.

### 2. Local Ollama GPU Ingress
- Installed Ollama on the local Linux host.
- Verified NVIDIA GPU acceleration configuration.
- Pulled the `qwen2.5-coder:7b` model to act as a highly capable, offline tool-calling developer agent.

### 3. Editor Configuration (VS Code + Continue)
- Resolved Continue.dev loading conflicts on Linux.
- Disabled `config.ts` and `config.yaml` overrides by renaming them to `.bak` and consolidated configurations into `~/.continue/config.json`.
- Configured Continue.dev with:
  - **Local Model**: `qwen2.5-coder:7b` running via Ollama.
  - **Obsidian MCP Server**: Remote SSE connection to `https://obsidian-mcp.home.purvishome.com/mcp` using secure headers for tool-use in the vault.

---

## 🎯 Future Automations & Enhancements to Review

To streamline your overall workload, here are the target automations to establish next:

### A. Local RAG & Workspace Indexing
- Enable Continue's indexing (by editing `.continuerc.json` or config settings) to let your local Qwen model search and read your local code files and project documentations dynamically.

### B. FitNotes Ingestion Pipeline (NewtonFit)
- Complete the Syncthing mirror from your Android phone to the Docker Swarm persistent volume storage.
- Set up the server-side watch directory with `chokidar` to parse and merge workout CSV logs automatically on incoming transfers.

### C. Daily Biometric Logs (Tasker -> Swarm)
- Build the Tasker task on Android to pull Samsung Health daily metrics (Weight/RHR) and POST them directly to the Traefik ingress endpoint `https://newtonfit.home.purvishome.com/api/data` when on home Wi-Fi.

### D. Obsidian Vault Bidirectional sync
- Test and deploy `sync-obsidian-goals.ps1` as a cron job to automatically update `Physical Goals & Biomarkers.md` frontmatter daily with real-time lift maximums and biometric averages.

---

## 💻 Dev Environment Automations, Agents & Git Hooks

Implement the following developer workspace workflows to optimize local workflow speed and deployment stability:

### 1. Git Hooks & Repository Automations
- **Pre-commit Linter & Build Validation**: Set up `pre-commit` hooks in `docker-swarm-home` and development repositories to execute lint checks, format files, and run local test suites (e.g. prisma validation, dotNET build checks) before commits are finalized.
- **Post-merge Dependency Updates**: A post-merge git hook to automatically trigger `npm install` or `dotnet restore` when pulling updates down to the host.
- **Git Push Deployment Webhooks**: Configure a local hook or git host webhook that triggers your Docker Swarm Traefik endpoint to pull the latest image and run a rolling update (`docker service update --image <image> <service>`) upon push to `main`.

### 2. Editor Rules & Custom Agent Instructions
- **Workspace-specific Rules (`.continue/rules` or `CLAUDE.md`)**: Create a ruleset containing the styling directives (e.g., vanilla CSS Nord theme, glassmorphic layout) and database migration commands (like database purge sequences) to guide coding assistants during generation.
- **Automated Changelogs & Roadmap Scripts**: Schedule local scripts (like `scripts/prepare_changelog.py` and `scripts/prepare_roadmap.py`) to automatically parse commit histories and draft Obsidian project changelogs.

### 3. Custom MCP Server Agents (Local Extensions)
- **Filesystem & Database MCPs**: Connect your IDE's agent to local workspace directories or read-only SQL databases so it can query schema definitions, database structures, or active configurations directly during development.
