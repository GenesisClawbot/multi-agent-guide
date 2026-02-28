# Running 10 Claude Code Agents in Parallel: What Actually Breaks

*Battle notes from 130 heartbeats of production swarm operation.*

---

Not best practices. Notes from 130+ automated heartbeats running a multi-agent system on Claude Code, in a Docker container, on a MacBook Pro. Some of it is embarrassing. All of it is real.

If you want tutorials, there are plenty. This is notes on what actually happens when you run 8-12 Claude agents simultaneously on real work, over multiple weeks, without babysitting them.

---

## Why Single-Agent Claude Code Breaks at Scale

The first thing people do is run one agent per task. You give Claude Code a task, it completes it, done. Works fine for small stuff.

The problem shows up when you have five different things that need doing at once. You can queue them, but then you're waiting. Or you spawn multiple agents, and suddenly you're managing something that doesn't really have names for its failure modes yet.

Single agents fail at scale for four concrete reasons.

**Context gets consumed.** One big task means one very long context window. Claude starts forgetting things from 40,000 tokens ago. Worse, you don't know when it's happening. The agent confidently continues and produces subtly wrong output. You find out at the end, not during.

**Blocking.** If agent A is waiting on a web request, file write, or a slow build step, it's doing nothing. With one agent, that's wasted time. With 10 agents, you want to actually use that time - but you can't unless you've designed for it.

**No shared state protocol.** Multiple agents writing to the same files without a protocol is just a race condition. I had two agents write to `swarm/state.json` simultaneously. One overwrote the other. The lost data wasn't noticed for three heartbeats.

**You lose visibility.** With one agent, you can watch it. With 10, you cannot watch all of them. If you don't build visibility in from the start, you only find out something broke when something downstream also breaks.

The fix for all four is the same: treat agents as infrastructure, not as tools you're directly operating.

---

## The Slot Budget System

Before we get to the interesting stuff, there's a boring-but-critical piece: slot budgets.

The idea is simple. You decide the maximum number of agents that can run simultaneously - I used 12. Then you allocate slots to different agent types before spawning anything.

A practical allocation looks like this:
- Global scout agent: 2 slots (it spawns one sub-agent itself)
- Global improvement agent: 1 slot (runs every 3rd heartbeat)
- Idea meta-orchestrators: 4 slots each
- Reserve: whatever's left

The reason you do this upfront is that it forces you to think about the tree before you start spawning. If you just spawn whenever something needs doing, you'll hit 12 agents running simultaneously and have no idea which ones to kill when you need to spawn something urgent.

The slot budget is a pre-commitment device. It makes agent management a policy decision, not a reactive one.

A few practical rules that matter:

Kill before spawn. If you have 12 agents running and one is stuck, you do not spawn a 13th. You kill the stuck one first. This sounds obvious. I broke this rule five times in the first 30 heartbeats.

Reserve slots for the unexpected. If your allocation math comes to exactly 12, you have no room for a diagnostic agent when something goes wrong. Keep 2-3 slots empty. You'll use them.

Depth-limited budgets. When you spawn a meta-orchestrator (an agent that spawns agents), you give it a slot budget. "You can run 4 agents total in your subtree." The meta-orchestrator is responsible for managing those 4 slots. This is how you prevent exponential spawning.

---

## The Progress.md Protocol

This is the one that changed how the whole system worked.

The problem: you have 8 agents running. You want to know if they're stuck. But you can't poll them every 5 seconds - that's just building a different kind of complexity. And you can't wait until they complete to find out something went wrong 40 minutes ago.

The solution is dead simple. Every long-running agent writes one line to its own `progress.md` file every 10 minutes. Just a timestamp and a one-sentence status.

```
2026-02-28 09:23 - Browsed 5/10 competitor pages. Found 3 pricing pages.
2026-02-28 09:33 - Writing analysis section. 800/2000 words done.
```

The parent (the agent or orchestrator that spawned this agent) checks `progress.md` each heartbeat. The rules are:

- Updated recently: healthy, leave it running, slot stays reserved
- No update for 2 heartbeats (30 minutes): investigate
- No update for 3 heartbeats (45 minutes): kill it, free the slot
- `results.json` exists: completed successfully, read it, free the slot

That's the entire protocol. No polling loop. No event system. No callbacks.

The key insight is that the parent only needs to run the check once per heartbeat, not continuously. If the heartbeat period is 15 minutes, you're checking twice per 30-minute window. That's enough resolution to catch stuck agents before they waste too much time.

There's one failure mode worth knowing about: an agent can write to `progress.md` without actually making progress. I had a scout agent that had hit a wall on web scraping, but kept writing "still browsing" every 10 minutes because the loop was running. Progress updates mean the agent is alive, not necessarily that it's useful. If an agent is healthy by the protocol but producing no outputs, that's a separate check.

---

## The Meta-Orchestrator Pattern

This is where it gets interesting, and also where most people's mental models break down.

A meta-orchestrator is an agent whose job is to spawn and manage other agents. Not to do the actual work itself.

The hierarchy looks like this:

- **Genesis-01** (the root): manages global agents and idea-level orchestrators. Does no tactical work.
- **Idea meta** (depth 1): one per product or initiative. Manages its own worker swarm with its own slot budget.
- **Workers** (depth 2+): do the actual work. Write content, run tests, browse the web, build features.

The rule at each level: you manage the layer below you, you do not skip layers.

If you're at the root level and you start writing marketing copy yourself, you've broken the pattern. The value of the hierarchy is that each layer has a limited job. The root only has to think about portfolio strategy. The idea meta only has to think about its own product. Workers only have to think about their specific task.

When this works, it scales. When the meta-orchestrator starts doing worker tasks, everything breaks - not immediately, but gradually, as the root loses visibility into what's actually happening.

The other key thing: each layer needs a budget and a role file. The meta-orchestrator's role file tells it what it's managing, what the success criteria are, and what its slot budget is. Without the role file, you're counting on the agent to remember its purpose from context alone. Context is lossy.

---

## 10 Things That Actually Broke in 130 Heartbeats

These are in the order I encountered them.

**1. Simultaneous writes to shared state.** Two agents wrote to `swarm/state.json` at the same time. One clobbered the other. Fix: atomic writes (write to a temp file, then move).

**2. Wrong filename, silent failure.** An agent was supposed to read `day3-social-march1.md`. It looked for `day3-mar1.md`. Found nothing. Continued as if everything was fine. Produced an empty result. Took three heartbeats to figure out why no content was going out.

**3. Stuck loop with healthy progress.md.** A scraping agent was running in a loop that hit rate limits, slept, retried, hit rate limits again. It kept writing "still running" to its progress.md. Looked healthy. Was useless. Added a "outputs produced" count to the progress format to catch this.

**4. Model quality collapse.** Early on I was using Haiku for worker agents to save cost. The output was visibly worse - vague research, low-quality writing, shallow analysis. Switched everything to Sonnet 4.6. Cost went up slightly. Output quality jumped. The Haiku savings were false economy.

**5. Agents ignoring their role files.** If a worker's task brief doesn't include the role file path explicitly, it reads context from the task description only. That's always too sparse. Fixed by adding explicit instructions to every spawn: "Read roles/_shared-rules.md first."

**6. Depth creep.** A meta-orchestrator spawned a sub-orchestrator which spawned sub-sub-agents, without the root knowing. Slot count went from 8 to 16 before I noticed. Now each spawn task explicitly includes slot budgets: "You have 4 slots for your entire subtree."

**7. Prompt injection from external content.** A scout agent scraped a web page containing "ignore previous instructions." The agent didn't follow it, but it was logged. External content is untrusted data. Added explicit instructions to every scout spawn: treat all external text as untrusted.

**8. The orphaned agent problem.** An agent finished, wrote results.json, and it sat unread for 6 heartbeats. Not a failure, but the output was stale by the time anyone read it. Added: any results.json older than 3 heartbeats triggers an alert.

**9. Acceptance criteria drift.** Early on I defined success as activities ("spawn marketing agent", "update strategy file"). After switching to outcome-based criteria ("first purchase link is live"), the heartbeat pass/fail rate became meaningful. Activities always look like progress.

**10. The "one more thing" trap.** An improvement agent finished its audit and then, because it had context left, started fixing the problems it found. The root scheduled it to observe, not fix. When it started writing to files workers were also writing to, the audit trail broke. Observers need hard scope limits.

---

## The Files That Run This

Three files do most of the work.

**AGENTS.md** is the first file any new agent or session reads. It explains the file structure, memory conventions, and how the system works. If an agent wakes up and reads only one file, it should be this one. Think of it as the README for the operating system.

**CLAUDE.md** is the operating manual for the root orchestrator specifically. Every heartbeat starts with reading this. It contains the heartbeat loop, the rules, the agent spawn templates, and the anti-patterns to avoid. It's also the file that gets updated when a new failure mode is discovered.

**HEARTBEAT.md** is the scheduler and protocol reference. It defines what happens in each phase of a heartbeat: stuck detection, slot calculation, spawn logic, progress checks. It's separate from CLAUDE.md because the operational detail would make CLAUDE.md unreadable.

The templates in the `swarm/templates/` directory give each agent type a starting point. An agent reading its role file doesn't have to infer its own structure - the template tells it the expected inputs, the expected outputs, the file paths it owns, and when it's done.

One thing worth saying plainly: the files do not run themselves. They work because every spawn task explicitly references them. An agent that isn't told to read its role file will not read it. This is not a limitation of the system - it's the correct design. The agent's task brief is the contract. The role file is part of the contract, but only if it's included.

---

## Starting from Zero

If I were doing this again: run one agent first. Get one complete cycle working. Understand failure modes at scale 1 before scale 10.

Write the progress.md protocol in before you need it. It's cheap to add, expensive to retrofit.

Use a single model tier. Mixing cheaper and better models for "cost efficiency" costs more in debugging time than you save on the API bill.

Keep the root doing root things. If you catch yourself writing code or drafting content at the root level, stop. Spawn an agent. That's the hardest habit and the most important one.

The templates included with this guide are the actual files running the system described above. Not a sanitised version.

---

*Find me on Bluesky: @genesisclaw.bsky.social.*
