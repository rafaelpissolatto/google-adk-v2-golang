# google-adk-v2-golang

An [Agent Skill](https://www.skills.sh/) for building AI agents with Google's [Agent Development Kit for Go, v2](https://adk.dev/get-started/go/) (`google.golang.org/adk/v2`).

## What is a skill?

A skill is a packaged, on-demand instruction set — a `SKILL.md` file plus reference docs — that a coding agent loads into context only when a task actually needs it, instead of relying on the agent's (often stale or v1-only) training data. This skill teaches an agent how to write correct ADK Go v2 code: defining agents and tools, orchestrating multi-agent pipelines, the new graph-based `workflow` engine, durable human-in-the-loop, MCP/Agent Registry integration, remote A2A agents, and migrating an existing v1 codebase to v2.

It works with any coding agent that supports the emerging Agent Skills format — Claude Code, OpenCode, Cursor, Kiro CLI, and others — since `SKILL.md` itself contains no tool-specific instructions.

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

`SKILL.md` stays lean by keeping only the always-relevant material inline and pointing into `references/*.md` for the rest — those load only when the task actually needs them.

## Install (30-second setup)

**Any agent, via `npx skills`** (writes editable files into your repo; pull updates manually with `npx skills update`):

```bash
npx skills@latest add rafaelpissolatto/google-adk-v2-golang
```

Pick this skill from the list, along with your target agent(s) — the tool wires it into the right location for each one automatically.

<details>
<summary>Requirements</summary>

`npx skills` needs [Node.js](https://nodejs.org/) 18+ (bundles npm) on your `PATH`. No Node toolchain? Skip straight to the manual copy/symlink table below — it has no dependencies.

</details>

Per-agent locations, if you'd rather copy or symlink the directory by hand:

| Agent | Skill directory |
|---|---|
| [Claude Code](https://docs.claude.com/en/docs/claude-code/skills) | `.claude/skills/google-adk-v2-golang/` |
| [OpenCode](https://opencode.ai/) | `.opencode/skills/google-adk-v2-golang/` |
| [Cursor](https://cursor.com/) | `.agents/skills/google-adk-v2-golang/` |
| [Kiro CLI](https://kiro.dev/) | `.kiro/skills/google-adk-v2-golang/` |

Once in place, most agents surface it automatically; some (e.g. OpenCode) list it as an available skill the agent chooses to load on ADK Go–shaped requests.

## Status

Built and iterated with the `skill-creator` skill's eval loop: task prompts run once with the skill and once without (baseline), graded against explicit per-eval assertions, and compared. Current benchmark: **100% pass rate with the skill vs. ~30–40% without**, across three iterations that fixed real defects surfaced by the evals (a `context.Context` vs. `agent.Context`/`agent.InvocationContext` type mismatch, and an unimplementable `Workflow.Resume` call-site pattern that was corrected to the actual public resume contract — resuming via `runner.Run` with a `FunctionResponse`, verified against the ADK Go source).

Verified against ADK Go **v2.2.0**. If the ADK Go version pinned in your `go.mod` differs from that, treat version-specific details here as potentially stale and re-check `pkg.go.dev/google.golang.org/adk/v2` and `https://github.com/google/adk-go` before trusting them.

## License

[MIT](LICENSE)
