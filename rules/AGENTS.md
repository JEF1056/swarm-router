# Swarm Escalation & Lazy Routing

- **When to `/swarm`**: Skip swarms for simple tasks. Start a swarm directly (even unprompted) for complex/multi-module work, `>3` parallel slices, subtle concurrency/security bugs, or when Flash/Hydrogen stalls — Gemini Smart plans/verifies, Grunt executes.
- **Lazy-load routing**: Read `swarm-router` (`SKILL.md`) ONLY before calling `swarm{action:"add"}`, checking quotas, or designing a roster; never read it in normal non-swarm chats. Default to Gemini; reserve Claude for genuinely hard tasks.
