<p align="center">
  <img src="assets/logo.svg" alt="Swarm Router logo" width="120" height="120" />
</p>

# Swarm Router (`v2.0.0`)

Live quota-aware, **100% model-agnostic** routing plugin for `/swarm` workers and `invoke_subagent` calls.

Instead of hardcoding model or vendor names that go stale across releases and environments, `swarm-router` inspects live `models` metrics (`cost`, `quota_pool`, `pool_size`, short-window and weekly `used`/`remaining`, and `resets` / `resets_in`) right before spawning any worker or subagent.

## Architecture

1. **Always-read trigger (`rules/AGENTS.md`, <= 25 lines)**
   - Encourages `invoke_subagent` for bounded delegation and proactive `/swarm` orchestration for parallelizable (`>=3` slices) or long-horizon tasks.
   - Instructs the main agent to lazy-load `skills/swarm-router/SKILL.md` and inspect live `models` metrics right before spawning any subagent or swarm worker.
2. **Lazy-loaded routing manual (`skills/swarm-router/SKILL.md`)**
   - **Three-tier routing**:
     - **Grunt tier** *(lowest `cost`, zero-cost/`custom`, or largest / request-based `pool_size`)*: planned, clear-directive, verification-defined work — scouting, multi-file reading/searching, log triage, scoped code edits, tests, builds, and docs.
     - **Smart tier** *(mid-tier `cost` with strong reasoning and healthy pools)*: planning & interface locking, debugging/RCA, cross-module synthesis, and independent review/verification.
     - **Escalation tier** *(highest `cost` or smallest/tightest `pool_size`)*: strictly when a task proves too difficult for Smart/Grunt or iterates `>2` rounds without converging.
   - **Pool balancing math**:
     - **Relative burn (`cost / pool_size`)**: weighs burn as a fraction of pool capacity and favors request-based or large pools for high-turn Grunt loops.
     - **Weekly-pool protection**: routes by `min(short_remaining%, weekly_remaining%)` so scarce weekly caps are never wasted even when a short window reads `100%`.
     - **Reset-time arbitrage**: spends expiring short-window capacity (`resets_in`) first when weekly headroom is healthy, and avoids draining long-reset pools before heavy waves.
     - **In-place switching**: uses `swarm{action:"model"}` to change models without losing worker conversation history, caps author/verifier loops at 2 rounds, and kills temporary/expensive workers promptly on `[done]`.

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
