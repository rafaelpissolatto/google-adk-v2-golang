---
name: google-adk-v2-golang
description: Guides development of AI agents using Google's Agent Development Kit
  (ADK) for Go v2 (`google.golang.org/adk/v2`). Use when creating agents, defining
  tools, orchestrating multi-agent workflows — including the new graph-based
  `workflow` engine for deterministic/dynamic orchestration and durable
  human-in-the-loop — integrating MCP servers or the Cloud Agent Registry,
  connecting remote A2A agents, or building agentic applications with ADK Go.
  Also use when migrating an existing ADK Go v1.x codebase to v2.
---

# google-adk-v2-golang

Google's Agent Development Kit for Go (`google.golang.org/adk/v2`) is a code-first toolkit for building AI agents. It is optimized for Gemini and model-agnostic via the `model.LLM` interface. Documented floor is Go 1.25+; the module's own toolchain directive on `main` is newer (1.26.6) — a consuming project's `go.mod` only needs to declare 1.25.

> **Verified against ADK Go v2.2.0** (v2 GA'd 2026-06-30; v2.1.0 ~2026-07-23; v2.2.0 ~2026-08-10). Official docs: https://adk.dev/get-started/go/. Source: https://github.com/google/adk-go.
>
> **v1 is not dead.** Google maintains v1 (`google.golang.org/adk`, no `/v2`) on a parallel branch — v1.5.1/v1.6.0 shipped in the same window as v2. If you're touching an existing v1 codebase, don't assume it needs porting; only migrate when you actually want v2's new capabilities (the graph `workflow` engine, unified `agent.Context`, built-in HITL, OpenAI backend, Agent Registry). See **Migrating from v1** below.
>
> Several "recalled" details from pre-v2 training data will be wrong: the import path lost the version suffix, `ToolContext`/`CallbackContext` no longer exist as separate types, and `session.NewEvent` gained a leading `context.Context` parameter. Pin the exact ADK Go version in `go.mod` and re-check `pkg.go.dev/google.golang.org/adk/v2` for anything this skill doesn't cover — several sub-package signatures (`functiontool`, `telemetry`, `session/database`, `artifact`, `memory`) were not independently re-verified against v1 in this skill's research pass; they're documented here as carried forward from v1 because no evidence of a change surfaced, not because a v2 diff was directly confirmed.

```bash
go get google.golang.org/adk/v2
```

## Key Imports

```go
import (
    "google.golang.org/adk/v2/agent"                                  // Core agent interface + unified agent.Context
    "google.golang.org/adk/v2/agent/llmagent"                         // LLM-powered agents
    remoteagent "google.golang.org/adk/v2/agent/remoteagent/v2"       // Remote A2A agents (non-/v2 remoteagent package is legacy)
    "google.golang.org/adk/v2/agent/workflowagents/sequentialagent"   // Fixed sequential orchestration (still supported)
    "google.golang.org/adk/v2/agent/workflowagents/parallelagent"     // Fixed parallel orchestration (still supported)
    "google.golang.org/adk/v2/agent/workflowagents/loopagent"         // Fixed loop orchestration (still supported)
    "google.golang.org/adk/v2/workflow"                               // NEW: graph-based orchestration engine
    "google.golang.org/adk/v2/agentregistry"                          // NEW: Google Cloud Agent Registry client
    "google.golang.org/adk/v2/auth"                                   // NEW: credential/auth layer for tools & MCP
    "google.golang.org/adk/v2/model/gemini"                           // Gemini model provider
    "google.golang.org/adk/v2/model/openaimodel"                      // NEW: OpenAI model provider
    "google.golang.org/adk/v2/model/apigee"                           // Gemini via an Apigee proxy
    "google.golang.org/adk/v2/runner"                                 // Agent runtime
    "google.golang.org/adk/v2/session"                                // Session and state
    "google.golang.org/adk/v2/tool"                                   // Tool interface
    "google.golang.org/adk/v2/tool/functiontool"                      // Go functions as tools
    "google.golang.org/adk/v2/tool/agenttool"                         // Agent-as-tool wrapper
    "google.golang.org/adk/v2/tool/mcptoolset"                        // MCP server integration
    "google.golang.org/adk/v2/tool/exitlooptool"                      // Break out of loops
    "google.golang.org/adk/v2/tool/geminitool"                        // Gemini native tools
    "google.golang.org/adk/v2/tool/loadartifactstool"                 // LLM-invoked artifact loading
    "google.golang.org/adk/v2/tool/loadmemorytool"                    // LLM-invoked memory search
    "google.golang.org/adk/v2/tool/preloadmemorytool"                 // Auto-injects memory per request
    "google.golang.org/adk/v2/tool/skilltoolset"                      // Agent Skills (progressive disclosure)
    "google.golang.org/adk/v2/tool/exampletool"                       // Few-shot example injection
    "google.golang.org/adk/v2/memory/vertexai"                        // Vertex AI Memory Bank backend
    "google.golang.org/adk/v2/util/instructionutil"                   // Manual {key} substitution helper
    "google.golang.org/adk/v2/telemetry"                              // OpenTelemetry setup
    "google.golang.org/adk/v2/plugin"                                 // Plugin system
    "google.golang.org/adk/v2/plugin/retryandreflect"                 // Self-healing tool retries
    "google.golang.org/adk/v2/plugin/functioncallmodifier"            // Rewrite tool schemas
    "google.golang.org/adk/v2/plugin/loggingplugin"                   // Console event logger
    "google.golang.org/adk/v2/cmd/launcher"                           // Launcher config
    "google.golang.org/adk/v2/cmd/launcher/full"                      // All launcher modes (dev + prod)
    "google.golang.org/adk/v2/cmd/launcher/prod"                      // Production launcher (no console, no web UI)
    "google.golang.org/genai"                                         // Google GenAI types
)
```

`tool/functiontool`, `tool/agenttool`, `tool/exitlooptool`, `tool/geminitool`, `tool/loadartifactstool`, `tool/loadmemorytool`, `tool/preloadmemorytool`, `tool/skilltoolset`, `tool/exampletool` all still exist under `/v2` at these paths; no signature change was found, but none were byte-diffed against v1 either — treat their usage examples below as reliable in shape, and re-check `pkg.go.dev` if something doesn't compile.

## Creating a Model

```go
// Gemini API (default). The official v2 quickstart uses the rolling alias
// "gemini-flash-latest" rather than a dated model snapshot.
model, err := gemini.NewModel(ctx, "gemini-flash-latest", &genai.ClientConfig{
    APIKey: os.Getenv("GOOGLE_API_KEY"),
})

// Vertex AI
model, err := gemini.NewModel(ctx, "gemini-flash-latest", &genai.ClientConfig{
    Project:  "my-project",
    Location: "us-central1",
    Backend:  genai.BackendVertexAI,
})
```

For Gemini behind an Apigee proxy, use `apigee.NewModel`. For OpenAI, use the new `model/openaimodel` package (added v2.1.0) — see `references/integrations.md`; note it has an open bug where multi-turn conversations 400 due to a content-type mismatch (`input_text` vs `output_text`), so test multi-turn flows before relying on it. There is still no native Anthropic backend as of v2.2.0 (tracked by adk-go#1097); write a custom `model.LLM` for Anthropic or other providers (see `references/integrations.md`).

### Model Registry (new in v2.1.0)

Resolve models by name through registered factories instead of calling `gemini.NewModel` directly everywhere:

```go
import "google.golang.org/adk/v2/model"

model.Register("gemini-*", myGeminiFactory)
m, err := model.NewLLM("gemini-flash-latest")
```

## Creating an Agent

The primary agent type is `llmagent`. It wraps an LLM with instructions, tools, and optional sub-agents.

```go
myAgent, err := llmagent.New(llmagent.Config{
    Name:        "assistant",
    Description: "Helpful coding assistant.",
    Model:       model,
    Instruction: "You are a helpful coding assistant. Help the user write Go code.",
    Tools:       []tool.Tool{myTool},
    GenerateContentConfig: &genai.GenerateContentConfig{
        Temperature: genai.Ptr[float32](0.7),
    },
})
```

### Key llmagent.Config Fields

| Field | Purpose |
|---|---|
| `Name` | Unique name within the agent tree. Cannot be `"user"`. |
| `Description` | One-line description used by parent agents for delegation decisions. |
| `Model` | `model.LLM` implementation (Gemini, OpenAI, custom, etc.) |
| `Instruction` | System prompt. Supports `{state_key}`, `{artifact.key}`, and `{key?}` (optional) substitution. |
| `GlobalInstruction` | Prepended to all sub-agent instructions. Same substitution syntax. |
| `InstructionProvider` | `func(agent.ReadonlyContext) (string, error)` for dynamic instructions. `{key}` placeholders are **not** auto-injected when using a provider; call `instructionutil.InjectSessionState(ctx, template)` to substitute manually. |
| `Mode` | **New in v2.** `llmagent.ModeChat` (default-equivalent), `llmagent.ModeTask`, or `llmagent.ModeSingleTurn`. Lets a coordinator delegate to a sub-agent with isolated conversation history and automatic control return — set `ModeSingleTurn` for agents that run as a focused step inside a `workflow` graph rather than a free-form conversational participant. |
| `Tools` | Slice of `tool.Tool` the agent can invoke. |
| `Toolsets` | Slice of `tool.Toolset` (e.g., `mcptoolset`) for dynamic tool discovery. |
| `SubAgents` | Child agents. Enables LLM-driven delegation via `transfer_to_agent`. |
| `DisallowTransferToParent` | Prevents sub-agent from delegating back to parent. Default `false`. |
| `DisallowTransferToPeers` | Prevents sub-agent from delegating to siblings. Default `false`. |
| `OutputKey` | Stores the agent's final text response in session state under this key. |
| `IncludeContents` | `IncludeContentsDefault` (send history) or `IncludeContentsNone` (current turn only). |
| `InputSchema` / `OutputSchema` | Structured I/O via `*genai.Schema`. `OutputSchema` disables tool use and transfers. |
| `Before/AfterModelCallbacks` | Intercept or replace LLM requests/responses. Return non-nil `*model.LLMResponse` to skip the model call. |
| `Before/AfterToolCallbacks` | Intercept tool execution. `BeforeToolCallback` can return a result map to skip the tool. |
| `OnToolErrorCallbacks` | Handle tool errors. Can return a replacement result or propagate the error. |

For the complete config including all callback fields, read `references/api-reference.md`.

## The Unified `agent.Context` (v2's biggest breaking change)

v1 had two separate, overlapping interfaces: `agent.ToolContext` (inside tool functions) and `agent.CallbackContext` (inside callbacks). **v2 merges them into a single `agent.Context`.** `agent.ReadonlyContext` and `agent.InvocationContext` are unchanged as narrower parent interfaces underneath it.

```go
func myTool(ctx agent.Context, args MyArgs) (MyResult, error) {
    val, _ := ctx.State().Get("user:preferences")      // Read state
    ctx.State().Set("temp:last_result", "value")        // Write state
    ctx.Actions().TransferToAgent = "support_agent"     // Transfer to another agent
    ctx.Actions().Escalate = true                       // Exit a loop
    return MyResult{}, nil
}
```

`agent.Context` adds, on top of `InvocationContext`/`ReadonlyContext`: `Artifacts()`, `State()`, `FunctionCallID()`, `Actions()`, `SearchMemory(ctx, query)`, `ToolConfirmation()`/`RequestConfirmation(hint, payload)` (HITL), `ResumedInput(interruptID)`, and graph-specific accessors (`Path()`, `RunID()`, `SubScheduler()`, `OutputForAncestors()`) used when the context is running inside a `workflow` graph node.

Construct it via `agent.NewContext`, `agent.NewCallbackContext`, `agent.NewToolContext`, or `agent.Promote`/`agent.PromoteWithDelta` — not by hand. All of these return the same `Context` type, just pre-populated differently for the call site (e.g. `NewToolContext` sets `FunctionCallID`).

### Migrating v1 code

- `tool.Context` (v1's deprecated alias) → use `agent.Context` directly.
- Hand-written test mocks of `CallbackContext` **will fail to compile** — they're missing the tool-related methods the merged interface now requires (`Actions()`, `FunctionCallID()`, `ToolConfirmation()`, `RequestConfirmation()`, `SearchMemory()`). Either add those methods manually, or (recommended) embed `agent.StrictContextMock`:

```go
type fakeContext struct {
    agent.StrictContextMock
}

func TestSomething(t *testing.T) {
    cc := &fakeContext{agent.StrictContextMock{Ctx: context.Background()}}
    // Override only the methods your test actually needs.
}
```

`StrictContextMock` implements every context interface and **panics** on any unimplemented method (except base `context.Context` methods, which delegate to `Ctx`) — this is deliberate so a future ADK release that grows the interface makes your test panic loudly instead of silently no-op-ing.

- `session.NewEvent(invocationID)` → `session.NewEvent(ctx, invocationID)`. Use the ambient `context.Context` from the agent/tool/callback, an incoming request context, or `t.Context()` in tests — don't fabricate a fresh one mid-chain.

## Defining Tools

### FunctionTool

Wrap any Go function as an agent tool. Argument and result types are auto-converted to JSON schemas. Args must be a struct or map (or a pointer to one); primitives are rejected.

```go
type WeatherArgs struct {
    City string `json:"city" jsonschema:"The city to get weather for."`
}
type WeatherResult struct {
    Report string `json:"report"`
}

func getWeather(ctx agent.Context, args WeatherArgs) (WeatherResult, error) {
    return WeatherResult{Report: "Sunny, 72F in " + args.City}, nil
}

weatherTool, err := functiontool.New(functiontool.Config{
    Name:        "get_weather",
    Description: "Gets the current weather for a city.",
}, getWeather)
```

For tools that stream incremental results during live (bidi) sessions, use `functiontool.NewStreaming(cfg, func(ctx agent.Context, args TArgs) iter.Seq2[string, error] {...})`.

### MCP Toolset

Connect to MCP servers using the official Go MCP SDK (`github.com/modelcontextprotocol/go-sdk`, not `mark3labs/mcp-go`):

```go
mcpTools, err := mcptoolset.New(mcptoolset.Config{
    Transport: &mcp.CommandTransport{Command: exec.Command("myserver")},
})

agent, err := llmagent.New(llmagent.Config{
    Name:     "mcp_agent",
    Model:    model,
    Toolsets: []tool.Toolset{mcpTools},
})
```

Supports `CommandTransport` (stdio), `StreamableClientTransport` (HTTPS), and in-memory transports. `mcptoolset.Config` gained an `Auth` field (a `auth.CredentialProvider`) in v2 for structured per-request authentication — see `references/integrations.md`. For HITL confirmation on any toolset, use `tool.WithConfirmation`. Two open gaps worth knowing before you rely on MCP heavily: no tool-name-prefixing across multiple servers yet (adk-go#1126), and MCP tool result `_meta` is currently discarded, which blocks some auth-challenge flows like SEP-1036 URL elicitation (adk-go#1165). For full MCP and confirmation details, read `references/integrations.md`.

### AgentTool, Built-in Gemini Tools, Memory/Artifact Tools

Unchanged in shape from v1 — see `references/api-reference.md` for `agenttool.New`, `geminitool.GoogleSearch`, and the `loadartifactstool`/`loadmemorytool`/`preloadmemorytool` trio (still require `ArtifactService`/`MemoryService` set on the `runner.Config`, and `loadartifactstool` still panics if `ArtifactService` is nil — this v1 gotcha was not reported fixed).

## Orchestration: Fixed Workflow Agents vs. the New Graph Engine

ADK Go v2 gives you two orchestration layers. Pick the simplest one that fits:

| Situation | Use |
|---|---|
| A fixed line, fan-out, or bounded loop of whole agents | `sequentialagent` / `parallelagent` / `loopagent` (unchanged from v1) |
| Anything with conditional branching, code-driven dynamic dispatch, typed I/O validation, per-node retry/timeout, fan-in joins, or durable human-in-the-loop pauses | `workflow` (new graph engine) |
| LLM decides routing at runtime based on sub-agent descriptions | `llmagent` + `SubAgents` (`transfer_to_agent`, unchanged) |
| Parent explicitly invokes a child as a tool call | `agenttool.New()` (unchanged) |
| Arbitrary Go control flow not expressible as a graph | Custom `agent.New()` with a hand-written `Run` func (unchanged) |

**`workflow` did not exist in v1.** It models orchestration as a directed graph of `Node`s (agents, functions, sub-workflows) connected by `Edge`s, with built-in retry, timeout, typed schema validation per node, fan-out/fan-in, code-driven dynamic branching (`DynamicNode` + `RunNode`), and durable pause/resume for human-in-the-loop that survives process restarts. Google's own rationale: pure LLM-autonomous multi-step agents are non-deterministic enough in production that a worked refund-processing example sometimes skipped steps across runs; composing deterministic graph structure with LLM steps fixed that and cut token usage roughly in half in their benchmark.

```go
import "google.golang.org/adk/v2/workflow"

upper  := workflow.NewFunctionNode("upper",  upperFn,  cfg)
suffix := workflow.NewFunctionNode("suffix", suffixFn, cfg)
edges := workflow.Chain(workflow.Start, upper, suffix)
wf, err := workflow.New("my_pipeline", edges)

// wf.Run takes agent.InvocationContext, not a bare context.Context — *Workflow
// satisfies agent.Agent, so drive it through a runner like any other agent:
r := runner.NewInMemory("my_app", wf)
for event, err := range r.Run(ctx, "user1", sessionID, input, agent.RunConfig{}) { /* ... */ }
```

**Don't call `wf.Run` with a bare `context.Context`, and don't call `wf.Resume` at all.** `Run` wants `agent.InvocationContext`, which a `runner` supplies for you. `Resume` is lower-level still — its `agent.Context` param is built from constructors that live in an unexported `internal/` package, so application code can't call it directly at all; resuming is done by calling `runner.Run` *again* with a message that answers the pending interrupt, and the runner calls `wf.Resume` internally. See `references/workflow.md` for the exact signatures and the resume-after-restart pattern.

For the full `workflow` API — dynamic orchestration, fan-out/join, durable HITL pause/resume, retry config, conditional routing, embedding `llmagent` agents as graph nodes — read `references/workflow.md`. For the fixed-topology agents (sequential/parallel/loop pipelines, critic/refiner loops, dynamic delegation, composite workflows, remote agents in orchestration), read `references/orchestration.md`.

## State Management

State is scoped by key prefix:

| Prefix | Scope | Persistence |
|---|---|---|
| `app:` | All users, all sessions | Permanent |
| `user:` | Current user, all sessions | Permanent |
| `temp:` | Current invocation only | Discarded after invocation |
| *(none)* | Current session | Session lifetime |

`OutputKey` stores an agent's final text response in session state; `{key}` placeholders in `Instruction` are auto-replaced with state values.

```go
resp, _ := sessionService.Create(ctx, &session.CreateRequest{
    AppName: "my_app",
    UserID:  "user1",
    State:   map[string]any{"topic": "quantum computing"},
})
```

**Known race (open, adk-go#1344):** `session/inmemory` doesn't isolate an unmerged session delta on `Create`, which can leak stale app/user state into a subsequent `Get`. Be cautious concurrently creating sessions that share `AppName`/`UserID` state in tests or high-concurrency demos.

## Running Agents

```go
sessionService := session.InMemoryService()
resp, _ := sessionService.Create(ctx, &session.CreateRequest{AppName: "my_app", UserID: "user1"})

r, _ := runner.New(runner.Config{
    AppName:           "my_app",
    Agent:             myAgent,
    SessionService:    sessionService,
    AutoCreateSession: false, // true: Run creates the session if the ID is unknown
})

input := genai.NewContentFromText("Hello!", genai.RoleUser)
for event, err := range r.Run(ctx, "user1", resp.Session.ID(), input, agent.RunConfig{}) {
    if err != nil { log.Fatal(err) }
    if event.IsFinalResponse() && event.Content != nil {
        for _, part := range event.Content.Parts {
            if part.Text != "" { fmt.Println(part.Text) }
        }
    }
}
```

**New in v2.1.0 — quick-start convenience constructor:**

```go
r := runner.NewInMemory("my_app", myAgent) // fully in-memory session/artifact/memory services, auto-create on
```

`Runner.Run` also accepts trailing `RunOption`s: `runner.WithStateDelta(map[string]any{...})` (inject state before the run) and `runner.WithYieldUserMessage()`.

For live (bidirectional/audio) streaming via `RunLive`, and the `launcher`/`adkgo` deploy CLI, see `references/api-reference.md`. **Security note:** running the launcher's web mode (`go run agent.go web`, or `adk web`) binds the unauthenticated REST API and `/run_live` WebSocket to *all* network interfaces by default, with no WebSocket Origin allowlist (open, adk-go#1154). Don't run it on an untrusted network without a reverse proxy or firewall in front of it.

## Human-in-the-Loop — Two Distinct Mechanisms

v2 has two separate HITL primitives at different granularities; know which one you need.

1. **`tool.WithConfirmation`** — per-tool-call approve/reject gate. Still marked **experimental** in v2.2.0 (unchanged maturity from v1). Wraps any `Toolset`; inside a tool, check `ctx.ToolConfirmation()` / call `ctx.RequestConfirmation(hint, payload)`. `mcptoolset.Config` also exposes `RequireConfirmation`/`RequireConfirmationProvider` directly.
2. **`workflow.ResumeOrRequestInput`** — new in v2, graph-scoped. Pauses an entire multi-node run (not just one tool call) and durably persists the pause — state survives process restarts. Resume by calling `runner.Run` again with a `genai.Content` carrying a `FunctionResponse` keyed to the pause's `InterruptID` (not by calling `Workflow.Resume` directly — that's an internal primitive the runner uses on your behalf). See `references/workflow.md`.

## A2A Remote Agents

Use `agent/remoteagent/v2` (built on a2a-go v2); the non-`/v2` `remoteagent` package is legacy.

```go
import remoteagent "google.golang.org/adk/v2/agent/remoteagent/v2"

remoteAgent, err := remoteagent.NewA2A(remoteagent.A2AConfig{
    Name:              "prime_agent",
    Description:       "Checks if numbers are prime.",
    AgentCardProvider: remoteagent.NewAgentCardProvider("http://localhost:8001"),
})

rootAgent, _ := llmagent.New(llmagent.Config{
    Name:      "root",
    Model:     model,
    SubAgents: []agent.Agent{localAgent, remoteAgent},
})
```

**Gotcha (open, adk-go#1220):** outbound A2A messages from `remoteagent/v2` currently ignore `IsolationScope` — a graph node dispatched with `workflow.WithIsolationScope(...)` can still leak the full shared-session history and a prior dispatch's `contextID` to the remote agent. Don't rely on isolation scoping for A2A privacy boundaries yet. Also open: `AfterA2ARequestCallbacks` aren't invoked on aggregated events synthesized from partial artifact chunks (#948), and remote task cleanup can leave an orphaned task if the parent is cancelled before the first remote event (#1076).

For exposing an agent over A2A (server side) and full `A2AConfig`, read `references/api-reference.md`.

## Agent Registry (new)

`google.golang.org/adk/v2/agentregistry` is a client for Google Cloud's Agent Registry — a governed catalog of A2A agents, MCP servers, and model endpoints. Instead of hand-wiring `remoteagent.NewA2A`/`mcptoolset.New` with hardcoded URLs and cards, discover and instantiate them by name:

```go
client, err := agentregistry.New(ctx, agentregistry.Config{ /* ... */ })
remote, err := client.RemoteAgent(ctx, "billing-agent")
mcpTools, err := client.MCPToolset(ctx, "internal-search")
```

See `references/integrations.md` for `ListAgents`/`AllAgents`/`GetAgent` and the parallel MCP server/endpoint methods.

## Auth (new)

`google.golang.org/adk/v2/auth` is a lightweight, per-request credential layer used by `mcptoolset.Config.Auth` and other tool integrations. It delegates actual token refresh to `golang.org/x/oauth2` rather than reimplementing it, and resolves credentials on every request so per-user credentials are never accidentally shared across users.

```go
type Credential interface { Apply(h http.Header) error }
// BearerCredential, BasicCredential, APIKeyCredential, OAuth2Credential
type CredentialProvider interface { Credential(ctx context.Context) (Credential, error) }
// StaticToken, APIKey, TokenSourceProvider, ADC, ServiceAccount, ProviderFunc
```

See `references/integrations.md` for the full surface.

## Plugins & Telemetry

Unchanged in shape from v1 (`retryandreflect`, `functioncallmodifier`, `loggingplugin`, attach via `runner.PluginConfig`). New in v2.1.0: a **BigQuery Agent Analytics plugin**. It has an open bug logging one `MODEL_RESPONSE` row per streamed chunk instead of once per turn (adk-go#1210), so expect duplicated usage-metadata rows if you enable it with streaming until that's fixed. For the full `plugin.Config` type and telemetry options, read `references/api-reference.md`.

## Migrating from v1

If you have existing ADK Go v1.x code and want v2's new capabilities:

1. Change every `google.golang.org/adk/...` import to `google.golang.org/adk/v2/...` and run `go mod tidy`.
2. Replace every `session.NewEvent(id)` / `session.NewEventWithContext(ctx, id)` call with `session.NewEvent(ctx, id)`.
3. Replace `tool.Context` and any direct `agent.ToolContext`/`agent.CallbackContext` usage with `agent.Context`.
4. Fix hand-written test mocks of `CallbackContext`/`ToolContext` — either add the newly-required methods or switch to embedding `agent.StrictContextMock` (recommended; see above).
5. You do **not** need to adopt `workflow` to migrate — `sequentialagent`/`parallelagent`/`loopagent` and everything else in the v1.4.0 skill's orchestration surface still works. Adopt `workflow` only where you actually want graph features (dynamic branching, durable HITL, per-node retry/typed schemas).
6. Re-run your test suite — the context-merge change is the one most likely to cause compile errors, not runtime behavior changes.

## Known Gotchas

Open issues verified against `gh issue list --repo google/adk-go` (2026-08-17) unless noted otherwise as carried forward from v1.

- **`wf.Run` does not take a bare `context.Context`, and `wf.Resume` isn't meant to be called by application code at all.** This is the single most common error observed when generating code against this skill. `Run` needs `agent.InvocationContext` — drive it through a `runner` (since `*Workflow` satisfies `agent.Agent`) rather than calling it directly with a hand-built context. `Resume` needs an `agent.Context` built from constructors in an unexported `internal/context` package your code can't import — the public way to resume is calling `runner.Run` again with a `FunctionResponse`-carrying message keyed to the pause's `InterruptID`; the runner calls `wf.Resume` for you. See `references/workflow.md`.
- **`adk web` / launcher web mode binds all interfaces with no auth and no WebSocket Origin check** (#1154, open, security-relevant). Never expose it directly on an untrusted network.
- **Data race in parallel `ModeSingleTurn` dispatch** (#1137, open). Concurrent single-turn dispatches of the same sub-agent from a `ParallelWorker`/parallel graph fan-out can race on shared agent state.
- **`IsolationScope` not honored for outbound A2A** (#1220, open) — see A2A section above.
- **OpenAI backend breaks on multi-turn** (#1197, open) — `input_text` vs `output_text` content-type mismatch causes a 400. No native Anthropic backend exists yet (#1097 tracks it).
- **`session/inmemory` Create doesn't isolate unmerged deltas** (#1344, open) — can leak stale app/user state into a subsequent `Get`.
- **No MCP tool-name prefixing across multiple servers** (#1126, open) and **MCP tool result `_meta` is discarded** (#1165, open) — blocks some auth-challenge flows (e.g. SEP-1036 URL elicitation).
- **`preloadmemorytool` nil-pointer panic** on nil parts in a memory entry (#1348, open).
- **`loadartifactstool` still panics** if `ArtifactService` is nil on the runner (carried forward from v1, not reported fixed) — always set `ArtifactService` when using artifact tools.
- **`OutputSchema` disables tool use and transfers.** Use it only on leaf agents producing structured final output (unchanged from v1).
- **Agents cannot implement `agent.Agent` directly** — it has an unexported method. Always construct via `agent.New`, `llmagent.New`, workflow-agent constructors, or `workflow.New` (unchanged from v1).
- **Agent identity auto-injection** — `Name`/`Description` are injected into the LLM system prompt automatically; avoid duplicating identity text in `Instruction` (unchanged from v1).
- **`adkgo deploy cloudrun` has no way to set the runtime service account**, and its Secret Manager requirement is undocumented (#1343, open).
- **BigQuery Agent Analytics plugin over-logs streamed chunks** — see Plugins section above (#1210, open).
- **Pinned dependency versions drift across ADK Go patch releases** (genai SDK was 1.63.0 at v2.1.0, 1.66.0 on `main` shortly after) — check the target ADK Go version's own `go.mod` rather than hardcoding a genai/MCP-SDK/a2a-go version in your own `go.mod` reasoning.

## Reference Files

| File | Contents | Load when |
|---|---|---|
| `references/workflow.md` | The new graph-based `workflow` engine: nodes, edges, dynamic orchestration (`DynamicNode`/`RunNode`), fan-out/fan-in (`ParallelWorker`/`JoinNode`), durable HITL pause/resume, retry config, conditional routing, embedding `llmagent` as a node | Designing any orchestration beyond a fixed sequential/parallel/loop pipeline, or implementing durable human-in-the-loop |
| `references/orchestration.md` | Fixed workflow-agent patterns with complete code: sequential pipeline, parallel fan-out/gather, critic/refiner loop, dynamic delegation, custom `Run`-func agents, composite workflows, remote agents in orchestration | Designing a fixed-topology multi-agent workflow, or implementing one of these specific patterns |
| `references/api-reference.md` | Complete types: `agent.Context`/`agent.StrictContextMock`, `llmagent.Config` (incl. `Mode`), all callback signatures, `session.Event`, `EventActions`, `genai.GenerateContentConfig`, `runner.Config`/`RunOption`, `RunLive`/`LiveRunConfig`, `remoteagent/v2`, `skilltoolset`, `exampletool`, `plugin.Config`, built-in plugin configs, telemetry options | Looking up specific field names, types, or callback signatures |
| `references/integrations.md` | MCP toolset (all transports, filtering, HITL, `Auth`), Agent Registry (`agentregistry`), the `auth` package, OpenAI/custom `model.LLM` providers, Apigee model proxy, Vertex AI config, service backends (database/Vertex AI sessions, Memory Bank, GCS artifacts) | Connecting to MCP servers or the Agent Registry, wiring credentials, using non-Gemini models, writing a custom provider, or choosing service backends |
