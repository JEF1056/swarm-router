---
name: swarm-router
description: >-
  Swarm model routing, quota balancing, thinking-effort scaling, and plan-first
  execution policy. Read ONLY when /swarm is invoked or when the user asks to
  check live Gemini/3P quotas or design a swarm roster.
---

# Swarm Model Routing, Quota Balancing & Plan-First Policy

When `/swarm` is active, this policy overrides the default "leave `model` off" rule unless the user explicitly specifies a model for a worker or task.

## 1. Inspect Live Models & Quotas First

Before spawning workers, call `swarm{action:"models", all:true}` (or `manage_subagents{Action:"models", All:true}` outside an active swarm) to inspect live `models`, `quotas`, and `quota_windows`.

- **Never hardcode version numbers**: Resolve exact model IDs dynamically from the live `models` list by matching family names (preferring the latest available version of a family when multiple versions exist).
- **Inspect live quota buckets**: Read the actual `quota_windows` groups and reset windows (e.g., Gemini vs. 3P/Claude short-term `5h` and `weekly` buckets). Multiple machines share the same account quotas — never assume 100% headroom or that this machine is the only consumer.

## 2. Model Tiers (Highest → Lowest Quota Cost)

Always pass an explicit `model` ID from the live `models` list on `swarm{action:"add"}`.

### Smart Tier — Planning, Verification, Coordination, Hard RCA
Match the latest available version by family keyword (in cost-to-quota order):
1. **`opus`** *(3P / Claude pool)* — Hardest architecture plans, critical final verification, or stuck RCAs.
2. **`neon`** *(Gemini pool)* — High-capability Gemini planner, coordinator, or complex verifier.
3. **`sonnet`** *(3P / Claude pool)* — Default 3P workhorse for planning, verification, and coordination.
4. **`carbon`** *(Gemini pool)* — Balanced Gemini planner and verifier.
5. **`argon`** *(Gemini pool)* — Most quota-efficient smart model for routine verification, triage, and light coordination.

### Grunt Tier — Implementation, Tests, Refactors, Research, Docs, Narrator/Monitor
Match the latest available version by family keyword (in cost-to-quota order):
1. **`hydrogen`** *(Gemini pool)* — Non-trivial multi-file coding, tricky bug fixes, and complex refactors.
2. **`flash`** *(Gemini pool)* — Default for straightforward edits, unit tests, builds, codebase scans, docs, and narrator/monitor roles (use `flash-lite` / `flash_lite` for trivial `invoke_subagent` scans when available).

### Adapting to Future / Renamed Models
If a future release renames or adds model families not listed above:
- Infer **pool** from `quota_windows` / provider prefix (`gemini-*` vs. `claude-*` / 3P).
- Infer **tier** from live `models` metadata (`tiers`, `display_names`, `notes`): place frontier/Pro/reasoning-class models in the **Smart Tier** and fast/Flash/Lite/coding-grunt models in the **Grunt Tier**, preserving the same smart-vs-grunt split and cross-pool balancing.

## 3. Starting `/swarm` From a Lightweight Main Model

If the user starts `/swarm` from a grunt main model (e.g., Flash or Hydrogen):
- **Upgrade coordination when needed**: For complex multi-slice tasks, switch yourself (the coordinator) to a smart model via `swarm{action:"model", model:"<smart-model-id>"}` (e.g., latest `sonnet`, `carbon`, or `argon` at `-medium` based on live pool headroom), **or** keep coordination light and immediately delegate planning and verification to dedicated smart workers (`planner`, `verifier`).
- **Always route planning and verification to smart models**, never leave them on the grunt main model.

## 4. Cross-Quota Balancing

- **Protect Gemini quota for grunt work**: Grunt workers (`hydrogen`, `flash`) draw from the Gemini pool. When the 3P pool's remaining headroom >= Gemini's (and its short-term window >= 25%), route smart roles (planning, verification, coordination) to **3P** (`sonnet` default; `opus` only for the hardest tasks).
- **Shift smart roles to Gemini when 3P is lower**: When Gemini headroom > 3P headroom (or 3P short-term < 25%), route smart roles to **Gemini** (`argon` / `carbon` default; `neon` for hard tasks).
- **Balance within grunt tier**: Default to `flash` for routine implementation, tests, docs, and scans; assign `hydrogen` to harder coding slices or after 1 failed `flash` attempt.
- **Short-term & weekly pacing**: If any short-term window drops below 30% or a weekly window is burning faster than time elapsed, downshift one tier (`opus -> sonnet`, `neon -> carbon -> argon`, `hydrogen -> flash`), lower concurrency, and avoid speculative fan-outs.
- **On quota exhaustion**: Switch the blocked worker with `swarm{action:"model", name, model}` to the healthiest pool in the same tier — never `kill` and re-`add` just for quota.

## 5. Thinking Effort Scaling

Match the effort variant available for the chosen model in `models` (`-low`, `-medium`, `-high`; if a model has no effort suffix, use its base ID). Default to **`-medium`** and cap at **`-high`**:
- **`-low`**: Mechanical or trivial work (formatting, renames, boilerplate, simple docs, file/log scans, narrator, monitor).
- **`-medium`** *(default)*: Most tasks — standard implementation, test writing, refactoring, normal planning, routine verification, and coordination.
- **`-high`** *(hard cap)*: Extremely difficult tasks only — subtle concurrency/state bugs, hard RCA after `-medium` fails, or high-stakes architecture/security planning and verification.
- **Never select `-xhigh` or `-max`** unless the user explicitly requests it.

## 6. Plan First on Complex Tasks

If a task is complex (touches multiple modules/systems, has architectural ambiguity, shared interfaces, or >3 parallel slices):
1. **Plan first**: Before spawning implementation workers, have the smart coordinator or a single smart `planner` worker (`workspace: "inherit"`, `sonnet` / `carbon` / `neon` / `opus` at `-medium` by difficulty and pool headroom) produce a concise plan with disjoint file ownership, interfaces, edge cases, and verification criteria.
2. **Execute lean**: Spawn grunt workers (`flash` / `hydrogen`, `workspace: "share"` for concurrent editors) against the locked plan, paired with 1 smart `verifier` (`workspace: "inherit"`) per 2–4 authors.
3. **Escalate lazily, cap loops**: Cap author<->verifier fix loops at 2 rounds on the same tier; escalate `flash -> hydrogen -> smart RCA` instead of burning retries.

## 7. Multi-Machine Shared Quota Guardrails

1. **Right-size concurrency**: Prefer 2–4 focused workers over 8+ overlapping ones; every `@all` wakes all workers and multiplies token burn.
2. **Keep briefs short**: Inline exact file paths and done criteria (`<=5` lines); reference earlier posts by `#seq` instead of copy-pasting logs.
3. **Kill finished workers immediately**: Land `share`/`branch` changes first, then kill finished workers; rotate long-running workers around ~45 turns to avoid >80k-token context bloat.
4. **Respect 3P prompt cache windows**: Run Claude (`opus`/`sonnet`) planners and verifiers in tight bursts rather than leaving them idle >5 minutes and resuming with large context.
