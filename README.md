# Swarm Router (Antigravity Plugin)

A token-lean Antigravity plugin for **Gemini-first** `/swarm` model routing, shared `5h`/`weekly` quota pacing, thinking-effort scaling, and plan-first execution — reserving scarce **3P Claude** quota for genuinely hard tasks.

## Install

Tell your Antigravity agent:
> Install the `swarm-router` plugin from `https://github.com/JEF1056/swarm-router`

Or install from a shell:

```bash
git clone https://github.com/JEF1056/swarm-router.git ~/.gemini/config/plugins/swarm-router
agy plugin validate ~/.gemini/config/plugins/swarm-router --strict
agy plugin list
```

Update later:

```bash
git -C ~/.gemini/config/plugins/swarm-router pull
```

## Zero-Bloat Progressive Disclosure

- **`rules/AGENTS.md`** *(always on, ~115 tokens)*: Keeps simple chats off `/swarm`, allows unprompted swarms for complex or `>3`-slice parallel tasks, and lazy-loads `SKILL.md` only when spawning or checking quotas.
- **`skills/swarm-router/SKILL.md`** *(lazy-loaded, ~1k tokens)*: Full Gemini-first routing, `5h`/`weekly` quota pacing, effort scaling, and token-discipline rules loaded only when needed.

## Model Routing (Gemini Default, Claude Only When Really Hard)

Model IDs are resolved dynamically from `swarm{action:"models", all:true}` by family keyword (newest version first).

### Smart Tier — Planning, Audits, Reviews, Verification, Coordination, Hard RCA
| Priority | Family | Pool | When to Use |
| :---: | :--- | :--- | :--- |
| 1 | **Neon** (`*neon*`) | Gemini | **Default** planner, coordinator, auditor, and verifier |
| 2 | **Carbon** (`*carbon*`) | Gemini | Deep multi-module architecture, complex plans, high-stakes audits/verification |
| 3 | **Argon** (`*argon*`) | Gemini | Fallback **only** if both Neon and Carbon are unavailable/exhausted |
| Escalation | **Opus / Fable** (`*opus*`, `*fable*`) | 3P Claude | **Really hard tasks only**: gnarly architecture, subtle concurrency/security bugs, stuck RCA after 2 failed Gemini rounds, critical final gate (1–2 short bursts; kill on `[done]`) |
| Fallback | **Sonnet** (`*sonnet*`) | 3P Claude | Only when Gemini Smart is exhausted (`<15%` bottleneck) or a 2nd opinion / stuck RCA needs a non-Gemini model |

### Grunt Tier — Implementation, Tests, Refactors, Research, Docs
| Priority | Family | Pool | When to Use |
| :---: | :--- | :--- | :--- |
| 1 | **Flash** (`*flash*`, `*flash-lite*`) | Gemini | **Default** for edits, unit tests, builds, scans, docs (`flash-lite` for trivial scans) |
| 2 | **Hydrogen** (`*hydrogen*`) | Gemini | Non-trivial multi-file coding, tricky fixes, complex refactors, or after 1 failed Flash attempt |

### Custom / Local & Unknown Models
- **Custom / Local**: Zero cloud quota — use for lateral ideas, refusal fallback, or quota-free tasks (low local concurrency).
- **Unknown codenames**: Inferred from `tiers` (`flash_lite`/`flash` -> Grunt, `pro` -> Smart) and name cues (`flash|lite|mini|haiku|oss` -> Grunt; unknown 3P or above grunt -> Smart, never bulk fan-out). Experiment arms (`*-bon-arm`/`*-bon-le`) are ignored.

## Thinking Effort Scaling

| Effort | When to Use |
| :--- | :--- |
| **`-low`** | Mechanical / trivial tasks: renames, formatting, scans, simple docs |
| **`-medium`** | **Default**: coding, tests, refactors, planning, audits, reviews, verification, coordination |
| **`-high`** | **Hard cap**: really hard bugs/architecture/security or stuck RCA after `-medium` fails |
| **`-xhigh` / `-max`** | Avoid unless explicitly requested |

## Shared-Quota & Token Guardrails

1. **Stay on Gemini by default**: Keep all Grunt and routine Smart roles on Gemini whenever `min(5h, weekly) >= 20%`, even if 3P % looks higher. Spill Smart roles to 3P (`sonnet`) only when Gemini `<20%` and 3P `>=30%`. Never spill Grunt work onto Claude.
2. **Plan first**: One Gemini Smart `planner` (`workspace: "inherit"`) locking interfaces and disjoint file ownership avoids multi-worker churn.
3. **Stay lean**: 2–4 `share` grunt authors per 1 `inherit` Smart verifier; briefs `<=5` lines; reference `#seq` instead of pasting logs; avoid `@all`; cap author<->verifier loops at 2 rounds; rotate around ~45 turns (`<=35` on Claude).
4. **Kill Claude & `share` workers immediately**: Respect the 5-minute 3P KV cache TTL — kill Claude workers right on `[done]` and never run long-idling coordinators or parallel fan-outs on Claude.
5. **Switch instead of killing on quota exhaustion**: Use `swarm{action:"model", name, model}` to preserve worker context.

## License

MIT
