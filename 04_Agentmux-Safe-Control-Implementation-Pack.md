# Agentmux Safe Control — Implementation Pack (Code-Ready)

This pack contains **code-ready snippets** and exact **feature prompts for Codex** to finish the
safe-control improvements: PID/stop handling, `worker --stop`, `reset`, `ps`, `unlock`, SQLite
resilience, and idempotent seeding.

---

## 1) Worker PID & Stop-File Logic (TypeScript)

**Files:** `packages/cli/src/worker/index.ts` (or shared utils)

```ts
import fs from "node:fs";
import path from "node:path";

const STATE_DIR = path.resolve(process.cwd(), "state");
const PID_DIR = path.join(STATE_DIR, "pids");
const CTRL_DIR = path.join(STATE_DIR, "control");

function ensureDirs() {
  fs.mkdirSync(PID_DIR, { recursive: true });
  fs.mkdirSync(CTRL_DIR, { recursive: true });
}

export function writePid(agent: string, workerId: string) {
  ensureDirs();
  const pidFile = path.join(PID_DIR, `${agent}-${workerId}.pid`);
  fs.writeFileSync(pidFile, String(process.pid), "utf8");
  // best-effort cleanup on exit
  const cleanup = () => { try { fs.unlinkSync(pidFile); } catch {} };
  process.on("exit", cleanup);
  process.on("SIGINT", () => { cleanup(); process.exit(0); });
  process.on("SIGTERM", () => { cleanup(); process.exit(0); });
}

export function shouldStop(agent: string, workerId: string) {
  const stopFile = path.join(CTRL_DIR, `${agent}-${workerId}.stop`);
  return fs.existsSync(stopFile);
}

export function clearStop(agent: string, workerId: string) {
  const stopFile = path.join(CTRL_DIR, `${agent}-${workerId}.stop`);
  try { fs.unlinkSync(stopFile); } catch {}
}
```

**Use in worker loop (pseudocode):**
```ts
writePid(agent, workerId);

while (true) {
  if (shouldStop(agent, workerId)) {
    // release current task if any (append TASK_RELEASED), write final heartbeat
    await releaseCurrentTaskIfAny(db, workerId, agent);
    clearStop(agent, workerId);
    break; // graceful exit
  }
  // ... existing claim/renew/execute/heartbeat logic ...
}
```

---

## 2) CLI: `worker --stop`, `ps`

**File:** `packages/cli/src/commands/worker.ts` and a new `ps.ts`

```ts
// worker --stop
import fs from "node:fs";
import path from "node:path";

export async function stopWorker(agent: string, workerId: string) {
  const stopFile = path.join("state", "control", `${agent}-${workerId}.stop`);
  fs.mkdirSync(path.dirname(stopFile), { recursive: true });
  fs.writeFileSync(stopFile, "");
  console.log(`Signaled stop for ${agent}:${workerId}.`);
}
```

```ts
// ps (list known PIDs)
import fs from "node:fs";
import path from "node:path";

export function listWorkers() {
  const dir = path.join("state", "pids");
  if (!fs.existsSync(dir)) return [];
  const entries = fs.readdirSync(dir).filter(f => f.endsWith(".pid"));
  return entries.map(f => {
    const pid = fs.readFileSync(path.join(dir, f), "utf8").trim();
    const [agent, rest] = f.replace(".pid","").split("-");
    const workerId = rest;
    return { agent, workerId, pid };
  });
}
```

**Wire into CLI router (example):**
```ts
// packages/cli/src/index.ts (or commander setup)
if (cmd === "worker" && argv.includes("--stop")) {
  const agent = getArg("--agent");
  const workerId = getArg("--worker-id");
  await stopWorker(agent, workerId);
  process.exit(0);
}
if (cmd === "ps") {
  console.table(listWorkers());
  process.exit(0);
}
```

---

## 3) CLI: `reset` (graceful drain + reseed)

**File:** `packages/cli/src/commands/reset.ts`

```ts
import fs from "node:fs";
import path from "node:path";
import { listWorkers } from "./ps";

async function waitForExit(pid: number, timeoutMs = 15000) {
  const start = Date.now();
  while (Date.now() - start < timeoutMs) {
    try { process.kill(pid, 0); } catch { return true; } // no such process -> exited
    await new Promise(r => setTimeout(r, 500));
  }
  return false;
}

export async function reset(opts: { dagPath?: string, force?: boolean }) {
  // 1) graceful stop
  const procs = listWorkers();
  for (const p of procs) {
    const stopPath = path.join("state","control",`${p.agent}-${p.workerId}.stop`);
    fs.mkdirSync(path.dirname(stopPath), { recursive: true });
    fs.writeFileSync(stopPath, "");
  }
  // 2) wait
  let allExited = true;
  for (const p of procs) {
    const exited = await waitForExit(Number(p.pid));
    if (!exited) allExited = false;
  }
  if (!allExited && !opts.force) {
    console.error("Some workers are still running. Use --force to proceed anyway.");
    process.exit(1);
  }
  // 3) rotate events
  const ev = path.join("state","events.jsonl");
  if (fs.existsSync(ev)) {
    const stamp = new Date().toISOString().replace(/[:.]/g,"-");
    fs.renameSync(ev, path.join("state",`events.${stamp}.jsonl`));
  }
  // 4) reseed (do not delete DB if you support upsert)
  if (opts.dagPath) {
    await seedFromDag(opts.dagPath); // call your existing seeding code
  }
  console.log("Reset completed.");
}
```

**Wire into CLI:**
```ts
if (cmd === "reset") {
  const force = argv.includes("--force");
  const dagPath = getArg("--dag") || "agentmux/dag.yaml";
  await reset({ dagPath, force });
  process.exit(0);
}
```

---

## 4) SQLite Resilience (WAL + busy_timeout)

**File:** shared DB init (e.g., `packages/cli/src/db.ts`)

```ts
import Database from "better-sqlite3";

export function openDb(path = "state/tasks.db") {
  const db = new Database(path);
  db.pragma("journal_mode = WAL");
  db.pragma("busy_timeout = 5000");
  db.pragma("foreign_keys = ON");
  return db;
}
```

---

## 5) Idempotent Seeding (UPSERT)

**File:** seeding code (e.g., `packages/cli/src/commands/seed.ts`)

```ts
const stmt = db.prepare(`
INSERT INTO tasks (id, name, agent, state, deps, payload, created_at, updated_at)
VALUES (@id, @name, @agent, 'READY', @deps, @payload, @now, @now)
ON CONFLICT(id) DO UPDATE SET
  name=excluded.name,
  agent=excluded.agent,
  deps=excluded.deps,
  payload=excluded.payload,
  updated_at=excluded.updated_at
`);
// Loop over tasks from DAG, with deps as JSON string
```

**Optional:** When removing tasks from the DAG, mark old tasks as `BLOCKED` instead of deleting.

---

## 6) Safe PowerShell Killers (targeted)

**Filter by agentmux worker in command line:**
```powershell
Get-CimInstance Win32_Process `
| Where-Object {
    $_.Name -ieq 'node.exe' -and
    $_.CommandLine -match 'agentmux worker'
} `
| ForEach-Object { Stop-Process -Id $_.ProcessId -Force }
```

**Filter by project path too:**
```powershell
$repo = (Get-Location).Path.Replace('\','\\')
Get-CimInstance Win32_Process `
| Where-Object {
    $_.Name -ieq 'node.exe' -and
    $_.CommandLine -match $repo -and
    $_.CommandLine -match 'agentmux worker'
} `
| ForEach-Object { Stop-Process -Id $_.ProcessId -Force }
```

**Filter by environment tag (if you set `process.env.AGENTMUX="1"`):**
```powershell
Get-CimInstance Win32_Process `
| Where-Object {
    $_.Name -ieq 'node.exe' -and $_.CommandLine -match 'AGENTMUX=1'
} `
| ForEach-Object { Stop-Process -Id $_.ProcessId -Force }
```

---

## 7) Feature Prompts for Codex (copy/paste)

- **PID/Stop logic:**
  > Feature: Add PID file write and .stop control file support to worker loop; on stop, release current task (TASK_RELEASED), write final heartbeat, cleanup pid file, and exit 0.

- **CLI worker --stop:**
  > Feature: Implement `agentmux worker --stop --agent <a> --worker-id <id>` which creates `state/control/<agent>-<id>.stop` and prints confirmation.

- **CLI ps:**
  > Feature: Implement `agentmux ps` that lists entries from `state/pids/*.pid` as a table with agent, workerId, pid.

- **CLI reset (graceful drain):**
  > Feature: Implement `agentmux reset [--force] [--dag <path>]` that signals stop to all known workers, waits up to 15s, rotates events.jsonl, and reseeds DAG without deleting tasks.db.

- **SQLite WAL + busy_timeout:**
  > Feature: Set SQLite pragmas (journal_mode=WAL, busy_timeout=5000, foreign_keys=ON) in the DB initializer used by CLI and workers.

- **Seeding upsert:**
  > Feature: Change seeding to UPSERT tasks by id rather than dropping DB; update name/agent/deps/payload on conflict and touch updated_at.

---

## 8) Acceptance Checklist

- `agentmux ps` shows live worker PIDs
- `agentmux worker --stop --agent claude --worker-id claude-1` stops only that worker
- `agentmux reset --dag agentmux/dag.yaml` drains workers, rotates events, reseeds
- Seeding twice doesn’t duplicate tasks; removed tasks become BLOCKED or ignored
- WAL + busy_timeout prevent “database is locked” under normal concurrency
- No Codex self-termination while stopping Agentmux workers

---

**End — Agentmux Safe Control Implementation Pack**
