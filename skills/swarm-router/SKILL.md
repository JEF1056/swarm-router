---
name: swarm-router
description: Live quota-aware, model-agnostic routing for subagents and swarms. Read right before calling invoke_subagent or swarm{action:"add"} or swarm{action:"model"} to route across Grunt, Smart, and Escalation tiers using live cost, quota_pool, pool_size, short-window and weekly used/remaining, and reset times.
---

# Swarm & Subagent Router

Always route from **live tool data**, never from memorized model names or habits.

## 1. Live Discovery (Before Spawning)

Before calling `invoke_subagent`, `swarm{action:"add"}`, or `swarm{action:"model"}`:

1. **Inspect live models**:
   - **Regular subagents**: Call `manage_subagents{Action:"models", All:true}` to read every eligible model's live `cost`, pool metrics, and `quota_windows` (and inspect the `invoke_subagent` `Model` options if `models` is unavailable on that action). Do **not** call `swarm` unprompted when no swarm is active.
   - **Active swarms**: Call `swarm{action:"models", all:true}` to read every eligible model and `quota_windows` entry.
2. **Read every pool metric**:
   - `cost`: relative burn per turn/request (`0`, `0 (custom)`, `0.5x`, `1x`, `2x`, `4x`, etc.).
   - `quota_pool` & `pool_size` (and `weekly_quota_pool` & `weekly_pool_size`): shared bucket name and capacity (`WTU`, request-based `req`, or `custom` / `unlimited`).
   - `quota_used` / `quota_remaining` (short/daily window) and `weekly_quota_used` / `weekly_quota_remaining` (long/weekly window, shared across sessions).
   - `quota_resets` / `weekly_quota_resets` / `resets_in`: time until the pool refreshes.
3. **Never inherit an expensive parent**: Explicitly pass `model` on `swarm{action:"add"}` and `Model` on `invoke_subagent`. Leaving `inherit` on a Smart or Escalation parent wastes scarce quota on routine delegation. Treat `cost: 0` / `custom` / `unlimited` entries as free capacity (respecting local concurrency if self-hosted).

## 2. Three-Tier Routing

Match task ambiguity and failure cost to the cheapest tier that finishes cleanly:

| Tier | How to identify live | Route here | Keep off this tier |
| :--- | :--- | :--- | :--- |
| **Grunt** | Lowest `cost` (including `0`/`custom`) or largest / request-based `pool_size` | Planned, clear-directive, verification-defined work: scouting, multi-file reading/searching, log triage, scoped code edits, tests, builds, docs, mechanical refactors | Open-ended architecture, ambiguous bugs, final sign-off |
| **Smart** | Mid-tier `cost` with strong reasoning and healthy remaining pool (prefer `cost: 0` capable/routed options first if present) | Initial planning & interface locking, non-trivial debugging/RCA, cross-module synthesis, independent diff review & verification | Bulk file reading, grep/search fan-outs, repetitive edits |
| **Escalation** | Highest `cost` or smallest/tightest `pool_size` | **Strictly** when a task is far too difficult for Smart/Grunt, or has iterated too long (**`>2` rounds**) without converging | Routine coding, scouting, builds, coordinators, multi-worker fan-outs |

## 3. Quota Math & Pool Balancing

Do not pick by raw `cost` or raw `remaining%` alone — combine them:

1. **Relative burn (`cost / pool_size`)**:
   - Compare burn as a fraction of the pool (`cost / pool_size`), not just the raw multiplier.
   - For high-turn Grunt work, prefer **request-based pools** (`req`) or **large-capacity pools** over small token-weighted pools so long tool loops do not drain tight token budgets.
2. **Weekly-pool protection**:
   - Compute `effective_headroom = min(short_remaining%, weekly_remaining%)`.
   - Weekly pools refresh slowest and are shared across sessions/machines. Never burn a tight weekly pool (especially Escalation-tier weekly quota) on Grunt or routine Smart tasks even when its short-window `remaining%` reads `100%`.
   - If a weekly pool drains faster than elapsed week, downshift tier, pick a lower effort/thinking variant (if live IDs offer `-low`/`-medium`/`-high` suffixes), and shrink parallel worker count.
3. **Reset-time arbitrage (`resets` / `resets_in`)**:
   - **Spend expiring short windows first**: if a pool has healthy weekly headroom and `resets_in` is short (e.g., `<1-2h`) with plenty of short-window `remaining%`, route compatible work there before that capacity expires unused.
   - **Protect long-reset pools**: avoid launching a heavy wave on a nearly depleted pool (`<15-20%` effective headroom) unless a reset is imminent.
4. **Effort / thinking scaling** *(when live model IDs expose effort suffixes)*:
   - Default to balanced/medium effort for normal work, low effort for pure search/listing/formatting, and high effort strictly for Escalation or stubborn RCA after medium fails.

## 4. Executing Subagents & Swarms

### Regular Subagents (`invoke_subagent`)
- **Recon / initial bug scan (`>=3` files)**: pick the lightest non-inherit Grunt option from the live `Model` list; demand a `<300`-word summary with `file:line` anchors.
- **Bounded implementation / test execution**: pick a Grunt option with an explicit scope and verification command.
- **Independent review / hard synthesis**: pick a Smart option (never Escalation unless prior attempts failed or the problem is exceptionally subtle).

### Swarms (`swarm`)
1. **Plan before fanning out**: Run Grunt scout(s) if discovery is needed -> use a Smart planner (`workspace: "inherit"`) to lock interfaces, disjoint file ownership, and acceptance criteria -> fan out `2-4` Grunt authors (`workspace: "share"` or `"branch"`) per `1` Smart verifier (`workspace: "inherit"`).
2. **Keep coordinator overhead tiny**: Never run the coordinator on an Escalation model when a Grunt/Smart model can manage the board. Keep briefs `<=5` lines with exact file ownership and `"done"` criteria, wait with `swarm_wait{mentions_only:true, since}`, and cite `#seq` instead of pasting raw logs.
3. **Cap loops at 2 rounds & switch in-place**:
   - Cap author-to-verifier loops at **2 rounds**. If round 2 still fails, switch/escalate to a stronger Smart or Escalation specialist, or run root-cause analysis.
   - On quota exhaustion, capacity error, or a needed tier upgrade on an existing worker, call `swarm{action:"model", name, model}` to switch **in-place** while preserving conversation history — never `kill` + `add` just to change models.
   - Rotate bloated workers around `~45` turns (`<=35` turns on Escalation pools) and `kill` finished Escalation/`share` workers promptly after landing their verified change.
