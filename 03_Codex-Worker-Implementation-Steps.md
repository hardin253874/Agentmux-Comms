# Codex CLI Worker Implementation & Testing Guide

This document describes the six next steps to continue building the **Agentmux Comms & MCP Bridge** system using **Codex CLI**.

---

## 1) Implement a Minimal Working Worker Loop (No-Op Executor)

**Goal:** Make `pnpm agentmux worker --agent codex --worker-id codex-1` run continuously and process tasks.

### Feature prompt for Codex

```
Feature: Implement `agentmux worker` (packages/cli) with a robust loop:
- Poll claimables for `--agent <name>`, atomically claim one with a 10-min lease.
- Append `TASK_CLAIMED` → set `RUNNING` → append `TASK_STARTED`.
- Temporary executor: if `payload.kind === "noop"` or no kind provided, sleep 2s and mark done.
- Append `TASK_COMPLETED` with a JSON result; set state `DONE`.
- Renew lease every 60s; write heartbeat JSON (state/heartbeats/<agent>.json).
- If no tasks ready, idle and sleep 10s; don’t exit.
- Graceful shutdown (Ctrl+C): release current task (TASK_RELEASED), set READY, exit 0.
```

### SQL snippet for atomic claim

```sql
UPDATE tasks
SET state='CLAIMED', claimed_by=@worker, lease_until=@leaseUntil, updated_at=@now
WHERE id=@id
  AND state='READY'
  AND (
    SELECT COUNT(*) FROM json_each(tasks.deps) d
    JOIN tasks t2 ON t2.id = d.value
    WHERE t2.state != 'DONE'
  ) = 0;
```

### Expected result
Running `pnpm agentmux worker --agent codex --worker-id codex-1` should stay active, heartbeat every ~60s, and mark any noop tasks as DONE.

---

## 2) Add a Smoke-Test Noop Task

Add this to the top of `agentmux/dag.yaml`:

```yaml
- id: "smoke:noop-codex"
  name: "Smoke test noop (codex)"
  agent: "codex"
  deps: []
  payload:
    kind: "noop"
```

Then reseed and run:

```bash
pnpm agentmux tasks seed agentmux/dag.yaml
pnpm agentmux worker --agent codex --worker-id codex-1
```

**Expected:** `TASK_CLAIMED`, `TASK_STARTED`, `TASK_COMPLETED` events appear and the task moves to `DONE`.

---

## 3) Add Idle, Lease Renewal, and Heartbeat Polish

Feature prompt for Codex:

```
Feature: Worker polish:
- Renew lease every 60s (if RUNNING)
- Write state/heartbeats/<agent>.json with {ts, agent, currentTaskId, leaseUntil}
- Exponential backoff while idle (10s → 20s → 30s, cap 30s)
- On crash/error, append TASK_FAILED and retry or BLOCKED depending on error
```

---

## 4) Implement `agentmux status` Command

Feature prompt for Codex:

```
Feature: Implement `agentmux status`:
- Fold state/events.jsonl + tasks.db → render state/status.md
- Show counts by agent/state, blocked tasks, last 10 events, heartbeat staleness
```

Run after completion:

```bash
pnpm agentmux status
```

---

## 5) Verify Everything Works

Commands to verify basic functionality:

```bash
# List tasks
pnpm agentmux tasks ls --agent codex

# Show current status
pnpm agentmux status
type .\state\status.md

# Confirm heartbeat file exists
type .\state\heartbeats\codex.json
```

**Expected outputs:**
- Status file updates with counts and last events.
- Heartbeat JSON updates every minute.
- Noop task marked DONE.

---

## 6) Extend to Real Executors

Once the loop is stable, extend Codex worker with modular executors:

| Kind | Behavior |
|------|-----------|
| `noop` | Sleep 2s, mark DONE |
| `shell` | Run allowed command in payload (`cmd`), capture exit code & logs |
| `mcp-call` | Call a local MCP tool, record response as artifact |

Guards:
- Whitelisted commands only.
- Timeout per run.
- Logs written to `artifacts/<taskId>/<runId>/...`.

---

**End of Document**
