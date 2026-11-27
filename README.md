<p align="center">
  <img src="assets/logo.svg" alt="Swarm Router logo" width="120" height="120" />
</p>

# Swarm Router (`v2.1.1`)

Live quota-aware, **100% model-agnostic** routing plugin for `/swarm` workers and `invoke_subagent` calls.

Instead of hardcoding model or vendor names or mandating one fixed swarm pipeline, `swarm-router` inspects live `models` metrics (`cost`, `quota_pool`, `pool_size`, short-window and weekly `used`/`remaining`, and `resets` / `resets_in`) right before spawning and chooses both the **topology** and **model tier** for lowest cost and highest first-pass quality.

## Architecture

1. **Always-read trigger (`rules/AGENTS.md`, `<= 25` lines)**
   - Routes trivial work inline, single bounded slices to `invoke_subagent`, and multi-slice / long-horizon work proactively to `/swarm`.
   - Lazy-loads `skills/swarm-router/SKILL.md` and live `models` metrics right before spawning.
2. **Lazy-loaded routing manual (`skills/swarm-router/SKILL.md`)**
   - **Four-tier routing**:
     - **Basic** *(ultra-cheap / `-lite`, `-mini`, `-nano`, `-small`, lowest `cost`, or huge lite pools)*: strictly zero-thinking, clear-cut actions (exact-spec edits, formatting, mechanical renames, command execution, literal extraction). Never used for reasoning, synthesis, or bug diagnosis.
     - **Grunt** *(low-`cost`, `0`/`custom`, or large / request-metered capable workhorses)*: clear-directive engineering (scouting, multi-file exploration, log triage, scoped edits, tests, builds, docs, structured refactors).
     - **Smart** *(mid-tier `cost` strong reasoners with healthy headroom)*: non-trivial planning & interface locking, ambiguous/cross-module RCA, complex single-owner edits, and independent review/verification.
     - **Escalation** *(highest `cost`, tightest `pool_size`, or max effort)*: strictly tasks too hard for Smart or stuck past 2 rounds.
   - **Expected burn & pool math**:
     - Weighs task difficulty, expected burn (`turns * context * workers`), relative pool cost (`cost / pool_size`), weekly-pool protection (`min(short_remaining%, weekly_remaining%)`), expiring-window arbitrage (`resets_in`), and effort suffixes (`-low`/`-medium`/`-high`/`-max`).
   - **Adaptive topology**:
     - Composes only the stages the task earns — always scout when uncertain rather than guessing, plan only when complexity or shared contracts warrant it, verify proportionally to risk (occasionally using a different model family within balanced pool budgets), and switch models in-place (`swarm{action:"model"}` / `manage_subagents{Action:"model"}`) instead of repeating failing loops.

## Install

```bash
git clone https://github.com/JEF1056/swarm-router.git swarm-router
```

Copy or symlink `swarm-router` into your agent harness plugins directory, then validate:

```bash
agy plugin validate ./swarm-router
```

## Layout

```text
swarm-router/
├── plugin.json
├── README.md
├── assets/
│   └── logo.svg
├── rules/
│   └── AGENTS.md
└── skills/
    └── swarm-router/
        └── SKILL.md
```
