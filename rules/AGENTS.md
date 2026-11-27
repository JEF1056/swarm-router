# Swarm & Subagent Router

Match execution shape and model tier to task difficulty, expected burn, and live quota — never guess models or mandate a fixed topology.

- **Stay inline** for `1-2` trivial calls.
- **Use `invoke_subagent`** for one bounded slice (noisy discovery, isolated edit/test, or independent check).
- **Use `/swarm`** (even unprompted) when work benefits from concurrency (`>=3` independent slices), cross-module coordination, or live author/verifier iteration.
- **Before spawning** (`invoke_subagent`, `swarm{action:"add"}`, `swarm{action:"model"}`, or `manage_subagents{Action:"model"}`), read the `swarm-router` skill (`SKILL.md`) and inspect live models (`manage_subagents{Action:"models", All:true}`, or `swarm{action:"models", all:true}` when a swarm is active). Never leave `inherit` on an expensive parent for cheaper delegated work.
