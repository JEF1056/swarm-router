# Swarm Model Routing & Quota Policy

When running `/swarm` (or delegating subagents), this policy overrides the default "leave `model` off" rule unless the user explicitly specifies a model for a worker or task.

## 1. Inspect Live Quotas First

Before spawning workers, call `swarm{action:"models", all:true}` (or `manage_subagents{Action:"models", All:true}` for non-swarm subagents) to inspect live `quota_windows` and per-model `quotas`.

Multiple machines share the same `5h` and `weekly` quota pools:
- **Gemini pool** (`gemini-5h` / `gemini-weekly`): Neon, Carbon, Argon, Hydrogen, Flash
- **3P pool** (`3p-5h` / `3p-weekly`): Opus, Sonnet, GPT-OSS

Never assume 100% headroom or that this machine is the only consumer.

## 2. Model Tiers (Highest → Lowest Quota Cost)

Always pass an explicit `model` on `swarm{action:"add"}`.

### Smart Tier — Planning, Verification, Coordination, Hard RCA
1. **Opus** (`claude-opus-5-5-{low,medium,high}`) — `3p` pool. Reserve for the hardest architecture plans, critical final verification, or stuck RCAs.
2. **Neon** (`gemini-4-neon-{low,medium,high}`) — `gemini` pool. High-capability Gemini planner, coordinator, or complex verifier.
3. **Sonnet** (`claude-sonnet-5-5-{low,medium,high}`) — `3p` pool. Default 3P workhorse for planning, verification, and coordination.
4. **Carbon** (`gemini-4-carbon-{low,medium,high}`) — `gemini` pool. Balanced Gemini planner and verifier.
5. **Argon** (`gemini-4-argon-{low,medium,high}`) — `gemini` pool. Most quota-efficient smart model for routine verification, triage, and light coordination.

### Grunt Tier — Implementation, Tests, Refactors, Research, Docs, Narrator/Monitor
1. **Hydrogen** (`gemini-4-hydrogen-{low,medium,high}`) — `gemini` pool. Non-trivial multi-file coding, tricky bug fixes, and complex refactors.
2. **Flash** (`gemini-3.8-flash-{low,medium,high}`) — `gemini` pool. Default for straightforward edits, unit tests, builds, codebase scans, docs, and narrator/monitor roles.
*(Use `flash_lite` / `gemini-3.5-flash-lite` for trivial `invoke_subagent` scans when available. If a model ID is absent on a machine, pick the closest available model in the same tier from `models`.)*

## 3. Cross-Quota Balancing

- **Protect Gemini quota for grunt work**: Grunt workers run on the Gemini pool (`hydrogen`, `flash`). When `3p` headroom >= `gemini` headroom (and `3p-5h` >= 25%), route smart roles (planning, verification, coordination) to **3P** (`sonnet` default; `opus` only for the hardest tasks).
- **Shift smart roles to Gemini when 3P is lower**: When `gemini` headroom > `3p` headroom (or `3p-5h` < 25%), route smart roles to **Gemini** (`argon` / `carbon` default; `neon` for hard tasks).
- **Balance within grunt tier**: Default to `flash` for routine implementation, tests, docs, and scans; assign `hydrogen` to harder coding slices or after 1 failed `flash` attempt.
- **5h & weekly pacing**: If either `5h` window drops below 30% or `weekly` is burning faster than time elapsed, downshift one tier (`opus -> sonnet`, `neon -> carbon -> argon`, `hydrogen -> flash`), lower concurrency, and avoid speculative fan-outs.
- **On quota exhaustion**: Switch the blocked worker with `swarm{action:"model", name, model}` to the healthiest pool in the same tier — never `kill` and re-`add` just for quota.

## 4. Thinking Effort Scaling

Scale effort by task difficulty; default to **`-medium`** and cap at **`-high`**:
- **`-low`**: Mechanical or trivial work (formatting, renames, boilerplate, simple docs, file/log scans, narrator, monitor).
- **`-medium`** *(default)*: Most tasks — standard implementation, test writing, refactoring, normal planning, routine verification, and coordination.
- **`-high`** *(hard cap)*: Extremely difficult tasks only — subtle concurrency/state bugs, hard RCA after `-medium` fails, or high-stakes architecture/security planning and verification.
- **Never select `-xhigh`** unless the user explicitly requests it.

## 5. Plan First on Complex Tasks

If a task is complex (touches multiple modules/systems, has architectural ambiguity, shared interfaces, or >3 parallel slices):
1. **Plan first**: Before spawning implementation workers, have the coordinator or a single smart `planner` worker (`workspace: "inherit"`, `sonnet-medium` / `carbon-medium` / `neon-medium` / `opus-medium` by difficulty and pool headroom) produce a concise plan with disjoint file ownership, interfaces, edge cases, and verification criteria.
2. **Execute lean**: Spawn grunt workers (`flash` / `hydrogen`, `workspace: "share"` for concurrent editors) against the locked plan, paired with 1 smart `verifier` (`workspace: "inherit"`) per 2–4 authors.
3. **Escalate lazily, cap loops**: Cap author<->verifier fix loops at 2 rounds on the same tier; escalate `flash -> hydrogen -> smart RCA` instead of burning retries. Kill finished `share` workers immediately after verification passes (land changes first), and rotate heavy workers before ~45 turns.
