# Integrations

MCP server integration, the new Agent Registry and `auth` packages, OpenAI/custom model providers, Apigee proxying, and Vertex AI configuration for ADK Go v2.

## MCP Toolset

Connect ADK agents to MCP (Model Context Protocol) servers using the official Go MCP SDK (`github.com/modelcontextprotocol/go-sdk`, currently pinned around v1.7.0 by ADK Go's own `go.mod` — check your target ADK Go version's `go.mod` rather than hardcoding this). Do **not** use the third-party `mark3labs/mcp-go`.

```go
import (
    "github.com/modelcontextprotocol/go-sdk/mcp"
    "google.golang.org/adk/v2/tool/mcptoolset"
)
```

### Transport Types

| Transport | Use Case | Import |
|---|---|---|
| `mcp.CommandTransport` | Local subprocess (stdio) | `os/exec` |
| `mcp.StreamableClientTransport` | Remote HTTPS server (preferred) | - |
| In-memory | Testing / in-process | `mcp.NewInMemoryTransports()` |

### Stdio Transport (Local Process)

```go
import "os/exec"

mcpTools, err := mcptoolset.New(mcptoolset.Config{
    Transport: &mcp.CommandTransport{
        Command: exec.Command("npx", "-y", "@modelcontextprotocol/server-filesystem", "/tmp"),
    },
})

agent, err := llmagent.New(llmagent.Config{
    Name:     "fs_agent",
    Model:    model,
    Toolsets: []tool.Toolset{mcpTools},
})
```

### Streamable HTTP Transport (Remote Server)

```go
mcpTools, err := mcptoolset.New(mcptoolset.Config{
    Transport: &mcp.StreamableClientTransport{
        Endpoint: "https://my-mcp-server.example.com/mcp/",
    },
})
```

### Authenticating MCP Servers — two ways

**(a) Ad-hoc, via the HTTP client (works in v1 and v2):**
```go
import "golang.org/x/oauth2"

ts := oauth2.StaticTokenSource(&oauth2.Token{AccessToken: os.Getenv("GITHUB_PAT")})
mcpTools, err := mcptoolset.New(mcptoolset.Config{
    Transport: &mcp.StreamableClientTransport{
        Endpoint:   "https://api.githubcopilot.com/mcp/",
        HTTPClient: oauth2.NewClient(ctx, ts),
    },
})
```

**(b) Structured, via `mcptoolset.Config.Auth` (new in v2) and the `auth` package:**
```go
import "google.golang.org/adk/v2/auth"

mcpTools, err := mcptoolset.New(mcptoolset.Config{
    Transport: &mcp.StreamableClientTransport{Endpoint: "https://internal-mcp.example.com/mcp/"},
    Auth:      auth.StaticToken(os.Getenv("MCP_TOKEN")),
})
```
The `auth` package resolves credentials fresh on every request rather than caching a single shared client-wide token — this matters for multi-tenant agents where different users must never share a token. See §Auth below for the full credential surface.

### In-Memory Transport (Testing)

```go
type WeatherInput struct {
    City string `json:"city" jsonschema:"city name"`
}
type WeatherOutput struct {
    Summary string `json:"weather_summary"`
}

func GetWeather(ctx context.Context, req *mcp.CallToolRequest, input WeatherInput) (*mcp.CallToolResult, WeatherOutput, error) {
    return nil, WeatherOutput{Summary: "Sunny in " + input.City}, nil
}

clientTransport, serverTransport := mcp.NewInMemoryTransports()

server := mcp.NewServer(&mcp.Implementation{Name: "weather", Version: "v1.0.0"}, nil)
mcp.AddTool(server, &mcp.Tool{
    Name:        "get_weather",
    Description: "Gets weather for a city.",
}, GetWeather)
server.Connect(ctx, serverTransport, nil)

mcpTools, err := mcptoolset.New(mcptoolset.Config{
    Transport: clientTransport,
})
```

### Tool Filtering

```go
filtered := tool.FilterToolset(mcpTools, tool.StringPredicate([]string{
    "get_weather",
    "get_forecast",
}))

agent, err := llmagent.New(llmagent.Config{
    Toolsets: []tool.Toolset{filtered},
})
```

**Gap (open, adk-go#1126):** no built-in tool-name prefixing when composing multiple MCP servers into one toolset (adk-python has this — Go doesn't yet). If two servers expose a tool with the same name, you must disambiguate yourself (e.g. wrap and rename, or don't combine them into one toolset).

### Human-in-the-Loop Confirmation

```go
mcpTools, err := mcptoolset.New(mcptoolset.Config{
    Transport:           transport,
    RequireConfirmation: true,  // All tools require confirmation
})

// Or dynamic confirmation per tool:
mcpTools, err := mcptoolset.New(mcptoolset.Config{
    Transport: transport,
    RequireConfirmationProvider: func(toolName string, args any) bool {
        return toolName == "delete_file"
    },
})
```

### Generic Toolset Confirmation (`tool.WithConfirmation`)

```go
import "google.golang.org/adk/v2/tool"

confirmed := tool.WithConfirmation(myToolset, func(toolName string, toolInput any) bool {
    return toolName == "delete_record"
})

agent, err := llmagent.New(llmagent.Config{
    Name:     "safe_agent",
    Model:    model,
    Toolsets: []tool.Toolset{confirmed},
})
```

Inside a tool function, use `ctx.ToolConfirmation()` (on the unified `agent.Context`) to check status and `ctx.RequestConfirmation(hint, payload)` to trigger the approval flow. **Still marked EXPERIMENTAL at v2.2.0** — same maturity level as v1, excluded from the stability guarantee.

For pausing an *entire workflow run* (not just one tool call) for human input, durably, use the `workflow` package's `ResumeOrRequestInput` instead — see `references/workflow.md`.

### mcptoolset.Config

```go
type Config struct {
    Client                      *mcp.Client                // Optional custom MCP client.
    Transport                   mcp.Transport              // Required.
    Auth                        auth.CredentialProvider    // (v2) Structured per-request auth.
    ToolFilter                  tool.Predicate             // Deprecated: use tool.FilterToolset instead.
    RequireConfirmation         bool
    RequireConfirmationProvider tool.ConfirmationProvider
}
```

**Known bugs (open):**
- **#1352:** successful tool executions with an empty text response are erroneously treated as errors.
- **#1319:** `loadartifactstool`'s `load_artifacts` is ignored when preceded by other tool responses in multi-part content.
- **#1165:** MCP tool result `_meta` is discarded, blocking auth-challenge flows that rely on it (e.g. SEP-1036 URL elicitation).
- **#1348:** `preloadmemorytool` nil-pointer panics on nil parts in a `memory.Entry`.

**Behavior notes (carried forward from v1, not contradicted by v2 research):**
- MCP sessions are created lazily on first LLM request.
- Automatic reconnection on `mcp.ErrConnectionClosed`, `mcp.ErrSessionMissing`, `io.ErrClosedPipe`, `io.EOF`.
- Tool discovery happens via `ListTools()` with pagination.
- For HTTP-based MCP servers with idle timeouts, the cached `*mcp.ClientSession` may go stale between requests; mitigate by tuning server-side timeouts or recreating the toolset on connection errors.

## Agent Registry (new in v2.1.0)

`google.golang.org/adk/v2/agentregistry` is a client for Google Cloud's Agent Registry (`agentregistry.googleapis.com`) — a governed catalog of A2A agents, MCP servers, and model endpoints. Use it to discover and directly instantiate remote resources by name instead of hardcoding URLs/cards in your own code.

```go
import "google.golang.org/adk/v2/agentregistry"

client, err := agentregistry.New(ctx, agentregistry.Config{ /* project/location/etc */ })

// Discovery
a, err := client.GetAgent(ctx, "billing-agent")
page, err := client.ListAgents(ctx, agentregistry.WithFilter("state=ACTIVE"), agentregistry.WithPageSize(50))
for a, err := range client.AllAgents(ctx) { /* iterate all pages */ }
// Parallel Get/List/All exist for MCPServer and Endpoint types.

// Direct instantiation — skips manually wiring remoteagent.NewA2A/mcptoolset.New
remoteAgent, err := client.RemoteAgent(ctx, "billing-agent",
    agentregistry.WithA2AHTTPClient(customClient),
    agentregistry.WithA2AHeaders(headers),
)
mcpTools, err := client.MCPToolset(ctx, "internal-search",
    agentregistry.WithMCPHTTPClient(customClient),
    agentregistry.WithMCPHeaders(headers),
)
```

Key types: `Agent`, `MCPServer`, `Endpoint`, `Protocol`, `Interface`, `Skill`, `Tool`, `Annotations`, `Card`, `APIError`. Options include `WithFilter`, `WithPageSize`, `WithPageToken`, `WithA2AHTTPClient`, `WithA2AHeaders`, `WithMCPHTTPClient`, `WithMCPHeaders`.

Use this when your organization already curates agents/MCP servers/model endpoints centrally in Agent Registry; otherwise, wiring `remoteagent.NewA2A`/`mcptoolset.New` directly (as shown above and in `references/orchestration.md`) is simpler for one-off integrations.

## Auth (new in v2.1.0)

`google.golang.org/adk/v2/auth` is a lightweight credential/authentication layer, wired into `mcptoolset.Config.Auth` and usable anywhere you need per-request credentials for an outbound call. It delegates actual token refresh to `golang.org/x/oauth2` rather than reimplementing OAuth flows.

```go
type Credential interface { Apply(h http.Header) error }
// Concrete credential types: BearerCredential, BasicCredential, APIKeyCredential, OAuth2Credential

type CredentialProvider interface { Credential(ctx context.Context) (Credential, error) }
// Concrete providers: StaticToken, APIKey, TokenSourceProvider, ADC, ServiceAccount, ProviderFunc

type Transport struct{ /* http.RoundTripper wrapper that applies a CredentialProvider */ }

func WithHeaders(inner Credential, headers map[string]string) Credential

type ServiceAccountConfig struct {
    JSONKey  []byte
    Scopes   []string
    Audience string
}
```

The resolver runs **on every request**, not once at construction — this is deliberate so per-user credentials are never accidentally cached/shared across users in a multi-tenant agent. Prefer a `CredentialProvider` (e.g. `auth.ProviderFunc` wrapping your own per-user token lookup) over a single `auth.StaticToken` whenever different end users of your agent should authenticate as themselves against a downstream MCP server or API.

## OpenAI Integration (new in v2.1.0: `model/openaimodel`)

```go
import "google.golang.org/adk/v2/model/openaimodel"

m, err := openaimodel.NewModel(ctx, "gpt-4o", openaimodel.Config{
    APIKey: os.Getenv("OPENAI_API_KEY"),
})
```

**Known bug (open, adk-go#1197):** assistant messages are sent with an `input_text` content-type part instead of `output_text` on multi-turn requests, causing a 400 from the OpenAI API. Test multi-turn conversations explicitly before relying on this backend; single-turn use may work fine. There is still no native Anthropic backend (tracked by adk-go#1097, "V1/V2 Parity and OpenAI/Anthropic Endpoint Support").

The underlying SDK is `github.com/openai/openai-go/v3`:
```go
import (
    "github.com/openai/openai-go/v3"
    "github.com/openai/openai-go/v3/option"
)

client := openai.NewClient(option.WithAPIKey(os.Getenv("OPENAI_API_KEY")))

// Azure OpenAI:
client := openai.NewClient(
    option.WithBaseURL("https://my-resource.openai.azure.com/openai"),
    option.WithAPIKey(os.Getenv("AZURE_OPENAI_KEY")),
)
```

### Writing a Custom model.LLM Provider (for Anthropic or anything else not built in)

```go
package myprovider

import (
    "context"
    "iter"

    "google.golang.org/adk/v2/model"
    "google.golang.org/genai"
)

type MyModel struct {
    name string
}

func NewModel(name string) (model.LLM, error) {
    return &MyModel{name: name}, nil
}

func (m *MyModel) Name() string { return m.name }

func (m *MyModel) GenerateContent(
    ctx context.Context,
    req *model.LLMRequest,
    stream bool,
) iter.Seq2[*model.LLMResponse, error] {
    return func(yield func(*model.LLMResponse, error) bool) {
        // 1. Convert req.Contents ([]*genai.Content) to your provider's format
        // 2. Convert req.Config (*genai.GenerateContentConfig) for temperature, etc.
        // 3. Convert req.Tools to your provider's tool/function format
        // 4. Call your LLM API
        // 5. Convert response back to model.LLMResponse

        yield(&model.LLMResponse{
            Content: &genai.Content{
                Parts: []*genai.Part{genai.NewPartFromText("Hello!")},
                Role:  "model",
            },
            TurnComplete: true,
        }, nil)

        // For streaming: yield(&model.LLMResponse{Content: chunk, Partial: true}, nil) per chunk,
        // then a final yield with TurnComplete: true.
    }
}
```

Register it for name-based resolution (new in v2.1.0) if you want other code to construct it by string name:
```go
model.Register("myprovider-*", func(name string) (model.LLM, error) { return myprovider.NewModel(name) })
```

**Key conversion requirements:**

| ADK Type | You Must Handle |
|---|---|
| `req.Contents` | Convert `[]*genai.Content` (Parts: text, function calls, function responses) to your format |
| `req.Config` | Map `Temperature`, `MaxOutputTokens`, `ResponseMIMEType`, etc. |
| `req.Tools` | Convert `genai.FunctionDeclaration` tool schemas to your provider's format |
| Response `Content` | Convert your provider's response to `*genai.Content` with appropriate `Parts` |
| Function calls | Map your provider's tool calls to `genai.FunctionCall` parts |
| Streaming | Set `Partial: true` for intermediate chunks, `TurnComplete: true` for final |

## Gemini / Vertex AI Configuration

The official Go quickstart (2026-08) uses the rolling alias **`gemini-flash-latest`**, not a dated snapshot — prefer it unless you have a specific reason to pin an exact model version.

### Gemini API (Default)

```go
model, err := gemini.NewModel(ctx, "gemini-flash-latest", &genai.ClientConfig{
    APIKey: os.Getenv("GOOGLE_API_KEY"),
})
```

Environment variables: `GOOGLE_API_KEY` or `GEMINI_API_KEY`.

### Vertex AI

```go
model, err := gemini.NewModel(ctx, "gemini-flash-latest", &genai.ClientConfig{
    Project:  "my-gcp-project",
    Location: "us-central1",
    Backend:  genai.BackendVertexAI,
})
```

Environment variables: `GOOGLE_GENAI_USE_VERTEXAI=true`, `GOOGLE_CLOUD_PROJECT=my-gcp-project`, `GOOGLE_CLOUD_LOCATION=us-central1`.

### Auto-Detection

```go
model, err := gemini.NewModel(ctx, "gemini-flash-latest", &genai.ClientConfig{})
```

### genai.ClientConfig

```go
type ClientConfig struct {
    APIKey      string
    Project     string
    Location    string
    Backend     Backend    // BackendGeminiAPI (default) or BackendVertexAI
    HTTPOptions *HTTPOptions
}
```

### Apigee Proxy (`model/apigee`)

```go
import "google.golang.org/adk/v2/model/apigee"

model, err := apigee.NewModel(ctx, "gemini-flash-latest",
    apigee.WithProxyURL("https://my-apigee-host/v1/gemini"),
    apigee.WithCustomHeaders(http.Header{"x-api-key": []string{os.Getenv("APIGEE_KEY")}}),
)
```

## Service Backends

| Service | In-memory | Production backends |
|---|---|---|
| Sessions | `session.InMemoryService()` | `session/database` (GORM dialectors: Postgres, SQLite, ...; run `database.AutoMigrate`), `session/vertexai` (Agent Engine sessions) |
| Memory | `memory.InMemoryService()` | `memory/vertexai` (Vertex AI Memory Bank) |
| Artifacts | `artifact.InMemoryService()` | `artifact/gcsartifact` (Google Cloud Storage) |

All under `google.golang.org/adk/v2/...`. See `references/api-reference.md` for constructor signatures and known open bugs against `session/database` and `session/inmemory`.

## Key Dependencies

Versions drift across ADK Go patch releases — check the target version's own `go.mod` rather than hardcoding these in your project's reasoning. As observed on the `main` branch, 2026-08-17 (post-v2.2.0):

```
google.golang.org/adk/v2                            # ADK core
google.golang.org/genai v1.66.0                     # drifted from v1.63.0 at the v2.1.0 tag — re-check
github.com/modelcontextprotocol/go-sdk v1.7.0        # official MCP SDK; NOT mark3labs/mcp-go
github.com/openai/openai-go/v3 v3.49.0               # OpenAI SDK, backs model/openaimodel
```

`github.com/a2aproject/a2a-go` version was not independently confirmed via a direct `go.mod` fetch in this skill's research (WebSearch-only claim of v0.3.15, with a note that ADK was updated to work against an `a2a-go/v2`); re-verify directly before citing a specific number.

## `agents-cli` (Python scaffolding tool) — not applicable here

Google also ships a separate, Python-only CLI called `agents-cli` (`uv tool install google-agents-cli`) with its own skills (`google-agents-cli-scaffold`, `-adk-code`, `-deploy`, `-eval`, `-observability`, `-publish`). It scaffolds and deploys **Python** ADK agents; confirmed via its own docs (explicit "Python 3.11+" prerequisite, no `--language` flag, no Go templates) that it has no Go support and no announced plans for it. For Go, use `go get google.golang.org/adk/v2` directly plus the ADK-Go-repo-bundled `cmd/adkgo` CLI for deploy/test (`adkgo deploy cloudrun`, `adkgo deploy agentengine`) — a separate, smaller tool from `agents-cli`. Don't reach for `agents-cli` skills/commands when working in a Go ADK project.
