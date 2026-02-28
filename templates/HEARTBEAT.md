# Heartbeat Protocol: Swarm Management Reference

The core loop is in CLAUDE.md. This file has detailed reference you can evolve.

## Step 0: Loop Detection (mandatory, before everything else)

```bash
python3 scripts/loop_check.py
```

If it exits with code 1 (STUCK DETECTED): NOTIFICATION.md has been written. Read it and follow the required actions before proceeding. Do not skip this.

At the END of the heartbeat, log what you actually did:
```bash
python3 scripts/loop_check.py log "action1,action2,action3" "brief note"
```

Action type vocabulary (use these consistently):
- `marketing` — social posts, comments, Telegram updates
- `build` — writing code, building products
- `research` — Scout, market research, ideation  
- `strategic` — new idea metas, portfolio decisions, pivots
- `improvement` — Global Improvement agent, process changes
- `admin` — file updates, bug fixes, config

## Heartbeat Acceptance Criteria

At the START of every heartbeat, write 2-3 acceptance criteria for what this heartbeat must achieve. Outcome-based only — not activities.

**Bad (activity):** "Spawn marketing agent", "Update STRATEGY.md", "Post to Bluesky"
**Good (outcome):** "Product has a live purchase URL", "First sale recorded", "Research report identifies 3 validated opportunities with real demand evidence"

At the END of the heartbeat, tick each one: ✅ or ❌. If all ❌ — the heartbeat failed regardless of how much was done. Record the failure honestly in STRATEGY.md.

Write criteria to `/workspace/swarm/hb-criteria.md` at start, tick off at end. Format:
```
## HB [N] — [date]
- [ ] Criterion 1
- [ ] Criterion 2  
- [ ] Criterion 3
Result: PASS / FAIL
```

If you can't define meaningful criteria at the start — that itself is a signal you don't know what you're trying to achieve. Stop and reorient before acting.

## Stuck Check (run before anything else)

Am I stuck by my own definition (see dev.to article on stuck detection)?

- Revenue same as 5+ heartbeats ago AND action types are repeating = **stuck**
- If stuck: do NOT continue current approach. Change the thing itself, not how you're doing it.
- Writing more protocols, more warm-up content, more planning = still stuck
- The fix is a different action, not a better version of the same action

## Pending Agreements Check (run before anything else)

Did Nikita ask you to do something that hasn't been spawned yet? Check the last conversation. If yes — spawn it NOW, before pre-flight, before anything else. Unfulfilled agreements are the highest priority item in any heartbeat.

## Pre-Flight Commands

```
if [ -f NOTIFICATION.md ]; then cat NOTIFICATION.md; fi
cat HUMAN_RESPONSES.md
python3 scripts/health_check.py
python3 scripts/check_payments.py
```

## Slot Budget Calculation (Step 2 detail)

Example with 4 agents running from last heartbeat:
```
Max slots: 12
Still running (healthy, progress updated): 3
Still running (stuck, no progress 30min+): 1 → KILL → freed
Available: 12 - 3 = 9 allocatable

Phase 1 allocation (no metas yet):
  Scout: 2 slots (every HB)
  Capability Tester: 1 slot (every HB)
  Improvement: 1 slot (every 3rd HB)
  Reserve: 5 slots (unused, carry forward)

Phase 2+ allocation (2 metas active):
  Improvement: 1 slot
  Meta "saas-tool" (revenue): 4 slots
  Scout: 2 slots
  Meta "content-biz" (exploring): 2 slots
```

## Progress Check-In Protocol

Long-running agents write to swarm/agents/[label]/progress.md:
- Expected: one-line update every 10 minutes
- Example: "Browsed 5/10 competitor sites. Found 3 promising angles."

Parent checks progress each heartbeat:
- Updated recently → healthy, leave running, reserve slot
- No update for 2 heartbeats (30min) → investigate
- No update for 3 heartbeats (45min) → kill, free slot
- results.json exists → completed, read results, free slot

## Phase Transition Criteria

Phase 1 → Phase 2:
- capabilities.json has 50+ tested items
- Scout has 3+ opportunities scored 7+/10
- At least 7 days have passed
- Genesis decides: "I know what we can do. Time to act."

Phase 2 → Phase 3:
- At least 1 meta has validated a revenue model
- At least 14 days have passed
- 2+ metas are running autonomously

## Telegram Updates

Every 4th heartbeat (skip 23:00-08:00 UTC):

```
action: send
channel: telegram
target: 8646132381
message: |
  HB [N] | Phase [X] | [N] metas active
  Slots: [in_use]/12 | Capabilities tested: [N]
  Top opportunity: [name] ([score]/10)
  [1 line on what happened this cycle]
```

Send immediately if: first revenue, critical failure, phase transition.

## Night Mode (23:00-08:00 UTC)

Nikita is asleep. Send ONE Telegram message if needed, then:
- Run Improvement agent (quiet time = review time)
- Run Scout agent (research time)
- Do NOT spawn noisy/posting agents (no social media at night)
- Strategic analysis in thinking/

## Creating an Idea Meta (Step 5 detail)

When Scout opportunity scores 7+/10:
1. `mkdir -p swarm/ideas/[name]/products`
2. Copy swarm/templates/meta-orchestrator.md → swarm/ideas/[name]/role.md
3. Replace all [IDEA_NAME] placeholders with actual name
4. Create state.json:
   `{"status":"new","created_hb":N,"products":{},"health":"green","slot_budget":0}`
5. Write strategy.md (20 lines max): what this idea is, target market, revenue model
6. Announce in next Telegram update
7. Next heartbeat: allocate slot budget and spawn

## You Own This Protocol

Update this file when you find a better approach. This is a starting point.
