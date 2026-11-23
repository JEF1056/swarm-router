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

- **`rules/AGENTS.md`** *(always on, 2 bullets)*: Tells lightweight main models (Flash / Hydrogen) when to suggest `/swarm` for complex tasks, and instructs the agent to load the `swarm-router` skill **only** when `/swarm` is invoked.
- **`skills/swarm-router/SKILL.md`** *(lazy-loaded on `/swarm`)*: Uses Antigravity progressive disclosure so the full routing matrix, quota balancing policy, thinking-effort ladder, and plan-first workflow are **not read during normal conversations** — they load only when `/swarm` starts (or when you ask to inspect quotas).

## Version-Agnostic Model Routing & Cost Order

Model IDs are resolved dynamically from `swarm{action:"models", all:true}` by matching **family keywords** (preferring the newest available version) and inferring tier/pool from live metadata for any future or renamed models.

Models are ordered from **highest to lowest quota cost** within each tier:

### Smart Tier — Planning, Verification, Coordination, Hard RCA
| Order | Family | Pool | Best For |
| :---: | :--- | :--- | :--- |
| 1 | **Opus** (`*opus*`) | 3P / Claude | Hardest architecture plans, critical final verification, stuck RCAs |
| 2 | **Neon** (`*neon*`) | Gemini | Complex Gemini planning, coordination, high-stakes verification |
| 3 | **Sonnet** (`*sonnet*`) | 3P / Claude | Default 3P planner, verifier, and coordinator |
| 4 | **Carbon** (`*carbon*`) | Gemini | Balanced Gemini planner and verifier |
| 5 | **Argon** (`*argon*`) | Gemini | Most quota-efficient smart model for routine verification and triage |

### Grunt Tier — Implementation, Tests, Refactors, Research, Docs
| Order | Family | Pool | Best For |
| :---: | :--- | :--- | :--- |
| 1 | **Hydrogen** (`*hydrogen*`) | Gemini | Non-trivial multi-file coding, tricky bug fixes, complex refactors |
| 2 | **Flash** (`*flash*`) | Gemini | Default for straightforward edits, unit tests, builds, scans, docs, narrator/monitor |

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
3. **Offload smart roles to 3P when headroom allows**: Since grunt workers (`hydrogen`, `flash`) consume Gemini quota, routing planning/verification/coordination to `sonnet` (or `opus` for critical work) when 3P headroom >= Gemini headroom keeps both pools balanced. Shift smart roles to `argon`/`carbon`/`neon` whenever 3P drops below Gemini or under 25%.
4. **Plan first on complex tasks**: One smart `planner` turn (`workspace: "inherit"`) locking interfaces and disjoint file ownership is far cheaper than 3 grunt workers colliding and retrying.
5. **Escalate lazily, cap retry loops**: Start routine work on `flash-medium`, upgrade to `hydrogen-medium` if it struggles, and escalate to a smart model for RCA after 2 failed rounds instead of burning retries.
6. **Keep swarms lean and short-lived**:
   - Aim for 2–4 parallel grunt authors + 1 smart verifier (`workspace: "inherit"`).
   - Keep briefs <=5 lines and reference earlier posts by `#seq` instead of pasting logs.
   - Kill finished `share` workers immediately after verification (land changes first), and rotate long-running workers around ~45 turns to prevent >80k-token context bloat.
7. **Respect 3P prompt cache TTL**: Use `opus`/`sonnet` in tight bursts (plan -> handoff, or verify -> verdict) rather than leaving them idle >5 minutes and resuming with large context.
8. **Switch instead of killing on quota exhaustion**: Use `swarm{action:"model", name, model}` to move a blocked worker to the healthiest pool without losing its conversation state.
9. **Pace the weekly window**: If `weekly` remaining % is dropping faster than the fraction of the week left before `resets`, downshift one tier across all roles (`opus -> sonnet`, `neon -> carbon -> argon`, `hydrogen -> flash`) and reduce parallel worker count.

## License

MIT
