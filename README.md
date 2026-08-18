# google-adk-v2-golang

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) for building AI agents with Google's [Agent Development Kit for Go, v2](https://adk.dev/get-started/go/) (`google.golang.org/adk/v2`).

Claude Code skills are packaged instructions Claude loads on demand — this one teaches Claude how to write correct ADK Go v2 code instead of relying on (often stale or v1-only) training data. Drop this directory into a project as a skill, and Claude will pick it up automatically whenever a task involves ADK Go: defining agents and tools, orchestrating multi-agent pipelines, the new graph-based `workflow` engine, durable human-in-the-loop, MCP/Agent Registry integration, remote A2A agents, or migrating an existing v1 codebase to v2.

## Why this exists

ADK Go v2 introduced several breaking changes from v1 (`agent.ToolContext`/`agent.CallbackContext` merged into a unified `agent.Context`, a new `/v2` import path, `session.NewEvent` gaining a `context.Context` parameter) plus substantial new surface area with no v1 equivalent — most notably the `workflow` graph-orchestration engine. A model's pretraining data is likely to know v1 (or nothing) and will confidently generate code that no longer compiles. This skill was built by researching the actual v2.2.0 release (GA'd 2026-06-30) against official docs and the `google/adk-go` source, not by assuming version continuity.

## Layout

```
SKILL.md                       Entry point: overview, key imports, core concepts, gotchas, migration guide
references/
  api-reference.md             Full type reference (agent.Context, llmagent.Config, session.Event, ...)
  orchestration.md             Fixed workflow agents: sequential/parallel/loop pipelines, critic/refiner loops, delegation
  workflow.md                  The v2 graph engine: Node/Edge, DynamicNode+RunNode, fan-out/fan-in, durable HITL
  integrations.md              MCP toolsets, Agent Registry, auth, OpenAI/custom model providers
evals/
  evals.json                   Benchmark prompts + assertions used to test and iterate on the skill
```

`SKILL.md` stays under Claude's context budget by keeping only the always-relevant material inline and pointing into `references/*.md` for the rest — those load only when the task actually needs them.

## Using it

Copy or symlink this directory into a project's skills directory (e.g. `.claude/skills/google-adk-v2-golang/` — see the [Claude Code skills docs](https://docs.claude.com/en/docs/claude-code/skills) for supported locations). Claude triggers it automatically on ADK Go–shaped requests; no manual invocation needed.

## Status

Built and iterated with the `skill-creator` skill's eval loop: task prompts run once with the skill and once without (baseline), graded against explicit per-eval assertions, and compared. Current benchmark: **100% pass rate with the skill vs. ~30–40% without**, across three iterations that fixed real defects surfaced by the evals (a `context.Context` vs. `agent.Context`/`agent.InvocationContext` type mismatch, and an unimplementable `Workflow.Resume` call-site pattern that was corrected to the actual public resume contract — resuming via `runner.Run` with a `FunctionResponse`, verified against the ADK Go source).

Verified against ADK Go **v2.2.0**. If the ADK Go version pinned in your `go.mod` differs from that, treat version-specific details here as potentially stale and re-check `pkg.go.dev/google.golang.org/adk/v2` and `https://github.com/google/adk-go` before trusting them.

## License

[MIT](LICENSE)
