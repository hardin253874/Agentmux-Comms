# Agentmux Comms & MCP Bridge

## 1) Overview
Agentmux Comms orchestrates multiple local AI agents against a single workspace using a SQLite task store, an append-only JSONL event log, and an artifacts directory for durable inputs/outputs. Two optional MCP servers - `mcp-state` and `mcp-artifacts` - provide a stable API boundary so tools interact with task state and artifacts without direct file or DB access. Use this stack when you need deterministic multi-agent runs with transparent artifacts and a reproducible local workflow.

## 2) Features
- Atomic task claiming with leases and dependency checks
- Append-only event log paired with a human-friendly status page
- Dedicated workers per agent (e.g., claude, codex, gemini)
- Safe controls: `worker --stop`, `ps`, `reset`, `unlock`
- SQLite configured with WAL mode and a `busy_timeout`
- Idempotent DAG seeding via UPSERT logic
- Optional MCP servers for tool integrations

## 3) Prerequisites
- Node.js 20+, pnpm 9+, and Git
- Windows PowerShell 7+ or macOS/Linux Bash
- (Optional) Python 3.12+ to compile native deps such as `better-sqlite3`
- Enable Corepack and pnpm
  - PowerShell
    ```powershell
    corepack enable
    corepack prepare pnpm@latest --activate
    ```
  - Bash
    ```bash
    corepack enable && corepack prepare pnpm@latest --activate
    ```

## 4) Installation & Build
Clone, install dependencies, and build the CLI package.
- PowerShell
  ```powershell
  git clone <REPO_URL> agentmux-comms
  cd agentmux-comms
  pnpm install
  pnpm --filter @agentmux/cli run build
  ```
- Bash
  ```bash
  git clone <REPO_URL> agentmux-comms
  cd agentmux-comms
  pnpm install
  pnpm --filter @agentmux/cli run build
  ```

## 5) Quick Start (Single Repo)
Expected workspace structure:
```
.
  agentmux/
    dag.yaml
  state/           # created automatically
  artifacts/       # created automatically
  README.md
```
Seed the DAG and launch the first worker.
- PowerShell
  ```powershell
  pnpm agentmux tasks seed agentmux/dag.yaml
  pnpm agentmux worker --agent claude --worker-id claude-1
  ```
- Bash
  ```bash
  pnpm agentmux tasks seed agentmux/dag.yaml
  pnpm agentmux worker --agent claude --worker-id claude-1
  ```
Add more agents (run each in its own terminal):
```powershell
pnpm agentmux worker --agent codex  --worker-id codex-1
pnpm agentmux worker --agent gemini --worker-id gemini-1
```
```bash
pnpm agentmux worker --agent codex  --worker-id codex-1
pnpm agentmux worker --agent gemini --worker-id gemini-1
```

## 6) DAG Example
A minimal `agentmux/dag.yaml` with a smoke test:
```yaml
tasks:
  - id: "smoke:noop-codex"
    name: "Smoke test (codex)"
    agent: "codex"
    deps: []
    payload: { kind: "noop" }

  - id: "init:workspace"
    name: "Bootstrap workspace"
    agent: "claude"
    deps: []
    payload: { kind: "noop" }

  - id: "docs:status"
    name: "Render status page"
    agent: "codex"
    deps: ["init:workspace"]
    payload: { kind: "noop" }
```
Extend this file with additional tasks, dependencies, and payloads as needed for your workflow.

## 7) Status & Observability
Inspect task state, status summaries, and raw events.
- PowerShell
  ```powershell
  pnpm agentmux tasks ls
  pnpm agentmux status
  type .\state\status.md
  type .\state\events.jsonl
  ```
- Bash
  ```bash
  pnpm agentmux tasks ls
  pnpm agentmux status
  cat ./state/status.md
  tail -n 50 ./state/events.jsonl
  ```
Each worker writes heartbeats to `state/heartbeats/<agent>.json`; a heartbeat is stale if its timestamp exceeds roughly two renewal intervals (typically >120 seconds by default).

## 8) Safe Controls (IMPORTANT)
Use the CLI controls instead of killing `node.exe`.
- PowerShell
  ```powershell
  pnpm agentmux ps
  pnpm agentmux worker --stop --agent claude --worker-id claude-1
  pnpm agentmux reset --dag agentmux/dag.yaml
  pnpm agentmux unlock <taskId>
  ```
- Bash
  ```bash
  pnpm agentmux ps
  pnpm agentmux worker --stop --agent claude --worker-id claude-1
  pnpm agentmux reset --dag agentmux/dag.yaml
  pnpm agentmux unlock <taskId>
  ```
These commands read PID files from `state/pids/`, request graceful stops via `.stop` control files, drain workers before reseeding, and clear leases without manual SQL. They avoid indiscriminately terminating processes, preserving your CLI session and preventing Codex from killing itself.

## 9) Multi-Project Integration
Two common patterns:
- **Standalone orchestrator per project** - keep this repository next to the target codebase, maintain a project-specific `agentmux/dag.yaml`, and use per-project `writeAllow`/`execAllow` rules in `agentmux.yaml` to confine workers.
- **Shared global orchestrator** - publish `@agentmux/cli` to a private registry or use workspace linking; projects include their own DAG files and invoke the CLI via `pnpm agentmux ...` or `npx agentmux ...`.
Payloads can point to other repositories via fields such as `payload.repoPath`, while workers should only read/write within sanctioned artifact subdirectories. Do not grant broad filesystem access; rely on MCP artifact whitelists and command allowlists.

## 10) MCP Servers (Optional)
- `mcp-state` exposes operations like `list_ready_tasks`, `claim_task`, `renew_lease`, `complete_task`, `fail_task`, `release_task`, `seed_from_dag`, `get_task`, and `append_event`.
- `mcp-artifacts` handles artifact reads/writes, directory listings, and JSON helpers under the `artifacts/` whitelist.
Run them (if scripts are provided):
```powershell
pnpm mcp:state
pnpm mcp:artifacts
```
MCP servers let LLM agents interact through stable APIs rather than touching the DB or filesystem directly.

## 11) Configuration Notes
- SQLite pragmas enabled: `journal_mode=WAL`, `busy_timeout=5000`, `foreign_keys=ON`
- All timestamps use UTC ISO-8601
- Event log is append-only JSONL
- DAG seeding is idempotent (tasks UPSERT by id)
- Worker PID files live in `state/pids/`, stop files in `state/control/`

## 12) Troubleshooting
- **Worker idles** - dependencies probably aren't `DONE`; inspect `pnpm agentmux tasks ls` and recent events.
- **Database locked** - use `pnpm agentmux ps` and `pnpm agentmux worker --stop ...` to halt workers, then `pnpm agentmux reset --dag agentmux/dag.yaml`.
- **Codex killed itself** - follow the safe controls above; don't terminate every `node.exe`.
- **Reseed confusion** - reseeding is safe and idempotent; prefer `reset` followed by `tasks seed` instead of deleting the DB.
- **Path errors** - verify `agentmux/dag.yaml` exists and paths in payloads/commands are correct.

## 13) Security & Ethics
Keep secrets out of artifacts and events. Use explicit command and path allowlists for workers. Provide only the minimum filesystem access required by each agent and ensure artifact directories remain whitelisted by the MCP artifact server.

## 14) FAQ
- **Can this run without MCP servers?** Yes; workers can operate directly on the local state files.
- **Can I add custom executors (e.g., shell, mcp-call)?** Yes. Implement payload `kind` handlers in the worker runtime.
- **How do I run this on CI?** Seed the DAG, launch one or more workers (often headless), and collect the status artifacts (events, status.md, artifacts directory).
