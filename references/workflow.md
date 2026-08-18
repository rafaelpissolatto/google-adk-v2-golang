# The `workflow` Graph Engine (new in ADK Go v2)

`google.golang.org/adk/v2/workflow` has no v1 equivalent. It models orchestration as a directed graph of `Node`s connected by `Edge`s, with built-in retry/timeout per node, typed I/O schema validation, fan-out/fan-in, code-driven dynamic branching, and durable human-in-the-loop pause/resume that survives process restarts.

Use it instead of `sequentialagent`/`parallelagent`/`loopagent` (still available, see `references/orchestration.md`) whenever you need conditional branching, per-node retry/timeout policy, typed schema validation, a real fan-in barrier, or a pause that must survive a process restart. Google's stated rationale: pure-autonomous multi-step LLM agents are non-deterministic enough in production that a worked refund-processing benchmark sometimes skipped steps across runs; composing deterministic graph structure with LLM steps fixed that and, in their benchmark, cut token usage by roughly half (5,152 → 2,265 tokens) and latency by roughly a fifth (7.2s → 5.7s) versus a pure-autonomous-agent baseline for the same task.

Note: exact byte-for-byte signatures below were reconstructed from `pkg.go.dev` documentation, not a raw source dump — treat the shapes as reliable, but re-check `pkg.go.dev/google.golang.org/adk/v2/workflow` if something doesn't compile exactly as shown.

## Core Concept: `Node` and `Edge`

Everything in a graph — a plain function, an `llmagent`, a sub-workflow — implements `Node`:

```go
type Node interface {
    Name() string
    Description() string
    Config() NodeConfig
    InputSchema() *jsonschema.Schema
    OutputSchema() *jsonschema.Schema
    ValidateInput(any) error
    ValidateOutput(any) error
    Run(...) // scheduled by the graph engine
}

var Start Node // sentinel entry point for a graph
```

`NodeConfig` carries per-node policy: `ParallelWorker bool`, `RerunOnResume *bool`, `WaitForOutput *bool`, `RetryConfig *RetryConfig`, `Timeout time.Duration`, `EmitsOwnSpan bool`.

## Building a Simple Chain

```go
import "google.golang.org/adk/v2/workflow"

upper  := workflow.NewFunctionNode("upper",  upperFn,  cfg)
suffix := workflow.NewFunctionNode("suffix", suffixFn, cfg)

edges := workflow.Chain(workflow.Start, upper, suffix)
wf, err := workflow.New("my_pipeline", edges)
```

**`*Workflow.Run`'s signature is `Run(ctx agent.InvocationContext) iter.Seq2[*session.Event, error]` — it does NOT take a bare `context.Context`.** This is the same shape as `agent.Agent.Run`, and in practice `*Workflow` satisfies `agent.Agent`, so the normal way to drive it is exactly like any other agent: hand it to a `runner`, which constructs the `InvocationContext` for you. Don't call `wf.Run(ctx)` directly with a `context.Context` you built yourself — it won't compile, and even if you hand-construct an `agent.InvocationContext`, you'd be bypassing session/event plumbing the runner otherwise gives you for free.

```go
r := runner.NewInMemory("my_app", wf) // *Workflow used as an agent.Agent
input := genai.NewContentFromText("start", genai.RoleUser)
for event, err := range r.Run(ctx, "user1", sessionID, input, agent.RunConfig{}) {
    if err != nil { log.Fatal(err) }
    // handle event
}
```

`workflow.New(name string, edges []Edge, opts ...Option)` accepts `WithMaxConcurrency(n int)`, `WithStateSchema(s *jsonschema.Resolved)`, and `WithRootWrapper()`. `Chain(nodes ...Node) []Edge` is the simplest edge builder for a straight-line graph; `Concat(items ...any) []Edge` composes edge lists/nodes more flexibly for non-linear graphs.

## Function Nodes

```go
func NewFunctionNode[IN, OUT any](name string, fn func(ctx agent.Context, input IN) (OUT, error), cfg NodeConfig) *FunctionNode

func NewFunctionNodeWithSchema[IN, OUT any](name string, fn ..., inputSchema, outputSchema *jsonschema.Schema, cfg NodeConfig) *FunctionNode
    // explicit schemas instead of reflection-inferred ones

func NewEmittingFunctionNode[IN, OUT any](name string, fn EmittingFunctionFn[IN, OUT], cfg NodeConfig) *FunctionNode
    // lets the node emit intermediate *session.Event values while it runs, not just a final result

func NewFunctionNodeFromState[Params, OUT any](name string, fn func(ctx agent.InvocationContext, p Params) (OUT, error), cfg NodeConfig) *FunctionNode
    // binds Params automatically from session state instead of the graph's data-flow input
```

## Dynamic Orchestration

This is the concrete answer to "how do I branch/loop based on runtime data inside a graph": a `DynamicNode`'s function body is **plain Go code**. It calls `workflow.RunNode` per child it wants to invoke — ordinary `if`/`switch`/`for` gives you conditional branching and loops, no special DSL required.

```go
func NewDynamicNode[IN, OUT any](name string, fn DynamicFn[IN, OUT], cfg NodeConfig) Node
type DynamicFn[IN, OUT any] func(ctx agent.Context, in IN, emit func(*session.Event) error) (OUT, error)

func RunNode[OUT any](ctx agent.Context, child Node, input any, opts ...RunNodeOption) (OUT, error)
// options:
//   WithRunID(id string)          — stable id for this child invocation (needed for resumable/idempotent reruns)
//   WithUseAsOutput()             — child's output becomes this dynamic node's output
//   WithUseSubBranch()            — run the child on an isolated event branch
//   WithOverrideBranch(branch)    — pin to a specific branch string
//   WithIsolationScope(scope)     — isolate session/state visibility (see A2A caveat in SKILL.md — not
//                                    yet honored for outbound A2A traffic, adk-go#1220)
//   WithRaiseOnWait()             — return immediately instead of blocking if the child pauses for HITL
```

A dynamically-invoked child scheduled via `RunNode` gets the same persistence, tracing, and pause/resume semantics as a statically-wired edge — it's not a lesser "escape hatch," it's first-class.

```go
router := workflow.NewDynamicNode("router", func(ctx agent.Context, in Ticket, emit func(*session.Event) error) (Resolution, error) {
    switch in.Category {
    case "billing":
        return workflow.RunNode[Resolution](ctx, billingNode, in)
    case "technical":
        return workflow.RunNode[Resolution](ctx, supportNode, in)
    default:
        return workflow.RunNode[Resolution](ctx, generalNode, in)
    }
}, cfg)
```

**Threshold-gated branch (e.g. "skip the approval node entirely below $500"):** when the condition decides whether a *whole downstream node* runs at all — not just how one node's own body behaves — model it as a `DynamicNode` that conditionally calls `RunNode` on that node, rather than folding the `if` into a plain `FunctionNode`/`EmittingFunctionNode` and always running it:

```go
gate := workflow.NewDynamicNode("approval_gate", func(ctx agent.Context, in PolicyCheckResult, emit func(*session.Event) error) (ApprovalResult, error) {
    if in.Amount <= approvalThreshold {
        return ApprovalResult{Approved: true, ApproverID: "system:auto-approved"}, nil // approvalNode never runs
    }
    return workflow.RunNode[ApprovalResult](ctx, approvalNode, in, workflow.WithRunID("approval-"+in.RequestID))
}, cfg)

edges := workflow.Chain(workflow.Start, validateNode, policyNode, gate, issueNode)
```

Both this and a single node with an internal `if` are valid ADK patterns — a lone `if` guarding a *call* inside one node's body (e.g. "call `ResumeOrRequestInput` only when `RequiresApproval`") is fine and common. Reach for `DynamicNode`+`RunNode` specifically when the branches are (or could become) distinct, independently-testable `Node`s, since routing through `RunNode` is what gives each branch its own tracing span, retry/timeout policy, and `WithRunID` for resumable reruns — an internal `if` shares all of that with the rest of its enclosing node.

## Embedding `llmagent` Agents as Graph Nodes

New plumbing in `llmagent` lets an existing `agent.Agent` (typically built with `llmagent.New`) run directly as a graph node instead of only via `runner`/`SubAgents`:

```go
func llmagent.PrepareLLMAgentInput(a agent.Agent, ctx agent.InvocationContext, nodeInput any) *session.Event
func llmagent.ProcessLLMAgentOutput(a agent.Agent, ev *session.Event) error
func llmagent.RunLLMAgentAsNode(a agent.Agent, ctx agent.Context, nodeInput any) iter.Seq2[*session.Event, error]
```

Set `llmagent.Config.Mode = llmagent.ModeSingleTurn` on agents meant to run as one focused graph step (isolated conversation history, automatic control return) rather than `ModeChat`, which is meant for open-ended conversational participants.

## Fan-Out / Fan-In

```go
func NewJoinNode(name string) *JoinNode
    // fan-in barrier: fires exactly once, after ALL predecessor edges into it have completed;
    // aggregates their outputs as map[string]any keyed by predecessor node name

func NewParallelWorker(name string, wrapped Node, maxConcurrency int, cfg NodeConfig) (*ParallelWorker, error)
    // runs `wrapped` concurrently over a list of input items, at most maxConcurrency in flight at once

func NewWorkflowNode(name string, edges []Edge) (*WorkflowNode, error)
    // embeds an entire nested sub-graph as a single node in a parent graph — compose graphs of graphs
```

**Known bug (open, adk-go#1137):** concurrent `ModeSingleTurn` dispatches of the *same* sub-agent from a `ParallelWorker` (or any parallel graph fan-out) can race on shared agent state. If you fan out over the same agent definition, prefer independent agent instances per branch, or verify this is fixed in your target ADK Go version before relying on it.

## Durable Human-in-the-Loop

This is the "built-in HITL" v2 introduces at the graph level — distinct from the older, still-experimental `tool.WithConfirmation` per-tool-call gate (see SKILL.md's HITL section for the distinction).

```go
func NewRequestInputEvent(ctx agent.InvocationContext, req session.RequestInput) *session.Event
func ResumeOrRequestInput(ctx agent.Context, emit func(*session.Event) error, req session.RequestInput) (any, error)
```

Any node can call `ResumeOrRequestInput` to pause the *entire run* — not just gate one tool call — and durably persist that pause: the state survives process restarts, not just an in-memory channel wait. `session.RequestInput` carries an `InterruptID` (auto-UUID if empty), an optional human-facing `Message`, optional JSON-schema validation for the expected response, and an optional `Payload` for context to show the human.

Resuming a paused run:

**Don't call `wf.Resume` yourself.** `Resume`'s real signature is `Resume(ctx agent.Context, state *RunState, responses map[string]any) iter.Seq2[*session.Event, error]`, but the `agent.Context`/`agent.InvocationContext` constructors it needs (`icontext.NewInvocationContext`) live in `google.golang.org/adk/v2/internal/context` — an `internal/` package, so Go's compiler refuses to let application code outside the `adk-go` module import it at all. `Resume` is the low-level primitive the *runner* calls internally, not a public entry point for your code, exactly the same way `Run` isn't meant to be called with a hand-built context — the difference is `Run`'s inputs happen to be constructible externally and `Resume`'s aren't.

The actual public way to resume: call `runner.Run` again — the *same* `Run` you used to start the workflow — with a message that answers the pending interrupt. The runner detects the paused `RunState`, matches your message against it, and calls `wf.Resume` for you.

```go
sess, err := sessionService.Get(ctx, &session.GetRequest{AppName: appName, UserID: userID, SessionID: sessionID})
// sess.Session carries the paused RunState; wf.ReconstructRunState(sess.Session, invocationID)
// is what the runner uses internally to detect the pause — you don't call it yourself either.

r, err := runner.New(runner.Config{AppName: appName, Agent: wf, SessionService: sessionService})

// The response message must carry a FunctionResponse part whose ID matches the
// session.RequestInput.InterruptID from the pause, and whose Name is
// workflow.WorkflowInputFunctionCallName — that's how the runner matches your
// answer to the specific pending interrupt.
resumeMsg := &genai.Content{
    Role: genai.RoleUser,
    Parts: []*genai.Part{{
        FunctionResponse: &genai.FunctionResponse{
            ID:   "approval-interrupt-id", // == the InterruptID the pause used
            Name: workflow.WorkflowInputFunctionCallName,
            Response: map[string]any{"payload": humanResponse},
        },
    }},
}

for event, err := range r.Run(ctx, userID, sess.Session.ID(), resumeMsg, agent.RunConfig{}) {
    // ...
}
```

This is what makes the pause "durable" rather than a simple in-process blocking wait: the paused state lives in the session, so the process that resumes it can be a completely different process than the one that paused, as long as they share the same durable `session.Service` backend. Resuming is then just "call `Run` again with the right message" — no separate resume code path, no manual context plumbing.

## Retry & Resilience

```go
func DefaultRetryConfig() *RetryConfig      // 5 attempts, 1s initial delay, 60s cap, 2x backoff
func ShouldRetry(cfg *RetryConfig, err error, failedAttempts int) bool
func CalculateDelay(cfg *RetryConfig, failedAttempts int) time.Duration
```

Set `NodeConfig.RetryConfig` per node; leave nil to disable retries for that node. `NodeConfig.Timeout` bounds a single attempt.

## Conditional Routing

```go
func StringRoute(string) ...
func IntRoute(int) ...
func BoolRoute(bool) ...
func MultiRoute[T comparable]([]T) ...
var Default ... // matches any case not otherwise routed
```

Use these to build `Edge` sets where the next node depends on a previous node's typed output value, without dropping into a `DynamicNode`.

## How This Relates to the Fixed Workflow Agents

- `sequentialagent`/`parallelagent`/`loopagent` (`references/orchestration.md`) — fixed, simple topologies, agent-only nodes, minimal config. Still present in v2, unchanged in spirit from v1.
- `workflow` — general directed graph; nodes can be functions, agents, tools, or nested sub-workflows; supports conditional routing, code-driven dynamic branching, fan-out/fan-in, per-node retry/timeout, typed I/O schema validation, and durable HITL pause/resume. Did not exist in v1 at all.

They're not mutually exclusive — `workflow.NewWorkflowNode` can embed a `sequentialagent`-style pipeline as one node inside a larger graph, and `llmagent.RunLLMAgentAsNode` lets any existing agent (including one that's itself a `sequentialagent`/`parallelagent`/`loopagent`) participate as a graph node.
