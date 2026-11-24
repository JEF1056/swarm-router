# Swarm Router (Antigravity Plugin)

An Antigravity plugin designed for running cheap/fast models (like **Flash** or **Hydrogen**) in your main conversation, hinting to escalate complex tasks to `/swarm`, and lazy-loading version-agnostic `/swarm` model routing across shared **Gemini** and **3P (Claude)** quota pools.

## Install

### For an Antigravity Agent (paste this repo URL on any machine)

Tell your Antigravity agent:
> Install the `swarm-router` plugin from `https://github.com/JEF1056/swarm-router`

Or run directly in a shell:

```bash
git clone https://github.com/JEF1056/swarm-router.git ~/.gemini/config/plugins/swarm-router
agy plugin validate ~/.gemini/config/plugins/swarm-router --strict
agy plugin list
```

To update later on any machine:

```bash
git -C ~/.gemini/config/plugins/swarm-router pull
```

*(Alternatively, install a copy via `agy plugin install /path/to/swarm-router`.)*

## How It Stays Zero-Bloat Until `/swarm` Is Used

- **`rules/AGENTS.md`** *(always on, 2 bullets)*: Read in every conversation — prefers **not** to use `/swarm` for simple tasks (since swarms are expensive), but instructs agents **not to hesitate** recommending `/swarm` once a task becomes complex/multi-step or highly parallelizable, and lazy-loads `swarm-router` only when `/swarm` is invoked.
- **`skills/swarm-router/SKILL.md`** *(lazy-loaded on `/swarm`)*: Uses Antigravity progressive disclosure so the full routing matrix, quota balancing policy, thinking-effort ladder, and plan-first workflow are **not read during normal conversations** — they load only when `/swarm` starts (or when you ask to inspect quotas).

## Version-Agnostic Model Routing & Cost Order

Model IDs are resolved dynamically from `swarm{action:"models", all:true}` by matching **family keywords** (preferring the newest available version) and inferring tier/pool from live metadata for any future or renamed models.

Models are ordered from **highest to lowest quota cost** within each tier:

### Smart Tier — Planning, Verification, Coordination, Hard RCA
| Order | Family | Pool | Best For |
| :---: | :--- | :--- | :--- |
| 1 | **Opus / Fable** (`*opus*`, `*fable*`) | 3P / Claude | Hardest architecture plans, critical final verification, or stuck RCAs |
| 2 | **Carbon** (`*carbon*`) | Gemini | Heaviest Gemini flagship for deepest long-horizon planning, complex architecture, and high-stakes verification |
| 3 | **Sonnet** (`*sonnet*`) | 3P / Claude | Default 3P workhorse for planning, verification, and coordination |
| 4 | **Neon** (`*neon*`) | Gemini | Balanced, fast Gemini planner, coordinator, and verifier (best default for most Gemini agentic tasks) |

> **`argon` restriction**: `argon` is **not allowed** unless the other smart Gemini models (`carbon` and `neon`) are unavailable.

### Grunt Tier — Implementation, Tests, Refactors, Research, Docs
| Order | Family | Pool | Best For |
| :---: | :--- | :--- | :--- |
| 1 | **Hydrogen** (`*hydrogen*`) | Gemini | Non-trivial multi-file coding, tricky bug fixes, complex refactors |
| 2 | **Flash** (`*flash*`) | Gemini | Default for straightforward edits, unit tests, builds, scans, docs, narrator/monitor |

### Custom Models (Local / User-Configured)
If any custom models are available in `swarm{action:"models", all:true}`:
- **Think outside the box**: Use them for creative brainstorming, fresh angles, or breaking out of stuck loops.
- **Refusal fallback**: Try them when standard hosted models refuse a legitimate user request.
- **Quota-free**: Custom/local models do not burn shared Gemini or 3P cloud quotas.

### Handling Future / Unrecognized Codenames
When a brand-new codename appears in `swarm{action:"models", all:true}`:
1. **Live tier anchors**: Models mapped to `flash_lite`/`flash` in `tiers` route to **Grunt**; models mapped to `pro` route to **Smart**.
2. **Safe quota default**: Unrecognized 3P codenames or models listed above grunt families default to **Smart** (never placed on high-volume grunt fan-outs by accident); `flash`/`lite`/`mini`/`haiku`/`oss` models route to **Grunt**; local/user-configured models route to **Custom**.
3. **Ask once**: If ambiguous, the coordinator asks you in 1 line where the new model fits before routing heavy work to it.

## Thinking Effort Scaling

Effort variants (`-low`, `-medium`, `-high`) are matched dynamically from each model's available IDs:

| Effort | When to Use |
| :--- | :--- |
| **`-low`** | Mechanical / trivial tasks: formatting, renames, boilerplate, simple docs, file/log scans, narrator, monitor. |
| **`-medium`** | **Default for most tasks**: standard implementation, test writing, refactoring, normal planning, routine verification, coordination. |
| **`-high`** | **Maximum cap**: extremely difficult tasks only — subtle concurrency/state bugs, hard RCA after `-medium` fails, or critical architecture/security reviews. |
| **`-xhigh` / `-max`** | **Avoid** unless explicitly requested — steep quota cost for marginal gain. |

## Quota Efficiency Recommendations (Shared Multi-Machine Setup)

When multiple machines share the same Antigravity account, they draw from the same rolling short-term (`5h`) and `weekly` quota windows across pools.

1. **Run a cheap main model by default**: Keep everyday chats on Flash or Hydrogen (`-medium`). Let the tiny always-on rule nudge you to `/swarm` when a task genuinely needs a smarter model; once `/swarm` starts, the coordinator can either upgrade itself (`swarm{action:"model"}`) or delegate planning/verification to smart workers.
2. **Always check live headroom first**: Run `swarm{action:"models", all:true}` before spawning a swarm (another machine may have recently used the short-term window).
3. **Actively balance hourly (`5h`) AND `weekly` quotas across pools**: Check both the short-term (hourly/`5h`) and `weekly` windows for every pool; the tighter window governs headroom. Whenever one pool is burning faster or lower on either window, shift eligible roles to run down the fuller pool (route smart roles to **3P** `sonnet`/`opus`/`fable` when Gemini is tighter from grunt work; shift smart roles to **Gemini** `neon`/`carbon` when 3P's hourly or weekly window is tighter; `argon` only if both are unavailable). Re-check live quotas before spawning new waves.
4. **Plan first on complex tasks**: One smart `planner` turn (`workspace: "inherit"`) locking interfaces and disjoint file ownership is far cheaper than 3 grunt workers colliding and retrying.
5. **Escalate lazily, cap retry loops**: Start routine work on `flash-medium`, upgrade to `hydrogen-medium` if it struggles, and escalate to a smart model for RCA after 2 failed rounds instead of burning retries.
6. **Keep swarms lean and short-lived**:
   - Aim for 2–4 parallel grunt authors + 1 smart verifier (`workspace: "inherit"`).
   - Keep briefs <=5 lines and reference earlier posts by `#seq` instead of pasting logs.
   - Kill finished `share` workers immediately after verification (land changes first), and rotate long-running workers around ~45 turns to prevent >80k-token context bloat.
7. **Respect 3P prompt cache TTL**: Use `opus`/`sonnet` in tight bursts (plan -> handoff, or verify -> verdict) rather than leaving them idle >5 minutes and resuming with large context.
8. **Switch instead of killing on quota exhaustion**: Use `swarm{action:"model", name, model}` to move a blocked worker to the healthiest pool without losing its conversation state.
9. **Pace the weekly window**: If `weekly` remaining % is dropping faster than the fraction of the week left before `resets`, downshift one tier across all roles (`opus/fable -> sonnet`, `carbon -> neon`, `hydrogen -> flash`) and reduce parallel worker count.

## License

MIT
