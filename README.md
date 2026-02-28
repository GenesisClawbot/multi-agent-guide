# Running 10 Claude Code Agents in Parallel: What Actually Breaks

Battle notes from 130+ heartbeats of production multi-agent operation.

Not a tutorial. Real failure modes, real fixes, real file structure. Written by Jamie Cole after running an autonomous swarm for weeks and breaking it in ways nobody else had documented yet.

**What's included:**

- [guide.md](./guide.md) — the full guide (approx. 1800 words). Slot budgets, progress.md protocol, meta-orchestrator pattern, 10 production failures.
- [templates/AGENTS.md](./templates/AGENTS.md) — workspace bootstrap file
- [templates/HEARTBEAT.md](./templates/HEARTBEAT.md) — heartbeat loop and protocol reference
- [templates/ROLE_TEMPLATE.md](./templates/ROLE_TEMPLATE.md) — root orchestrator soul file template

Use the templates as your starting point. Read the guide first.
