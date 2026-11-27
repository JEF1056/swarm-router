---
name: swarm-router
description: Live quota-aware, model-agnostic routing for subagents and swarms. Read right before calling invoke_subagent, swarm{action:"add"}, swarm{action:"model"}, or manage_subagents{Action:"model"} to choose an adaptive topology and route across Basic, Grunt, Smart, and Escalation tiers by task difficulty, expected burn, and live quota.
---

# Swarm & Subagent Router

Route every spawn from **live tool data**, task difficulty, task type, and expected burn — never memorized model names or a mandated topology.

## 1. Live Discovery

Before calling `invoke_subagent`, `swarm{action:"add"}`, `swarm{action:"model"}`, or `manage_subagents{Action:"model"}`:

1. **Fetch live models**:
   - **Regular subagents**: `manage_subagents{Action:"models", All:true}` (do **not** call `swarm` when no swarm is active).
   - **Active swarms**: `swarm{action:"models", all:true}`.
2. **Read every pool metric**:
   - `cost`: relative burn per request or token rate (`0`, `0 (custom)`, `0.15/1.25 WTU/1M`, `1x`, `2x`, `4x`, etc.).
   - `quota_pool` & `pool_size` (and `weekly_quota_pool` & `weekly_pool_size`): shared bucket and capacity (`req`, `WTU`, or `custom`/`unlimited`).
   - `quota_used` / `quota_remaining` and `weekly_quota_used` / `weekly_quota_remaining`: short-window and weekly consumption.
   - `quota_resets` / `weekly_quota_resets` / `resets_in`: refresh countdown.
3. **Pass explicit models**: Always set `Model` (`invoke_subagent`) or `model` (`swarm{action:"add"}`) so cheaper delegated work never inherits a Smart or Escalation parent. Treat `cost: 0`, `custom`, and `unlimited` entries as free capacity (respecting local concurrency if self-hosted).

## 2. Four-Tier Model Selection

Pick the cheapest tier that finishes cleanly **on the first pass** — avoid false economy where an underpowered model fails, churns retries, and costs more than starting one tier up.

| Tier | Live signals | Use for | Keep off this tier |
| :--- | :--- | :--- | :--- |
| **Basic** | Ultra-cheap / smallest-class models (`-lite`, `-mini`, `-nano`, `-small`, lowest `cost`, or huge lite pools) that are likely too weak for good outcomes requiring judgment | Strictly zero-thinking, clear-cut actions: exact-spec file edits, formatting/lint fixes, mechanical renames, boilerplate fill-in, running commands/tests and relaying output, literal text/line extraction | Reasoning, ambiguity, bug diagnosis, codebase synthesis, non-trivial code or tests, reviews |
| **Grunt** | Low-`cost` capable workhorse models (`0`/`custom`, request-metered, or low token rate) with large `pool_size` | Clear-directive, verification-defined engineering: codebase scouting & multi-file exploration, log/error triage, scoped feature/bug edits, unit tests, builds, docs, structured refactors | Open-ended architecture, subtle/ambiguous RCA, high-stakes final sign-off |
| **Smart** | Mid-tier `cost` strong reasoners with healthy headroom (prefer `cost: 0` capable/routed options first if present) | Non-trivial planning & interface locking, ambiguous/cross-module RCA, complex single-owner edits, independent diff review & verification | Bulk file reads, search fan-outs, mechanical edits, repetitive loops |
| **Escalation** | Highest `cost`, smallest/tightest `pool_size`, or max thinking effort | **Strictly** tasks too hard for Smart (deep concurrency/security/perf/architecture) or stuck past the 2-round cap | Routine coding, scouting, builds, coordinators, multi-worker fan-outs |

## 3. Task Type, Expected Burn & Pool Math

Weigh **task difficulty**, **expected burn**, **relative pool cost (`cost / pool_size`)**, and **headroom** together — never pick by raw `cost` or raw `remaining%` alone:

1. **Match burn profile to pool type**:
   - **High-turn / large-context loops** (broad scouting, build/test loops, noisy log triage, multi-file edits): token-weighted pools grow roughly quadratically as history accumulates across turns. Strongly prefer `cost: 0`, `custom`, **request-based (`req`)**, or large-capacity pools over tight token-weighted pools.
   - **Short, high-leverage turns** (planning, targeted RCA, diff review, subtle single-shot edits): few turns and bounded context mean a Smart model consumes little pool fraction while preventing expensive rework.
   - **Parallel waves**: budget `workers * expected_burn` against `cost / pool_size`, and spread concurrent workers across independent healthy pools when multiple compatible buckets exist.
2. **Protect weekly pools**:
   - Route by `effective_headroom = min(short_remaining%, weekly_remaining%)`.
   - Weekly pools refresh slowest and are shared across sessions; never burn tight weekly quota (especially Escalation) on Basic, Grunt, or routine Smart work even when the short window reads `100%`.
   - If weekly burn outpaces elapsed week, downshift tier, lower effort suffix, and shrink concurrency.
3. **Arbitrage reset windows (`resets_in`)**:
   - Spend expiring short-window capacity (`resets_in` soon, e.g., `<1-2h`) first when weekly headroom is healthy.
   - Avoid heavy waves on pools with `<15-20%` effective headroom unless a reset is imminent.
4. **Scale thinking/effort suffixes** *(when live model IDs expose them, e.g., `-low`/`-medium`/`-high`/`-max`)*:
   - Use `-low` for mechanical edits, search, and listing; `-medium` by default; `-high`+ strictly for Escalation or stubborn RCA after `-medium` fails.

## 4. Adaptive Topology & Execution

Do **not** mandate a fixed topology. Add each stage only when uncertainty, complexity, or failure risk justifies its cost:

- **Always scout when uncertain**: Never guess target files, signatures, call sites, or failure surfaces — whenever parameters or codebase facts are uncertain, run Grunt scout(s) (or Basic strictly for literal lookup/extraction) to return grounded `<300`-word `file:line` facts before planning or editing. Skip scouting only when scope is already verified.
- **Plan only when complexity warrants it**: If the task spans shared contracts, multiple interacting modules, or non-obvious trade-offs/blast radius, run a Smart planner (`workspace: "inherit"`) to lock interfaces, disjoint file ownership, and acceptance criteria before execution. Skip dedicated planning for clear, isolated, or mechanical work.
- **Execute at the right granularity**:
  - One bounded slice -> `invoke_subagent` with explicit scope and verification command (no swarm needed).
  - Independent parallel slices -> swarm workers matched to slice difficulty (Basic for exact zero-thinking edits, Grunt for scoped engineering); give concurrent editors `workspace: "share"` or `"branch"` and keep read-only workers on `"inherit"`.
  - Tightly coupled or subtle logic -> one Smart owner instead of splitting into parallel Grunt workers that churn on shared assumptions.
- **Verify proportionally to risk**:
  - Mechanical or deterministic changes fully proven by tests/typechecks/linters -> author self-verification suffices; skip a dedicated verifier.
  - Non-trivial code, multi-slice integration, or high failure cost -> pair with an independent Smart verifier (`workspace: "inherit"`, roughly `1` per `2-4` authors). Occasionally pick a verifier from a **different model family** (when live pools offer one with balanced `cost / pool_size` and healthy `effective_headroom`) to catch shared blind spots without straining tight weekly budgets.
- **Keep overhead low & converge cleanly**:
  - Keep coordinators cheap (never Escalation), briefs `<=5` lines with exact ownership and `"done"` criteria, waits on `swarm_wait{mentions_only:true, since}`, and cross-references by `#seq` instead of pasting raw logs.
  - If a Basic worker makes mistakes or stalls, upgrade immediately to Grunt instead of coaching it.
  - Cap unproductive retry or author/verifier loops at **2 rounds** — then run root-cause analysis or switch the worker's model **in-place** (`swarm{action:"model"}` or `manage_subagents{Action:"model"}`, preserving history rather than `kill` + `add`; also use in-place switching on quota or capacity errors).
  - Rotate bloated workers around `~45` turns (`<=35` on tight/Escalation pools) and `kill` finished `share`/Escalation workers promptly after landing verified work.
