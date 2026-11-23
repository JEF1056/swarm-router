---
name: swarm-quota
description: Inspect live Gemini and 3P (Claude) quota windows, design a cost-optimal /swarm model roster and effort plan for a task, or advise on multi-machine quota pacing.
---

# Swarm Quota Planner & Efficiency Guide

Use this skill when the user asks to check quota headroom, plan a cost-optimal `/swarm` roster, or optimize shared quota usage across machines.

## Step 1: Inspect Live Quota Windows

- If already in an active swarm (or user invoked `/swarm`), call `swarm{action:"models", all:true}`.
- Otherwise, call `manage_subagents{Action:"models", All:true}` (never call `swarm` unprompted).

Extract from `quota_windows`:
- `gemini-5h` and `gemini-weekly` (with reset times)
- `3p-5h` and `3p-weekly` (with reset times)

## Step 2: Recommend Roster by Role, Difficulty, and Headroom

Classify the task first:
- **Simple (1–2 files, clear spec)**: Recommend skipping `/swarm` — handle directly or with 1 subagent (`flash-medium` or `hydrogen-medium`).
- **Moderate (2–4 independent slices)**: Skip separate planner; spawn grunt authors + 1 smart verifier.
- **Complex (multi-system, shared interfaces, >3 slices, or high risk)**: **Plan first** with 1 smart `planner` (`workspace: "inherit"`), lock interfaces/file ownership, then spawn grunt authors (`workspace: "share"`) + smart `verifier` (`workspace: "inherit"`).

Pick models using the live pool balance:

| Role | Pool Preference | Normal Difficulty | Hard / Critical | Mechanical / Light |
| :--- | :--- | :--- | :--- | :--- |
| **Coordinator** | Healthier pool (`3p` if equal) | `claude-sonnet-5-5-medium` or `gemini-4-carbon-medium` | `gemini-4-neon-medium` or `claude-opus-5-5-medium` | `gemini-4-argon-medium` |
| **Planner** (complex tasks) | Healthier pool (`3p` if equal) | `claude-sonnet-5-5-medium` or `gemini-4-carbon-medium` | `claude-opus-5-5-high` or `gemini-4-neon-high` | — |
| **Verifier** (1 per 2–4 authors) | Healthier pool (`3p` if equal) | `claude-sonnet-5-5-medium` or `gemini-4-argon-medium` | `gemini-4-carbon-high` or `claude-opus-5-5-medium` | `gemini-4-argon-low` |
| **Author — hard coding/debug** | `gemini` | `gemini-4-hydrogen-medium` | `gemini-4-hydrogen-high` | `gemini-4-hydrogen-low` |
| **Author — routine/tests/docs** | `gemini` | `gemini-3.8-flash-medium` | `gemini-4-hydrogen-medium` | `gemini-3.8-flash-low` |
| **Research / Scans / Narrator** | `gemini` | `gemini-3.8-flash-low` | `gemini-3.8-flash-medium` | `gemini-3.8-flash-low` |

## Step 3: Multi-Machine Shared Quota Guardrails

Remind or apply these habits whenever multiple machines share the same account:
1. **Check before fanning out**: Another machine may have just drained `5h` quota; always inspect `quota_windows` before adding workers.
2. **Right-size concurrency**: Prefer 2–4 focused workers over 8+ overlapping ones; every broadcast `@all` wakes every worker and multiplies token burn.
3. **Keep briefs short & self-contained**: Inline exact file paths and done criteria (`<=5` lines); reference earlier posts by `#seq` instead of copy-pasting logs.
4. **Kill finished workers immediately**: Waiting workers are idle, but long-lived workers accumulate huge context (>80k tokens) if reused across unrelated waves — rotate around ~45 turns.
5. **Respect 3P prompt cache windows**: Avoid leaving active Claude (`opus`/`sonnet`) workers idle for >5 minutes and then resuming them with massive context; finish their verification/planning pass in one tight burst and kill them.
6. **Never use `-xhigh`**: Cap thinking effort at `-high` for genuinely extreme problems; `-medium` handles ~90% of work at substantially lower quota cost.
7. **Pacing check**: Compare `remaining %` on `weekly` against time left until `resets`; if burn rate exceeds steady-state pace, downshift one tier across all roles.
