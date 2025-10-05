# Repository Guidelines

## Project Structure & Module Organization
Agentmux Comms coordinates local agent collaboration. Keep the canonical design in `Agentmux-Comms-MCP-Bridge-Spec.md`; update diagrams and tables in place rather than duplicating them. Runtime code belongs under `.agentmux/`: store durable state in `.agentmux/state/` (SQLite DB, JSONL event log, heartbeats), share task outputs via `.agentmux/artifacts/`, and package MCP adapters inside `.agentmux/tools/`. The `agentmux` CLI should live in `.agentmux/packages/cli/` with `src/`, `tests/`, and `fixtures/` subdirectories. Add helper scripts under `scripts/` (PowerShell + POSIX sibling) and keep temporary data out of version control by default.

## Build, Test, and Development Commands
Run `npx prettier --check Agentmux-Comms-MCP-Bridge-Spec.md` before submitting spec edits. For CLI work, install dependencies with `pnpm install --filter agentmux` from `.agentmux/packages/cli`, then run `pnpm run build` to transpile and `pnpm run test` for the fast suite. Exercise the orchestration loop locally with `pnpm agentmux worker --agent codex --worker-id dev-1` and inspect the shared log via `pnpm agentmux status`. Document any new command in the spec and this guide.

## Coding Style & Naming Conventions
Default to TypeScript for the CLI and Rust or TypeScript for MCP servers; keep indentation at two spaces for TS and four for Rust. Name tasks with the `domain:slug` pattern (`impl:T-001`). Event types must remain uppercase snake case. Use Prettier and ESLint defaults; add `.editorconfig` entries when introducing new languages.

## Testing Guidelines
Write unit tests for DB access, event serialization, and MCP handlers. Map each public command to at least one integration test that seeds tasks, runs a worker, and inspects `.agentmux/state/status.md`. Name test files `<module>.spec.ts` (CLI) or `<module>_test.rs` (Rust). Target >90% branch coverage on critical orchestration modules and include concurrency race tests before merging.

## Commit & Pull Request Guidelines
Commit messages follow Conventional Commits (`feat:`, `fix:`, `docs:`) and describe the user-facing change in the imperative mood. Squash noisy WIP commits locally. Pull requests need: concise summary, linked issue or TODO, test evidence (`pnpm run test` output), and screenshots or log excerpts for tooling changes. Flag breaking schema updates clearly and call out required migrations for `.agentmux/state`.
