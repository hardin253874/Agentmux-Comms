# Agentmux Comms & MCP Bridge — 13‑Step Technical Specification

> Scope: This spec defines the **Agentmux Comms & MCP Bridge** only (local orchestration + MCP servers). It excludes any app/frontend/crawling projects.

---

## 1) System Overview
**Goal:** Coordinate multiple AI agents (e.g., Gemini CLI, Codex CLI, Claude CLI) via a shared local workspace.  
**Mechanics:** Agents collaborate through:
- **Append‑only event log** (`state/events.jsonl`)
- **SQLite task store with leases/dependencies** (`state/tasks.db`)
- **Artifacts directory** (`artifacts/…`) for inputs/outputs
- **Two MCP servers** exposing state and artifact APIs

**Directory layout**
```
.agentmux/
  state/
    tasks.db
    events.jsonl
    heartbeats/
      gemini.json
      codex.json
      claude.json
  artifacts/
  tools/
    mcp-state/
    mcp-artifacts/
  packages/
    cli/
```

---

## 2) Data Model (SQLite + JSONL)
### Tasks table (SQLite)
```sql
CREATE TABLE IF NOT EXISTS tasks (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  agent TEXT NOT NULL,
  state TEXT NOT NULL CHECK (state IN ('READY','CLAIMED','RUNNING','BLOCKED','DONE','FAILED')) DEFAULT 'READY',
  deps TEXT NOT NULL DEFAULT '[]',      -- JSON array of task ids
  payload TEXT NOT NULL DEFAULT '{}',   -- JSON input
  result TEXT,                          -- JSON output
  claimed_by TEXT,
  lease_until TEXT,                     -- ISO8601
  retries INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_tasks_state_agent ON tasks(state, agent);
CREATE INDEX IF NOT EXISTS idx_tasks_lease ON tasks(lease_until);
```
**Claimable rule:** `state='READY'` **and** all `deps` are `DONE`.

### Event log (append‑only JSON Lines)
Examples:
```json
{"ts":"2025-10-05T03:15:02Z","agent":"gemini","type":"TASK_STARTED","taskId":"spec:write","runId":"r-001"}
{"ts":"2025-10-05T03:21:44Z","agent":"gemini","type":"TASK_COMPLETED","taskId":"spec:write","output":{"spec":"artifacts/spec.md"}}
{"ts":"2025-10-05T03:30:10Z","agent":"claude","type":"TASK_FAILED","taskId":"impl:T-001","error":"test failure","retries":1}
{"ts":"2025-10-05T03:31:10Z","agent":"codex","type":"HEARTBEAT","taskId":null,"note":"idle"}
```
Event types: `TASK_CREATED`, `TASK_READY`, `TASK_CLAIMED`, `TASK_STARTED`, `TASK_PROGRESS`, `TASK_COMPLETED`, `TASK_FAILED`, `TASK_RELEASED`, `HEARTBEAT`.

### Heartbeats
Each agent updates `state/heartbeats/<agent>.json` with `{ts, agent, currentTaskId, leaseUntil}`.

---

## 3) Workflow Lifecycle
States: `READY → CLAIMED → RUNNING → (DONE | FAILED | BLOCKED | RELEASED→READY)`

**Worker protocol**
1. Poll claimables (READY + deps DONE). Atomically **CLAIM** with a lease (e.g., 10m).
2. Append `TASK_CLAIMED` → set `RUNNING` → append `TASK_STARTED`.
3. Execute logic (LLM/shell). Periodically append `TASK_PROGRESS`. Renew lease.
4. On success: write outputs → `TASK_COMPLETED` → set `DONE`.
5. On retryable failure: append `TASK_FAILED`, bump `retries`, return to `READY` (or `BLOCKED` if human input required).
6. Crash recovery: if lease expires, task becomes claimable again.

**Atomic claim (pseudo‑SQL)**
```sql
UPDATE tasks
SET state='CLAIMED', claimed_by=:worker, lease_until=:now_plus_10m, updated_at=:now
WHERE id=:id
  AND state='READY'
  AND (SELECT COUNT(*) FROM json_each(tasks.deps) d
       JOIN tasks t2 ON t2.id = d.value
       WHERE t2.state != 'DONE') = 0;
```

---

## 4) MCP Server: `mcp-state`
**Purpose:** Safe API for task state & events (no direct DB/JSONL touching).

**Methods**
- `list_ready_tasks({ agent? }) -> TaskSummary[]`
- `claim_task({ id, worker, leaseSeconds? }) -> ClaimedTask`
- `renew_lease({ id, worker, leaseSeconds? }) -> { leaseUntil }`
- `start_task({ id, worker }) -> { ok: true }`
- `append_event({ event }) -> { ok: true }`
- `complete_task({ id, worker, result? }) -> { ok: true }`
- `fail_task({ id, worker, reason, retryable? }) -> { ok: true }`
- `release_task({ id, worker }) -> { ok: true }`
- `seed_from_dag({ path }) -> { created }`
- `get_task({ id }) -> Task`

**Errors**
`TASK_NOT_FOUND`, `TASK_NOT_READY`, `LEASE_CONFLICT`, `NOT_CLAIMED_BY_WORKER`, `VALIDATION_ERROR`, `IO_ERROR`.

**Types (abbrev)**
```json
TaskSummary: { "id":"string","name":"string","state":"string","deps":"string[]" }
ClaimedTask: { "task": Task, "leaseUntil":"string" }
Event: { "ts":"string","agent":"string","type":"string","taskId":"string|null", ... }
```

---

## 5) MCP Server: `mcp-artifacts`
**Purpose:** Controlled read/write of artifacts under `artifacts/` only.

**Methods**
- `put_artifact({ path, contentBase64 }) -> { ok: true }`
- `get_artifact({ path }) -> { contentBase64, size }`
- `list_artifacts({ dir, pattern? }) -> { paths }`
- `write_json({ path, data }) -> { ok: true }`
- `read_json({ path }) -> { data }`

**Guards**
- Reject paths outside `artifacts/`.
- JSON helpers enforce UTF‑8, pretty formatting.

---

## 6) CLI Tool (`agentmux`)
**Commands**
- `agentmux tasks seed path/to/dag.yaml` — seed tasks & deps
- `agentmux tasks ls [--state ...] [--agent ...]`
- `agentmux worker --agent <name> --worker-id <id>` — claim/execute loop
- `agentmux status` — summarize into `state/status.md`

**DAG example**
```yaml
tasks:
  - id: "spec:write"
    name: "Write specification"
    agent: "gemini"
    deps: []
    payload: { sourceDir: "artifacts/input" }

  - id: "plan:ticketize"
    name: "Generate tickets"
    agent: "codex"
    deps: ["spec:write"]
    payload: { specPath: "artifacts/spec.md" }

  - id: "impl:T-001"
    name: "Implement feature step 1"
    agent: "claude"
    deps: ["plan:ticketize"]
    payload: { ticketId: "T-001" }
```

---

## 7) Concurrency & Safety
- **Atomic claims** with leases; renew every ~60s.
- **Crash recovery** via lease expiry.
- **Idempotent outputs:** `artifacts/<taskId>/<runId>/…`.
- **WIP limits** per agent (enforced in worker).
- **Write fences:** MCP restricts writes to `artifacts/` only.
- **Append‑only events**; never rewrite history.

---

## 8) Status Page
`state/status.md` snapshot with:
- Counts by state & agent (Ready/Running/Done/Failed)
- Blocked tasks (with reasons)
- Last N events (humanized)
- Heartbeat freshness indicators

Example:
```
# Agentmux Status (UTC)

| Agent  | Ready | Running | Done | Failed |
|--------|-------|---------|------|--------|
| Gemini |   0   |    1    |  2   |   0    |
| Codex  |   1   |    0    |  1   |   0    |
| Claude |   0   |    1    |  0   |   1    |
```

---

## 9) Testing Strategy
- **Unit:** DB ops (claim/renew/complete), event append, path guards.
- **Integration:** Two workers race for same task → exactly one wins.
- **Crash Recovery:** Kill a worker; ensure lease expiry enables reclaim.
- **E2E:** Seed DAG → run workers → verify all tasks to `DONE`.
- **Security:** Attempt out‑of‑bounds writes → rejected; verify logs sanitized.

---

## 10) Error Handling
| Condition            | Code                      |
|----------------------|---------------------------|
| Bad input            | `VALIDATION_ERROR`        |
| Lease conflict       | `LEASE_CONFLICT`          |
| Missing task         | `TASK_NOT_FOUND`          |
| Wrong worker/lease   | `NOT_CLAIMED_BY_WORKER`   |
| IO/FS/DB failure     | `IO_ERROR`                |

MCP returns `{ ok:false, code, message }` on error.

---

## 11) Security & Operations
- MCP binds to `127.0.0.1` only.
- Restrictive FS permissions for `state/` and `artifacts/`.
- Logs: `state/mcp-state.log`, `state/mcp-artifacts.log` with rotation.
- UTC timestamps everywhere.
- Avoid sensitive content in logs (only ids/paths/summaries).

---

## 12) Roadmap
1. Richer task queries & filters
2. Pluggable DB (SQLite → Postgres) behind `mcp-state`
3. Optional web dashboard (reads JSONL + SQLite)
4. Remote workers via authenticated TCP MCP bridge
5. Metrics: throughput, failure rate, cycle time

---

## 13) Acceptance & Roles
### Acceptance Criteria
- Tasks can be seeded, claimed, completed **atomically**.
- Events are append‑only and well‑formed.
- MCP servers enforce boundaries and path whitelists.
- Leases renew and expire correctly; recovery works.
- CLI renders accurate `status.md`.
- Concurrency tests pass; no double‑claims.

### Ownership Roles
| Role       | Responsibility                                    |
|------------|----------------------------------------------------|
| **Codex**  | Author spec/contracts, design schemas, diff review |
| **Claude** | Implement MCP servers + CLI, ensure tests pass     |
| **Gemini** | Optional scheduler/reporting (polls & status)      |

---

**End of 13‑Step Specification**
