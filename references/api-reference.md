# API Reference

Type reference for ADK Go v2. Most shapes are carried forward from v1.4.0 unchanged (no evidence of a diff surfaced in research); the ones confirmed changed or new in v2 are marked **(v2)**. Signatures came from `pkg.go.dev` documentation, not a raw source dump — treat as reliable but re-verify exact field/method names against `pkg.go.dev/google.golang.org/adk/v2/...` before treating this as byte-exact ground truth for anything unusual.

## Core Interfaces

### agent.Agent (unchanged in shape)

```go
type Agent interface {
    Name() string
    Description() string
    Run(InvocationContext) iter.Seq2[*session.Event, error]
    SubAgents() []Agent
    FindAgent(name string) Agent     // Recursive lookup by name.
    FindSubAgent(name string) Agent

    // Has an unexported method: user code CANNOT implement this interface.
    // Always construct agents via agent.New, llmagent.New, workflow-agent
    // constructors, or workflow.New (the new graph engine).
}
```

### model.LLM

```go
type LLM interface {
    Name() string
    GenerateContent(ctx context.Context, req *LLMRequest, stream bool) iter.Seq2[*LLMResponse, error]
}
```

Not independently confirmed whether this differs from v1.4.0 — the v1.4.0 baseline used for this skill's research didn't spell out the interface shape. Treat this as "current v2.2.0 shape," not "known to have changed."

**(v2) Model registry**, new in v2.1.0:
```go
func model.Register(pattern string, factory Factory)  // Factory constructs an LLM from a model name
func model.NewLLM(name string) (LLM, error)            // resolves via registered patterns
```

### tool.Tool / tool.Toolset (unchanged)

```go
type Tool interface {
    Name() string
    Description() string
    IsLongRunning() bool
}

type Toolset interface {
    Name() string
    Tools(ctx agent.ReadonlyContext) ([]Tool, error)
}
```

## agent.Config

Used with `agent.New()` for custom agents.

```go
type Config struct {
    Name                 string
    Description          string
    SubAgents            []Agent
    BeforeAgentCallbacks []BeforeAgentCallback
    AfterAgentCallbacks  []AfterAgentCallback
    Run                  func(InvocationContext) iter.Seq2[*session.Event, error]
}
func New(cfg Config) (Agent, error)
```

## llmagent.Config

All fields for `llmagent.New()`. **(v2)** `Mode` is the one clearly-new field vs. v1.4.0; everything else matches the v1.4.0 shape closely.

```go
type Config struct {
    // Identity
    Name        string          // Required. Unique in agent tree. Cannot be "user".
    Description string          // Used by parent agents for delegation routing.

    // Mode (v2)
    Mode Mode  // ModeUnset (default) | ModeChat | ModeTask | ModeSingleTurn

    // Model
    Model                   model.LLM
    GenerateContentConfig   *genai.GenerateContentConfig

    // Instructions
    Instruction               string               // Supports {state_key} substitution.
    InstructionProvider       InstructionProvider   // Dynamic instruction generation.
    GlobalInstruction         string               // Prepended to all sub-agent instructions.
    GlobalInstructionProvider InstructionProvider

    // Sub-Agents
    SubAgents                []agent.Agent
    DisallowTransferToParent bool     // Prevent delegating back to parent.
    DisallowTransferToPeers  bool     // Prevent delegating to siblings.

    // Tools
    Tools                []tool.Tool
    Toolsets              []tool.Toolset

    // Schema
    InputSchema  *genai.Schema    // Structured input validation.
    OutputSchema *genai.Schema    // Structured output. Disables tool use.

    // Output
    OutputKey       string           // Stores final response in state[OutputKey].
    IncludeContents IncludeContents  // "none" or "default".

    // Agent Callbacks
    BeforeAgentCallbacks []agent.BeforeAgentCallback
    AfterAgentCallbacks  []agent.AfterAgentCallback

    // Model Callbacks
    BeforeModelCallbacks  []BeforeModelCallback
    AfterModelCallbacks   []AfterModelCallback
    OnModelErrorCallbacks []OnModelErrorCallback

    // Tool Callbacks
    BeforeToolCallbacks  []BeforeToolCallback
    AfterToolCallbacks   []AfterToolCallback
    OnToolErrorCallbacks []OnToolErrorCallback
}
```

```go
type Mode int
const (
    ModeUnset Mode = iota
    ModeChat
    ModeTask
    ModeSingleTurn
)
```

**(v2) Node-integration functions** (for embedding an `llmagent`-built agent directly into a `workflow` graph — see `references/workflow.md`):
```go
func PrepareLLMAgentInput(a agent.Agent, ctx agent.InvocationContext, nodeInput any) *session.Event
func ProcessLLMAgentOutput(a agent.Agent, ev *session.Event) error
func RunLLMAgentAsNode(a agent.Agent, ctx agent.Context, nodeInput any) iter.Seq2[*session.Event, error]
```

### InstructionProvider

```go
type InstructionProvider func(ctx agent.ReadonlyContext) (string, error)
```

## Callback Signatures (unchanged in shape from v1)

### Agent Callbacks

```go
// Return non-nil *genai.Content to skip agent execution.
type BeforeAgentCallback func(CallbackContext) (*genai.Content, error)

// Called after agent completes.
type AfterAgentCallback func(CallbackContext) (*genai.Content, error)
```

Note: `CallbackContext` here now resolves to the unified `agent.Context` type (see §agent.Context below) rather than a distinct v1-style `CallbackContext` type.

### Model Callbacks

```go
type BeforeModelCallback func(
    ctx agent.CallbackContext,
    llmRequest *model.LLMRequest,
) (*model.LLMResponse, error)

type AfterModelCallback func(
    ctx agent.CallbackContext,
    llmResponse *model.LLMResponse,
    llmResponseError error,
) (*model.LLMResponse, error)

type OnModelErrorCallback func(
    ctx agent.CallbackContext,
    llmRequest *model.LLMRequest,
    llmResponseError error,
) (*model.LLMResponse, error)
```

### Tool Callbacks

```go
type BeforeToolCallback func(
    ctx agent.Context,
    tool tool.Tool,
    args map[string]any,
) (map[string]any, error)

type AfterToolCallback func(
    ctx agent.Context,
    tool tool.Tool,
    args map[string]any,
    result map[string]any,
    err error,
) (map[string]any, error)

type OnToolErrorCallback func(
    ctx agent.Context,
    tool tool.Tool,
    args map[string]any,
    err error,
) (map[string]any, error)
```

## Context Interfaces **(v2 — unified, see SKILL.md)**

### agent.ReadonlyContext (unchanged)

```go
type ReadonlyContext interface {
    context.Context
    UserContent() *genai.Content
    InvocationID() string
    AgentName() string
    ReadonlyState() session.ReadonlyState
    UserID() string
    AppName() string
    SessionID() string
    Branch() string
}
```

### agent.InvocationContext (unchanged)

```go
type InvocationContext interface {
    context.Context
    Agent() Agent
    Artifacts() Artifacts
    Memory() Memory
    Session() session.Session
    InvocationID() string
    Branch() string
    IsolationScope() string
    UserContent() *genai.Content
    RunConfig() *RunConfig
    EndInvocation()
    Ended() bool
    ResumedInput(interruptID string) any
    WithContext(context.Context) InvocationContext
}
```

### agent.Context — the unified type (v2)

Replaces v1's separate `agent.ToolContext` and `agent.CallbackContext`. Sits atop `InvocationContext`/`ReadonlyContext`.

```go
type Context interface {
    InvocationContext
    ReadonlyContext

    Artifacts() Artifacts                                          // Save/List/Load/LoadVersion
    State() session.State                                          // mutable session state
    FunctionCallID() string                                        // tool invocation id
    Actions() *session.EventActions                                // state/artifact deltas, agent transfer
    SearchMemory(ctx context.Context, query string) (*memory.SearchResponse, error)
    ToolConfirmation() *toolconfirmation.ToolConfirmation           // HITL confirmation handle
    RequestConfirmation(hint string, payload any) error             // initiate per-tool-call HITL

    // Graph/dynamic-node context (populated when running as a workflow node)
    Path() string
    RunID() string
    SubScheduler() any
    OutputForAncestors() any

    WithAgentContext(context.Context) Context
    WithAgentTimeout(time.Duration) Context
    WithAgentCancel() (Context, context.CancelFunc)
    WithDelta(CommonContextDelta) Context
}
```

Constructors:
```go
func NewContext(parent InvocationContext) Context
func NewCallbackContext(ic InvocationContext, actions *session.EventActions) Context
func NewCallbackContextWithArtifactTracking(ic InvocationContext, actions *session.EventActions) Context
func NewToolContext(ic InvocationContext, functionCallID string, actions *session.EventActions, confirmation *toolconfirmation.ToolConfirmation) Context
func NewCleanToolContextTestOnly(...) Context   // testing only
func Promote(parent InvocationContext) Context
func PromoteWithDelta(ctx InvocationContext, delta CommonContextDelta) Context
```

All construct the same `Context` type, pre-populated differently for their call site (e.g. `NewToolContext` sets `FunctionCallID`/confirmation; `NewCallbackContext` doesn't).

### agent.StrictContextMock — test double for `Context` (v2)

```go
type StrictContextMock struct {
    Ctx context.Context
    // ... embed and override only what a given test needs
}
```

Implements every context interface; **panics** on any unimplemented method call except the base `context.Context` methods (delegated to `Ctx`). Forward-compatible by design — a future interface addition makes an un-overridden test panic loudly rather than silently no-op. Recommended over hand-rolled mocks (see SKILL.md's Migrating section for the embedding pattern).

## session.Event (unchanged in shape; construction signature changed — see below)

```go
type Event struct {
    model.LLMResponse                     // Embedded: Content, metadata, etc.
    ID               string
    Timestamp        time.Time
    InvocationID     string
    Branch           string               // "agent1.agent2.agent3" hierarchy path
    Author           string               // "user" or agent name
    Actions          EventActions
    LongRunningToolIDs []string
}

// (v2) NewEvent now takes context.Context as the first argument.
// v1: session.NewEvent(invocationID)  /  session.NewEventWithContext(ctx, invocationID)
// v2: session.NewEvent(ctx, invocationID)
func NewEvent(ctx context.Context, invocationID string) *Event

func (e *Event) IsFinalResponse() bool
```

### session.Events (unchanged)

```go
type Events interface {
    All() iter.Seq[*Event]
    Len() int
    At(i int) *Event
}
```

### session.EventActions (unchanged)

```go
type EventActions struct {
    StateDelta                 map[string]any
    ArtifactDelta              map[string]int64
    RequestedToolConfirmations map[string]toolconfirmation.ToolConfirmation
    SkipSummarization          bool
    TransferToAgent            string    // Target agent name for delegation.
    Escalate                   bool      // Exit loop / escalate to parent.
}
```

### session.RequestInput **(v2, new — used by the `workflow` HITL primitives)**

```go
type RequestInput struct {
    InterruptID string          // auto-UUID if empty
    Message     string          // optional human-facing prompt
    Schema      *jsonschema.Schema // optional validation for the expected response
    Payload     any             // optional context shown to the human
}
```

See `references/workflow.md` for `workflow.NewRequestInputEvent`/`ResumeOrRequestInput` usage.

## model.LLMRequest / LLMResponse (unchanged in shape)

```go
type LLMRequest struct {
    Model    string
    Contents []*genai.Content
    Config   *genai.GenerateContentConfig
    Tools    map[string]any `json:"-"`
}

type LLMResponse struct {
    Content             *genai.Content
    CitationMetadata    *genai.CitationMetadata
    GroundingMetadata   *genai.GroundingMetadata
    UsageMetadata       *genai.GenerateContentResponseUsageMetadata
    CustomMetadata      map[string]any
    LogprobsResult      *genai.LogprobsResult
    InputTranscription  *genai.Transcription   // Live sessions.
    OutputTranscription *genai.Transcription   // Live sessions.
    ModelVersion        string
    Partial             bool             // Streaming: incomplete chunk.
    TurnComplete        bool             // Streaming: response fully complete.
    Interrupted         bool
    SessionResumptionHandle string       // Live sessions.
    ErrorCode           string
    ErrorMessage        string
    FinishReason        genai.FinishReason
    AvgLogprobs         float64
}
```

## session.State (unchanged)

```go
type State interface {
    Get(string) (any, error)
    Set(string, any) error
    All() iter.Seq2[string, any]
}

type ReadonlyState interface {
    Get(string) (any, error)
    All() iter.Seq2[string, any]
}
```

## session.Service (unchanged in shape)

```go
type Service interface {
    Create(context.Context, *CreateRequest) (*CreateResponse, error)
    Get(context.Context, *GetRequest) (*GetResponse, error)
    List(context.Context, *ListRequest) (*ListResponse, error)
    Delete(context.Context, *DeleteRequest) error
    AppendEvent(context.Context, Session, *Event) error
}

func InMemoryService() Service

type GetRequest struct {
    AppName   string
    UserID    string
    SessionID string
    NumRecentEvents int
    After           time.Time
}
```

Database-backed: `google.golang.org/adk/v2/session/database`. Vertex AI-backed: `google.golang.org/adk/v2/session/vertexai`. **Known bug (open, adk-go#1344):** `session/inmemory`'s `Create` doesn't isolate an unmerged session delta, which can leak stale app/user state into a subsequent `Get`. **Also open:** `session/database` has no write-lease/OCC mechanism for a second concurrent writer (#1229), and fails persisting app/user state on MySQL strict mode due to a zero-value `update_time` (#1177).

## runner.Config and Runner

```go
type Config struct {
    AppName           string
    Agent             agent.Agent
    SessionService    session.Service
    ArtifactService   artifact.Service    // Optional.
    MemoryService     memory.Service      // Optional.
    PluginConfig      PluginConfig
    AutoCreateSession bool
}

type PluginConfig struct {
    Plugins      []*plugin.Plugin
    CloseTimeout time.Duration
}

func New(cfg Config) (*Runner, error)

// (v2) NewInMemory: convenience constructor, added v2.1.0. Fully in-memory
// session/artifact/memory services, session auto-creation on.
func NewInMemory(appName string, a agent.Agent) *Runner

func (r *Runner) Run(ctx context.Context, userID, sessionID string, msg *genai.Content,
    cfg agent.RunConfig, opts ...RunOption) iter.Seq2[*session.Event, error]

func (r *Runner) RunLive(ctx context.Context, userID, sessionID string,
    cfg agent.LiveRunConfig, opts ...RunOption) (agent.LiveSession, iter.Seq2[*session.Event, error], error)

type RunOption func(*runOptions)
func WithStateDelta(delta map[string]any) RunOption   // Inject state before the run.
func WithYieldUserMessage() RunOption                  // (v2) also yield the synthesized user-message event.
```

**Known bug (open, adk-go#1346):** nil session events can trigger nil-pointer panics in `Run`, `RunLive`, and the REST `RunHandler`.

## agent.RunConfig

```go
type RunConfig struct {
    StreamingMode             StreamingMode
    SaveInputBlobsAsArtifacts bool
}

type StreamingMode string
const (
    StreamingModeNone StreamingMode = "none"
    StreamingModeSSE  StreamingMode = "sse"
)
```

This is a slimmer surface than some v1 documentation implied — could not confirm exhaustively whether other v1 `RunConfig` fields were dropped vs. simply not surfaced by the doc renderer used in this skill's research; flag for a direct diff if you rely on a field not listed here.

## Live Sessions (unchanged in shape)

```go
type LiveSession interface {     // agent.LiveSession
    Send(req LiveRequest) error
    Close() error
}

type LiveRequest struct {
    RealtimeInput any   // *genai.Blob, *genai.ActivityStart, or *genai.ActivityEnd
    Content *genai.Content
}

type LiveRunConfig struct {
    ResponseModalities       []genai.Modality
    SpeechConfig             *genai.SpeechConfig
    InputAudioTranscription  *genai.AudioTranscriptionConfig
    OutputAudioTranscription *genai.AudioTranscriptionConfig
    RealtimeInputConfig      *genai.RealtimeInputConfig
    EnableAffectiveDialog    bool
    Proactivity              *genai.ProactivityConfig
    SessionResumption        *genai.SessionResumptionConfig
    SaveLiveBlob             bool
    MaxLLMCalls              int
}
```

## functiontool (assumed unchanged — not independently byte-diffed against v1 in this skill's research)

```go
type Config struct {
    Name                        string
    Description                 string
    InputSchema                 *jsonschema.Schema  // Auto-inferred if nil.
    OutputSchema                *jsonschema.Schema  // Auto-inferred if nil.
    IsLongRunning               bool
    RequireConfirmation         bool
    RequireConfirmationProvider any                 // func(toolInput T) bool
}

type Func[TArgs, TResults any] func(agent.Context, TArgs) (TResults, error)
func New[TArgs, TResults any](cfg Config, handler Func[TArgs, TResults]) (tool.Tool, error)

type StreamingFunc[TArgs any] func(agent.Context, TArgs) iter.Seq2[string, error]
func NewStreaming[TArgs any](cfg Config, handler StreamingFunc[TArgs]) (tool.Tool, error)
```

`TArgs` must be a struct or map (or pointer to one); primitives are rejected. Note the handler's context parameter type is `agent.Context` in v2 (was `tool.Context`, a v1-only alias, in v1).

## agenttool.Config (unchanged)

```go
func New(agent agent.Agent, cfg *Config) tool.Tool

type Config struct {
    SkipSummarization bool
}
```

## tool.WithConfirmation (still experimental in v2.2.0, same maturity as v1)

```go
type ConfirmationProvider func(toolName string, toolInput any) bool
func WithConfirmation(toolset Toolset, provider ConfirmationProvider) Toolset
var ErrConfirmationRequired error
var ErrConfirmationRejected error
```

## tool.Predicate and FilterToolset (unchanged)

```go
type Predicate func(ctx agent.ReadonlyContext, tool Tool) bool
func StringPredicate(allowedTools []string) Predicate
func FilterToolset(toolset Toolset, predicate Predicate) Toolset
```

## genai.GenerateContentConfig (Key Fields, unchanged)

```go
type GenerateContentConfig struct {
    SystemInstruction  *Content
    Temperature        *float32
    TopP               *float32
    TopK               *float32
    MaxOutputTokens    int32
    StopSequences      []string
    ResponseMIMEType   string
    ResponseSchema     *Schema
    SafetySettings     []*SafetySetting
    ThinkingConfig     *ThinkingConfig
    Seed               *int32
    CandidateCount     int32
    PresencePenalty    *float32
    FrequencyPenalty   *float32
}
```

Use `genai.Ptr[float32](0.7)` for pointer fields. Prefer `llmagent.Config.Instruction` over `GenerateContentConfig.SystemInstruction` — ADK manages system instructions through `Instruction`.

## Workflow Agent Configs (unchanged)

```go
// sequentialagent.Config / parallelagent.Config
type Config struct {
    AgentConfig agent.Config
}

// loopagent.Config
type Config struct {
    MaxIterations uint
    AgentConfig   agent.Config
}
```

For the new `workflow` package's `Node`/`Edge`/`Workflow` types, see `references/workflow.md`.

## remoteagent/v2 A2AConfig

```go
// import remoteagent "google.golang.org/adk/v2/agent/remoteagent/v2"

func NewA2A(cfg A2AConfig) (agent.Agent, error)
func NewAgentCardProvider(source string, opts ...agentcard.ResolveOption) AgentCardProvider
type AgentCardProvider func(ctx context.Context) (*a2a.AgentCard, error)

type A2AConfig struct {
    Name        string
    Description string

    AgentCard         *a2a.AgentCard
    AgentCardProvider AgentCardProvider

    BeforeAgentCallbacks   []agent.BeforeAgentCallback
    BeforeRequestCallbacks []BeforeA2ARequestCallback
    Converter              A2AEventConverter
    AfterRequestCallbacks  []AfterA2ARequestCallback
    AfterAgentCallbacks    []agent.AfterAgentCallback

    A2APartConverter   adka2a.A2APartConverter
    GenAIPartConverter adka2a.GenAIPartConverter

    ClientProvider    A2AClientProvider
    MessageSendConfig *a2a.SendMessageConfig

    RemoteTaskCleanupCallback A2ARemoteTaskCleanupCallback
}
```

Server side: `server/adka2a/v2` (the launcher's `a2a` mode uses it); `server/adkrest` embeds the REST API in existing services (has open bugs — see §session.Service and #1306 "CreateSessionHandler silently ignores chunked request bodies").

**Known gotcha (open, adk-go#1220):** outbound A2A messages from `remoteagent/v2` currently ignore `IsolationScope`.

## plugin.Config (unchanged)

```go
type Config struct {
    Name                    string
    OnUserMessageCallback   OnUserMessageCallback
    OnEventCallback         OnEventCallback
    BeforeRunCallback       BeforeRunCallback
    AfterRunCallback        AfterRunCallback
    BeforeAgentCallback     agent.BeforeAgentCallback
    AfterAgentCallback      agent.AfterAgentCallback
    BeforeModelCallback     llmagent.BeforeModelCallback
    AfterModelCallback      llmagent.AfterModelCallback
    OnModelErrorCallback    llmagent.OnModelErrorCallback
    BeforeToolCallback      llmagent.BeforeToolCallback
    AfterToolCallback       llmagent.AfterToolCallback
    OnToolErrorCallback     llmagent.OnToolErrorCallback
    CloseFunc               func() error
}
```

## artifact.Service (unchanged)

```go
type Service interface {
    Save(ctx context.Context, req *SaveRequest) (*SaveResponse, error)
    Load(ctx context.Context, req *LoadRequest) (*LoadResponse, error)
    Delete(ctx context.Context, req *DeleteRequest) error
    List(ctx context.Context, req *ListRequest) (*ListResponse, error)
    Versions(ctx context.Context, req *VersionsRequest) (*VersionsResponse, error)
    GetArtifactVersion(ctx context.Context, req *GetArtifactVersionRequest) (*GetArtifactVersionResponse, error)
}
func InMemoryService() Service
```

GCS-backed: `google.golang.org/adk/v2/artifact/gcsartifact`.

## memory.Service (unchanged)

```go
type Service interface {
    AddSessionToMemory(ctx context.Context, s session.Session) error
    SearchMemory(ctx context.Context, req *SearchRequest) (*SearchResponse, error)
}
func InMemoryService() Service
```

Vertex AI Memory Bank backend: `google.golang.org/adk/v2/memory/vertexai`.
```go
func NewService(ctx context.Context, config *ServiceConfig) (memory.Service, error)

type ServiceConfig struct {
    vertexaiutil.AgentEngineData
    StateKeySessionLastUpdateTime string
    WaitForCompletion             bool
}
```

## agent.Loader (unchanged)

```go
type Loader interface {
    ListAgents() []string
    LoadAgent(name string) (Agent, error)
    RootAgent() Agent
}
func NewSingleLoader(a Agent) Loader
func NewMultiLoader(root Agent, agents ...Agent) (Loader, error)
```

## launcher.Config (unchanged)

```go
type Config struct {
    SessionService   session.Service
    ArtifactService  artifact.Service
    MemoryService    memory.Service
    AgentLoader      agent.Loader
    A2AOptions       []a2asrv.RequestHandlerOption
    PluginConfig     runner.PluginConfig
    TelemetryOptions []telemetry.Option
}
```

Launchers: `full.NewLauncher()` (console + web UI + API + A2A, dev), `prod.NewLauncher()` (REST API + A2A only), plus `cmd/launcher/console`, `cmd/launcher/web`, `cmd/launcher/universal`, `cmd/launcher/agentengine`. Pub/Sub and Eventarc trigger sublaunchers under `cmd/launcher/web/triggers/{pubsub,eventarc}`.

**Security gotcha (open, adk-go#1154):** the web launcher mode binds the unauthenticated REST API and `/run_live` WebSocket to all network interfaces by default, with no WebSocket Origin allowlist. Don't run it unprotected on an untrusted network.

## telemetry (shape not independently re-verified in depth for v2; carried forward from v1)

```go
func New(ctx context.Context, opts ...Option) (*Providers, error)

type Providers struct {
    TracerProvider *sdktrace.TracerProvider
    LoggerProvider *sdklog.LoggerProvider
}
func (t *Providers) SetGlobalOtelProviders()
func (t *Providers) Shutdown(ctx context.Context) error
```

| Option | Purpose |
|---|---|
| `WithOtelToCloud(bool)` | Enable/disable export to GCP `telemetry.googleapis.com`. |
| `WithResource(*resource.Resource)` | Custom OTel resource (merged with defaults). |
| `WithGoogleCredentials(*google.Credentials)` | Override application default credentials. |
| `WithGcpResourceProject(string)` | Set `gcp.project_id` resource attribute. |
| `WithGcpQuotaProject(string)` | Set quota project for telemetry export. |
| `WithSpanProcessors(...sdktrace.SpanProcessor)` | Register additional span processors. |
| `WithLogRecordProcessors(...sdklog.Processor)` | Register additional log processors. |
| `WithTracerProvider(*sdktrace.TracerProvider)` | Override the default TracerProvider. |
| `WithLoggerProvider(*sdklog.LoggerProvider)` | Override the default LoggerProvider. |
| `WithGenAICaptureMessageContent(bool)` | Log message content. |

## Plugin Callback Types (unchanged)

```go
type OnUserMessageCallback func(agent.InvocationContext, *genai.Content) (*genai.Content, error)
type BeforeRunCallback func(agent.InvocationContext) (*genai.Content, error)
type AfterRunCallback func(agent.InvocationContext)
type OnEventCallback func(agent.InvocationContext, *session.Event) (*session.Event, error)
```

## Built-in Plugin Configs

### retryandreflect (unchanged)

```go
func New(opts ...PluginOption) (*plugin.Plugin, error)
func MustNew(opts ...PluginOption) *plugin.Plugin

type TrackingScope string
const (
    Invocation TrackingScope = "invocation"
    Global     TrackingScope = "global"
)

func WithMaxRetries(maxRetries int) PluginOption         // Default: 3.
func WithErrorIfRetryExceeded(b bool) PluginOption       // Default: false.
func WithTrackingScope(scope TrackingScope) PluginOption  // Default: Invocation.
```

### functioncallmodifier (unchanged)

```go
func NewPlugin(cfg FunctionCallModifierConfig) (*plugin.Plugin, error)
func MustNewPlugin(cfg FunctionCallModifierConfig) *plugin.Plugin

type FunctionCallModifierConfig struct {
    Predicate           func(toolName string) bool
    Args                map[string]*genai.Schema
    OverrideDescription func(original string) string
}
```

### loggingplugin (unchanged)

```go
func New(name string) (*plugin.Plugin, error)
func MustNew(name string) *plugin.Plugin
```

### BigQuery Agent Analytics Plugin (v2, new in v2.1.0)

Exact import path not independently confirmed in this skill's research; likely under `plugin/`. **Known bug (open, adk-go#1210):** logs one `MODEL_RESPONSE` row per streamed chunk instead of once per turn — expect duplicated usage-metadata rows if enabled with streaming. Tracking issue #1325 notes the Go plugin still trails feature/bug-fix parity with the Python equivalent.

## tool/exampletool (unchanged)

```go
type Example struct {
    Input  *genai.Content   `json:"input"`
    Output []*genai.Content `json:"output"`
}
type ExampleToolConfig struct {
    Examples []*Example
}
func New(config ExampleToolConfig) (tool.Tool, error)
```

## tool/skilltoolset (unchanged)

```go
// import "google.golang.org/adk/v2/tool/skilltoolset"
//        "google.golang.org/adk/v2/tool/skilltoolset/skill"

type Config struct {
    Source            skill.Source
    Name              string
    SystemInstruction string
}
func New(ctx context.Context, cfg Config) (*SkillToolset, error)
```

## model/apigee (unchanged)

```go
func NewModel(ctx context.Context, modelName string, opts ...Option) (model.LLM, error)
// Options: WithProxyURL(string), WithCustomHeaders(http.Header), WithHTTPClient(*http.Client)
```

## util/instructionutil (unchanged)

```go
func InjectSessionState(ctx agent.ReadonlyContext, template string) (string, error)
```
