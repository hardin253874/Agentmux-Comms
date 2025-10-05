###Start
pnpm agentmux tasks seed agentmux/dag.yaml


###Feature 1: 
Bootstrap workspace (pnpm) and folders; add package.json, pnpm-workspace.yaml, tsconfig, eslint/prettier; create packages/cli with an agentmux bin that wires commands: tasks seed|ls, worker, status.

###Feature 2: 
Implement tasks seed <dag.yaml> (reads YAML, creates state/tasks.db + state/events.jsonl, inserts tasks with deps).

###Then Run
pnpm agentmux tasks seed agentmux/dag.yaml
pnpm agentmux worker --agent codex --worker-id codex-1
