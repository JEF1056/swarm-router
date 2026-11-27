# Swarm & Subagent Router

Use live quota metrics instead of guessing models. Keep normal direct work inline (`1-2` trivial calls).

- **Use subagents (`invoke_subagent`)** for bounded delegation: multi-file discovery/reading, isolated searches, single-scope edits/tests, or independent checks — never inherit an expensive parent model for cheap grunt work.
- **Use `/swarm` proactively** (even unprompted) whenever a task splits into `>=3` parallel slices, spans multiple modules, needs long-horizon coordination, or benefits from independent author/verifier loops.
- **Lazy-load before spawning**: Right before calling `invoke_subagent` or `swarm{action:"add"}`, read the `swarm-router` skill (`SKILL.md`) AND inspect live `models` metrics (`manage_subagents{Action:"models", All:true}` for regular subagent spawns, or `swarm{action:"models", all:true}` when in a swarm) to route by `cost`, `quota_pool`, `pool_size`, short-window & weekly `used`/`remaining`, and `resets` time.
