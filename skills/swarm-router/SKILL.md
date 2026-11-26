---
name: swarm-router
description: >-
  Gemini-first swarm model routing, 5h/weekly quota pacing, thinking-effort
  scaling, and token-lean execution. Read ONLY before swarm{action:"add"},
  checking quotas, or designing a roster.
---

# Swarm Router (Gemini-First, Token-Lean)

Overrides "leave `model` off" unless the user specifies a model. Always pass explicit `model` on `swarm{action:"add"}`.

## 1. Live Discovery

Run `swarm{action:"models", all:true}` (`manage_subagents{Action:"models", All:true}` outside a swarm) before spawning and before new waves:
- Never hardcode versions — pick the newest matching family ID from live `models`.
- Check **both** `5h` and `weekly` in `quota_windows` (shared across machines); `headroom = min(5h, weekly)`.

## 2. Tiers & Routing (Prefer Gemini; Claude Only When Really Hard)

- **Smart** *(Planning, Audits, Reviews, Verification, Coordination, Hard RCA)*:
  - **Gemini (default)**:
    1. `neon` — default planner, coordinator, auditor, verifier.
    2. `carbon` — deep multi-module architecture, complex plans, high-stakes audits/verification.
    3. `argon` — **only** if both `neon` and `carbon` are unavailable/exhausted.
  - **3P Claude (escalation only — scarce, much smaller weekly pool)**:
    - **Never default routine planning, coordination, verification, or audits to Claude**, even when 3P % >= Gemini %.
    - `opus` / `fable` — **really hard tasks only** (gnarly architecture, subtle concurrency/security bugs, stuck RCA after 2 failed Gemini rounds, critical final gate). Max 1–2 short bursts per swarm.
    - `sonnet` — **only** if Gemini Smart headroom `<15%` or a stuck RCA / 2nd opinion needs a non-Gemini model.
- **Grunt** *(Implementation, Tests, Refactors, Research, Docs)*:
  1. `flash` *(default; `flash-lite` for trivial scans/boilerplate)* — edits, tests, builds, scans, docs.
  2. `hydrogen` — multi-file coding, tricky fixes, complex refactors, or after 1 failed `flash` attempt.
- **Custom / Local**: Zero cloud quota — use for lateral ideas, refusal fallback, or quota-free work (keep local concurrency low).
- **Unknown codenames**: Map via `tiers` (`flash_lite`/`flash` -> Grunt, `pro` -> Smart) and name cues (`flash|lite|mini|haiku|oss` -> Grunt; local -> Custom; unknown 3P or above grunt -> Smart, never bulk fan-out). Ignore `*-bon-arm`/`*-bon-le`. Ask user in 1 line if ambiguous.

## 3. Quota Pacing & Effort

- **Gemini default**: Stay on Gemini (`flash`/`hydrogen` + `neon`/`carbon`) while Gemini `headroom >= 20%`. Spill Smart roles to `sonnet` only if Gemini `<20%` and 3P `>=30%`; never spill Grunt to Claude.
- **Tight pools**: If Gemini `<30%` or weekly % drops faster than elapsed week, downshift (`carbon -> neon`, `hydrogen -> flash`, `-high -> -medium`), cut concurrency, skip speculative fan-outs, and use custom/local models.
- **On exhaustion**: Switch in-place via `swarm{action:"model", name, model}` to the healthiest same-tier pool — never `kill` + `add`.
- **Effort suffix** (match available `-low`/`-medium`/`-high`, else base ID): `-low` mechanical/trivial (renames, formatting, scans, docs); `-medium` *(default)* everything standard; `-high` *(hard cap)* really hard bugs/architecture or stuck RCA after `-medium` fails. Never `-xhigh`/`-max` unless requested.

## 4. Plan-First & Token Discipline

- **Startup**: On standalone Flash/Hydrogen mains, upgrade coordinator (`swarm{action:"model"}` -> `neon-medium`) or delegate planning/verification to Gemini Smart workers. **Orchestrator coordinators (`neon-medium + hydrogen-medium`) are already Tier 2 — do NOT switch at startup.**
- **Plan first** on complex tasks: 1 Smart `planner` (`workspace: "inherit"`) locks interfaces, disjoint file ownership, and acceptance checks before spawning grunt workers.
- **Lean execution**:
  - **2–4** grunt authors (`workspace: "share"`) per **1** Smart verifier (`workspace: "inherit"`). Avoid `@all`.
  - Briefs **`<=5` lines** (exact paths + done criteria); reference posts by `#seq` instead of pasting logs/diffs.
  - Cap author<->verifier loops at **2 rounds**, then rotate or escalate (`flash -> hydrogen -> neon/carbon -> Claude`).
  - Rotate at **~45 turns** (`<=35` on Claude). Land work and **kill finished `share` and Claude workers immediately** (5m KV TTL; never idle Claude coordinators or fan out on Claude).
