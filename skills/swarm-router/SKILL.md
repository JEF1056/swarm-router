---
name: swarm-router
description: >-
  Swarm model routing, quota balancing, thinking-effort scaling, and plan-first
  execution policy. Read ONLY when /swarm is invoked, when proactively starting
  a swarm for a complex/highly parallelizable task, or when asked to check live
  Gemini/3P quotas or design a swarm roster.
---

# Swarm Model Routing, Quota Balancing & Plan-First Policy

When starting or running a swarm (via `/swarm` or proactively for complex/highly parallelizable tasks), this policy overrides the default "leave `model` off" rule unless the user explicitly specifies a model for a worker or task.

## 1. Inspect Live Models & Quotas First

Before spawning workers, call `swarm{action:"models", all:true}` (or `manage_subagents{Action:"models", All:true}` outside an active swarm) to inspect live `models`, `tiers`, `quotas`, and `quota_windows`.

- **Never hardcode version numbers**: Always pick exact model IDs from the live `models` list, preferring the newest version of a family when multiple exist.
- **Inspect live quota buckets**: Read the actual `quota_windows` groups and reset windows (e.g., Gemini vs. 3P short-term `5h` and `weekly` buckets). Multiple machines share the same account quotas — never assume 100% headroom or that this machine is the only consumer.

## 2. Model Tiers (Highest → Lowest Quota Cost)

Always pass an explicit `model` ID from the live `models` list on `swarm{action:"add"}`.

### Smart Tier — Planning, Verification, Coordination, Hard RCA
Match the latest available version by family keyword (highest → lowest quota cost):
1. **`opus`** / **`fable`** *(3P pool)* — Hardest architecture plans, critical final verification, or stuck RCAs.
2. **`carbon`** *(Gemini pool)* — Heaviest Gemini flagship for deepest long-horizon planning, complex architecture, and high-stakes verification.
3. **`sonnet`** *(3P pool)* — Default 3P workhorse for planning, verification, and coordination.
4. **`neon`** *(Gemini pool)* — Balanced, fast Gemini planner, coordinator, and verifier (best default for most Gemini agentic tasks).

- **`argon` restriction**: Do **not** use `argon` unless the other smart Gemini models (`carbon` and `neon`) are unavailable (missing from `models` or exhausted).

### Grunt Tier — Implementation, Tests, Refactors, Research, Docs, Narrator/Monitor
Match the latest available version by family keyword (highest → lowest quota cost):
1. **`hydrogen`** *(Gemini pool)* — Non-trivial multi-file coding, tricky bug fixes, and complex refactors.
2. **`flash`** *(Gemini pool)* — Default for straightforward edits, unit tests, builds, scans, docs, and narrator/monitor roles (`flash-lite` / `flash_lite` for trivial scans).

### Custom Models (Local / User-Configured)
If any custom models (e.g., local/self-hosted endpoints or user-added custom entries) appear in `models`:
- **Out-of-the-box thinking**: Feel free to spawn or switch to a custom model for fresh ideas, lateral brainstorming, or breaking out of a stuck loop when standard models converge on the same dead end.
- **Refusal fallback**: Try an available custom model when standard hosted models refuse a legitimate user request.
- **Zero cloud quota cost**: Custom/local models do not consume shared Gemini or 3P cloud quotas, though avoid overloading single-host local servers with excessive concurrency.

### Dynamically Classifying New / Unknown Models
When `models` returns a new or unfamiliar codename (not listed above), classify it dynamically in this order:
1. **Live tier anchors**: Check `tiers` in `models` output (`flash_lite`, `flash`, `pro`). Any model mapped to `flash_lite` or `flash` is **Grunt**; any model mapped to `pro` is **Smart** (or upper Grunt).
2. **Pool & list-position heuristic**:
   - Models with `flash`, `lite`, `mini`, `haiku`, or `oss` tags → **Grunt**.
   - Local/user-configured custom models → **Custom** (eligible for out-of-the-box thinking, refusal fallback, and quota-free work).
   - Within a provider's block in `models`, models are typically listed strongest/priciest → fastest/cheapest. An unrecognized codename in the **3P pool** or listed above known grunt models defaults to **Smart** — **never put an unknown codename on high-volume grunt fan-outs** (prevents accidentally burning flagship quota).
3. **Ask once if ambiguous**: If a brand-new codename appears and you want to use it, ask the user in 1 line (*"Saw new model `<id>` — should I treat it as Smart or Grunt, and where in cost order?"*) and use their answer for the session (optionally updating this skill if they want it saved across machines).

## 3. Starting `/swarm` From a Lightweight Main Model

If the user starts `/swarm` from a grunt main model (e.g., Flash or Hydrogen):
- **Upgrade coordination when needed**: For complex multi-slice tasks, switch yourself (the coordinator) to a smart model via `swarm{action:"model", model:"<smart-model-id>"}` (e.g., latest `sonnet` or `neon` at `-medium` based on live pool headroom), **or** keep coordination light and immediately delegate planning and verification to dedicated smart workers (`planner`, `verifier`).
- **Always route planning and verification to smart models**, never leave them on the grunt main model.

## 4. Cross-Quota Balancing (Hourly & Weekly)

Balance models across all active quota pools (e.g., Gemini vs. 3P) by checking **both** the short-term (hourly/`5h`) **and** `weekly` windows in `quota_windows` — never look at only one window:

- **Evaluate each pool's bottleneck window**: For each pool, look at **both** short-term (hourly/`5h`) remaining % and `weekly` remaining % (along with time until each resets). The tighter of the two windows governs that pool's true headroom (e.g., a pool with 90% hourly left but 15% weekly left is tight; a pool with 20% hourly left resetting in 4 hours is tight).
- **Actively drain the fuller quota**: If one pool is burning faster or has less headroom on **either** its hourly/`5h` or `weekly` window, shift eligible roles to the **other** pool so both quotas stay balanced:
  - **Gemini tighter (or 3P has >= Gemini headroom and 3P hourly/weekly >= 25%)**: Grunt workers (`hydrogen`, `flash`) draw from Gemini, so route smart roles (planning, verification, coordination, hard RCA) to **3P** (`sonnet` default; `opus`/`fable` only for the hardest tasks) to run down 3P and spare Gemini.
  - **3P tighter (3P hourly or weekly lower than Gemini, or 3P short-term < 25%)**: Route smart roles to **Gemini** (`neon` default; `carbon` for deepest planning, complex architecture, and high-stakes verification; `argon` only if both `carbon` and `neon` are unavailable) so 3P recovers.
- **Re-check mid-swarm**: On long or multi-wave swarms, re-run `swarm{action:"models", all:true}` before spawning new waves so routing adapts as hourly/weekly windows shift across machines.
- **Balance within grunt tier**: Default to `flash` for routine implementation, tests, docs, and scans; assign `hydrogen` to harder coding slices or after 1 failed `flash` attempt.
- **When both pools are tight**: If both pools' short-term windows drop below 30% or their weekly windows are burning faster than elapsed week, downshift one tier (`opus/fable -> sonnet`, `carbon -> neon`, `hydrogen -> flash`), lower concurrency, avoid speculative fan-outs, and offload eligible tasks to available custom/local models.
- **On quota exhaustion**: Switch the blocked worker with `swarm{action:"model", name, model}` to the healthiest pool in the same tier — never `kill` and re-`add` just for quota.

## 5. Thinking Effort Scaling

Match the effort variant available for the chosen model in `models` (`-low`, `-medium`, `-high`; if a model has no effort suffix, use its base ID). Default to **`-medium`** and cap at **`-high`**:
- **`-low`**: Mechanical or trivial work (formatting, renames, boilerplate, simple docs, file/log scans, narrator, monitor).
- **`-medium`** *(default)*: Most tasks — standard implementation, test writing, refactoring, normal planning, routine verification, and coordination.
- **`-high`** *(hard cap)*: Extremely difficult tasks only — subtle concurrency/state bugs, hard RCA after `-medium` fails, or high-stakes architecture/security planning and verification.
- **Never select `-xhigh` or `-max`** unless the user explicitly requests it.

## 6. Plan First on Complex Tasks

If a task is complex (touches multiple modules/systems, has architectural ambiguity, shared interfaces, or >3 parallel slices):
1. **Plan first**: Before spawning implementation workers, have the smart coordinator or a single smart `planner` worker (`workspace: "inherit"`, `sonnet` / `neon` default, or `carbon` / `opus` / `fable` for deepest/hardest architecture at `-medium`) produce a concise plan with disjoint file ownership, interfaces, edge cases, and verification criteria.
2. **Execute lean**: Spawn grunt workers (`flash` / `hydrogen`, `workspace: "share"` for concurrent editors) against the locked plan, paired with 1 smart `verifier` (`workspace: "inherit"`) per 2–4 authors.
3. **Escalate lazily, cap loops**: Cap author<->verifier fix loops at 2 rounds on the same tier; escalate `flash -> hydrogen -> smart RCA` (or try an available custom model for a fresh angle) instead of burning retries.

## 7. Multi-Machine Shared Quota Guardrails

1. **Right-size concurrency**: Prefer 2–4 focused workers over 8+ overlapping ones; every `@all` wakes all workers and multiplies token burn.
2. **Keep briefs short**: Inline exact file paths and done criteria (`<=5` lines); reference earlier posts by `#seq` instead of copy-pasting logs.
3. **Kill finished workers immediately**: Land `share`/`branch` changes first, then kill finished workers; rotate long-running workers around ~45 turns to avoid >80k-token context bloat.
4. **Respect 3P prompt cache windows**: Run 3P (`opus`/`fable`/`sonnet`) planners and verifiers in tight bursts rather than leaving them idle >5 minutes and resuming with large context.
