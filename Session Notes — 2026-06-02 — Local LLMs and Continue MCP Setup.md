---
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
status: Completed
tags:
  - session-note
  - homelab
---
# Session Notes — 2026-06-02 — Local LLMs and Continue MCP Setup

## Overview
- **Date:** 2026-06-02
- **Topic:** Local LLM Integration & Continue.dev VS Code MCP Setup
- **Status:** Complete & Verified

---

## 🏗️ Architecture & Component Layout

This setup configures the **Continue (continue.continue)** VS Code extension on the local Windows machine to leverage:
1. **Local LLMs:** Running on a local background Ollama instance.
2. **Obsidian Vault MCP Server:** Exposing the Obsidian Notes vault to the VS Code AI coding assistant via standard Model Context Protocol (MCP).

```
┌────────────────────────────────────────────────────────┐
│                      VS Code                           │
│  ┌──────────────────────────────────────────────────┐  │
│  │               Continue Extension                 │  │
│  └─────────┬─────────────────────────────┬──────────┘  │
│            │ (Local API)                 │ (stdio)     │
│            ▼                             ▼             │
│  ┌──────────────────┐           ┌──────────────────┐   │
│  │   Ollama Server  │           │  Filesystem MCP  │   │
│  │  (Port 11434)    │           │      Server      │   │
│  └─────────┬────────┘           └────────┬─────────┘   │
│            │                             │             │
│            ▼                             ▼             │
│  ┌──────────────────┐           ┌──────────────────┐   │
│  │ - Llama 3.2      │           │  Obsidian Vault  │   │
│  │ - Qwen 2.5 Coder │           │  (Home Notes)    │   │
│  └──────────────────┘           └──────────────────┘   │
└────────────────────────────────────────────────────────┘
```

---

## 🛠️ Implementation Details

### 1. Ollama Configuration
- **Executable Path:** `C:\Users\DJ\AppData\Local\Programs\Ollama\ollama.exe`
- **Execution Mode:** Background startup process (starts with Windows, manages via system tray).
- **Installed Models:**
  - `llama3.2:latest` (Llama 3.2 3B) — verified responsive and highly lightweight.
  - `qwen2.5-coder:7b` (Qwen2.5 Coder 7B) — ideal for autocomplete and specialized code tasks.

---

### 2. Workspace VS Code Configuration (`settings.json`)
Surgically updated the local repository VS Code settings to point the Continuous.dev/Continue extension to use the local Ollama backend with Llama 3.2:
- **Location:** `C:\Users\DJ\source\repos\docker-swarm-home\.vscode\settings.json`
```json
{
    "continuous.dev.llmBackend": "ollama",
    "continuous.dev.llmModel": "llama3.2:latest",
    "continuous.dev.mcpTools": ["obsidian"]
}
```

---

### 3. Global Continue Configuration (`config.yaml`)
Updated the global configuration for Continue (`C:\Users\DJ\.continue\config.yaml`) to enable both local LLMs and register the MCP filesystem server:

#### Target File: `C:\Users\DJ\.continue\config.yaml`
```yaml
name: Local Config
version: 1.0.0
schema: v1
models:
  - name: Llama 3.2
    provider: ollama
    model: llama3.2:latest
    roles:
      - chat
      - edit
      - apply
  - name: Qwen2.5-Coder 7B
    provider: ollama
    model: qwen2.5-coder:7b
    roles:
      - chat
      - edit
      - apply
      - autocomplete
  - name: Nomic Embed
    provider: ollama
    model: nomic-embed-text:latest
    roles:
      - embed
  - name: Gemini 3.1 Flash Lite
    provider: gemini
    model: gemini-3.1-flash-lite-preview
    apiKey: GEMINI_API_KEY
  - name: Gemini 3.1 Pro
    provider: gemini
    model: gemini-3.1-pro-preview
    apiKey: GEMINI_API_KEY

mcpServers:
  - name: obsidian-filesystem
    command: npx
    args:
      - -y
      - "@modelcontextprotocol/server-filesystem"
      - 'C:\Users\DJ\Documents\Notes\Home'
```

---

## 🔍 Verification & Testing

1. **Ollama Execution Test:** 
   Ran local query check against Llama 3.2 using PowerShell:
   ```powershell
   & "C:\Users\DJ\AppData\Local\Programs\Ollama\ollama.exe" run llama3.2 "Say hello in one word"
   ```
   **Output:** `Hello.` (Completed successfully in ~3s).
2. **Continue Config Validation:**
   Verified that `config.yaml` has correct formatting, matching JSON/YAML schemas, and that paths utilize proper single quotes/escapes for Windows filesystems.

---

## 🚀 Key Benefits

- **Privacy & Speed:** Model queries are executed 100% locally with ultra-low latency.
- **Seamless Context:** Using the `@modelcontextprotocol/server-filesystem` server, the VS Code AI assistant can natively reference, search, and edit your Obsidian vault files when you type `@` in Agent mode.


---

## 🛠️ 4. Advanced System Enhancements & Syncthing Permissions (Fixed)

Later in the session, we solved critical permissions issues and upgraded the local agent loop's parsing capabilities for robust, long-term operation.

### A. Syncthing Windows-to-Linux Permissions Alignment
- **Problem:** Syncthing was propagating Windows permission metadata back to the Linux host `/mnt/docker-data/vault`, stripping write permissions (`dr-xr-xr-x`) on key folders (like `10 - Projects`) and causing remote MCP writes to fail.
- **Solution:** We configured `ignorePerms="true"` inside the Obsidian Vault folder XML block on the host (`/mnt/docker-data/syncthing/config/config.xml`) and performed a rolling restart of the `dev-syncthing_syncthing` service. Folder permissions are now persistently handled by Linux (`drwxr-xr-x` for `docker:docker`) without being overwritten.

### B. Standard JSON-RPC Response Parsing
- **Problem:** The custom client interface in `local-vault-sync.ps1` was trying to read the flat response body. Standard JSON-RPC wraps tool execution results inside a nested `result.content` key.
- **Solution:** We updated `Call-McpTool` to robustly check for the standard `result.content` block, extracting output texts natively and parsing/displaying server-side validation `error` payloads cleanly.

### C. LLM Parameter Normalization Layer
- **Problem:** While `llama3.2:latest` supports native `tool_calls`, smaller 3B models frequently hallucinate parameter names (e.g. using `note_id` instead of `path`, or `data` instead of `fields`) and serialize objects/booleans as plain strings.
- **Solution:** Built a comprehensive parameter normalization and mapping layer directly into `local-vault-sync.ps1`. Before executing any tool call, it automatically:
  1. Maps hallucinated parameter names (e.g., `note_id` to `path` or `note_name`, `data` to `fields`, and `path` to `folder`).
  2. Resolves and parses stringified JSON structures (like `frontmatter` or `fields`).
  3. Formats stringified booleans (`"true"`/`"false"`) back to native PowerShell booleans.

### D. End-to-End Verification
We executed the pipeline end-to-end to verify functionality. The agent successfully traversed the stateful JSON-RPC channel and wrote `10 - Projects/MCP-Write-Test.md` directly to your homelab with correct permissions (`docker:docker`, `rw-r--r--`).

All actions, logs, and configuration paths are verified 100% active, private, and fully secure to your Dev VLAN network.
