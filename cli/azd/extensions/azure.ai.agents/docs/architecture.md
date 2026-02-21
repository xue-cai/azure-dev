# Azure AI Agents Extension — Technical Architecture

This document provides a deep-dive into how the `azd ai agent` extension is built, how it
integrates with the Azure Developer CLI (`azd`), and how it deploys agents to the
Azure AI Foundry Agent Service.

---

## 1. Extension Overview

The `azure.ai.agents` extension lives at `cli/azd/extensions/azure.ai.agents/`. It is a
**standalone Go binary** that communicates with the `azd` host process over **gRPC**. The
extension manifest (`extension.yaml`) declares five capabilities:

```yaml
# extension.yaml
id: azure.ai.agents
namespace: ai.agent
capabilities:
  - custom-commands      # azd ai agent init
  - lifecycle-events     # preprovision / predeploy / postdeploy hooks
  - mcp-server           # MCP (Model Context Protocol) tool server
  - service-target-provider  # azure.ai.agent host type for azd services
  - metadata             # exposes command metadata to azd
```

When a user runs `azd ai agent <command>`, `azd` core starts the extension binary and
routes the subcommand. When the extension is _listening_ (during `azd provision` / `azd deploy`),
`azd` core connects over gRPC and dispatches lifecycle events.

---

## 2. Entry Point & Command Tree

### `main.go`

```go
// main.go
func main() {
    ctx := azdext.NewContext()           // OpenTelemetry trace context from env
    rootCmd := cmd.NewRootCommand()      // Cobra command tree
    if err := rootCmd.ExecuteContext(ctx); err != nil {
        color.Red("Error: %v", err)
        os.Exit(1)
    }
}
```

`azdext.NewContext()` creates a background context enriched with trace propagation headers
(`TRACEPARENT`, `TRACESTATE`) passed by `azd` via environment variables, enabling
distributed tracing across the host–extension boundary.

### Root Command (`internal/cmd/root.go`)

The root Cobra command registers five subcommands:

| Subcommand | Visibility | Purpose |
|------------|-----------|---------|
| `listen`   | Hidden    | gRPC event listener — the extension's "daemon mode" |
| `version`  | Visible   | Prints build version/commit/date |
| `init`     | Visible   | Interactive project initialization wizard |
| `mcp`      | Hidden    | MCP server commands (`mcp start`) |
| `metadata` | Hidden    | Generates JSON command metadata for `azd` |

```go
// internal/cmd/root.go
func NewRootCommand() *cobra.Command {
    rootCmd := &cobra.Command{
        Use:   "agent <command> [options]",
        Short: "Extension for the Foundry Agent Service. (Preview)",
    }
    rootCmd.AddCommand(newListenCommand())
    rootCmd.AddCommand(newVersionCommand())
    rootCmd.AddCommand(newInitCommand(&rootFlags))
    rootCmd.AddCommand(newMcpCommand())
    rootCmd.AddCommand(newMetadataCommand())
    return rootCmd
}
```

---

## 3. Extension Host & gRPC Communication (`listen` command)

The `listen` command is the heart of the extension's integration with `azd`. It runs during
`azd provision`, `azd deploy`, and other lifecycle operations.

### How it works

```go
// internal/cmd/listen.go
func newListenCommand() *cobra.Command {
    return &cobra.Command{
        Use:    "listen",
        Hidden: true,
        RunE: func(cmd *cobra.Command, args []string) error {
            ctx := azdext.WithAccessToken(cmd.Context())

            azdClient, err := azdext.NewAzdClient()  // gRPC client → azd host
            defer azdClient.Close()

            projectParser := &project.FoundryParser{AzdClient: azdClient}

            host := azdext.NewExtensionHost(azdClient).
                WithServiceTarget("azure.ai.agent", func() azdext.ServiceTargetProvider {
                    return project.NewAgentServiceTargetProvider(azdClient)
                }).
                WithProjectEventHandler("preprovision", func(ctx context.Context, args *azdext.ProjectEventArgs) error {
                    return preprovisionHandler(ctx, azdClient, projectParser, args)
                }).
                WithProjectEventHandler("predeploy", func(ctx context.Context, args *azdext.ProjectEventArgs) error {
                    return predeployHandler(ctx, azdClient, projectParser, args)
                }).
                WithProjectEventHandler("postdeploy", projectParser.CoboPostDeploy)

            return host.Run(ctx)  // Blocking — listens for gRPC events
        },
    }
}
```

**Key flow:**

1. `azdext.NewAzdClient()` connects to `azd` via gRPC (address read from `AZD_SERVER` env var).
2. `azdext.NewExtensionHost(azdClient)` creates the extension host with chained registrations:
   - **Service target**: `"azure.ai.agent"` — handles Package/Publish/Deploy operations for services with `host: azure.ai.agent` in `azure.yaml`.
   - **Project event handlers**: `preprovision`, `predeploy`, `postdeploy` — run before/after provisioning and deployment.
3. `host.Run(ctx)` starts background goroutines for each manager (service target, events),
   registers them with `azd` core via gRPC, signals readiness, and blocks until shutdown.

### Extension Host `Run()` Lifecycle (from `azdext` framework)

```
1. Start receiver goroutines for each manager → broker.Run(ctx)
2. Wait for all managers to report Ready()
3. Register components with azd core via SendAndWait()
4. Signal Ready() to azd core
5. Block on select { ctx.Done(), receiver errors, completion }
```

The receiver goroutines must start **before** registration so they can process the registration
response messages that come back over the same gRPC stream.

---

## 4. Service Target Provider — Agent Deployment Pipeline

The `AgentServiceTargetProvider` implements the `azdext.ServiceTargetProvider` interface,
which provides the full deployment pipeline for `azure.ai.agent` host services.

### Interface Methods

```go
// azdext.ServiceTargetProvider interface
type ServiceTargetProvider interface {
    Initialize(ctx, serviceConfig)
    GetTargetResource(ctx, subscriptionId, serviceConfig, defaultResolver) → TargetResource
    Endpoints(ctx, serviceConfig, targetResource) → []string
    Package(ctx, serviceConfig, serviceContext, progress) → ServicePackageResult
    Publish(ctx, serviceConfig, serviceContext, targetResource, publishOptions, progress) → ServicePublishResult
    Deploy(ctx, serviceConfig, serviceContext, targetResource, progress) → ServiceDeployResult
}
```

### Initialize

Looks up the `agent.yaml` / `agent.yml` file in the service directory, retrieves Azure credentials
via `azidentity.NewAzureDeveloperCLICredential`, and resolves the AI Foundry project resource ID:

```go
// internal/project/service_target_agent.go
func (p *AgentServiceTargetProvider) Initialize(ctx context.Context, serviceConfig *azdext.ServiceConfig) error {
    // 1. Get project path from azdClient
    // 2. Get Azure subscription + tenant via environment variables
    // 3. Create Azure credential (AzureDeveloperCLICredential)
    // 4. Locate agent.yaml in service directory (or AGENT_DEFINITION_PATH env var)
    // 5. Parse AI Foundry project resource ID from environment
}
```

### Deploy — Two Agent Types

The `Deploy` method reads `agent.yaml`, determines the agent kind, and dispatches:

```go
func (p *AgentServiceTargetProvider) Deploy(ctx, serviceConfig, serviceContext, targetResource, progress) {
    data, _ := os.ReadFile(p.agentDefinitionPath)
    // ... validate and parse ...

    switch kind {
    case "prompt":
        return p.deployPromptAgent(ctx, serviceConfig, agentDef, azdEnv)
    case "hosted":
        return p.deployHostedAgent(ctx, serviceConfig, serviceContext, progress, agentDef, azdEnv)
    }
}
```

#### Prompt Agent Deployment

For `kind: prompt` agents (serverless, model-based):

```
1. Parse agent.yaml → agent_yaml.PromptAgent
2. Create API request via agent_yaml.CreateAgentAPIRequestFromDefinition()
3. Call agent_api.AgentClient.CreateAgentVersion()  (REST API → Foundry)
4. Register AGENT_<SVC>_NAME / AGENT_<SVC>_VERSION env vars
5. Return playground URL + agent endpoint as artifacts
```

#### Hosted Agent Deployment

For `kind: hosted` agents (containerized):

```
1. Parse agent.yaml → agent_yaml.ContainerAgent
2. Find published container image from serviceContext.Publish artifacts
3. Resolve environment variables (${VAR} substitution from azd env)
4. Create API request with image URL, CPU, memory, protocols, env vars
5. Call agent_api.AgentClient.CreateAgentVersion()
6. Call agent_api.AgentClient.StartAgentContainer() → long-running operation
7. Poll GetAgentContainerOperation() every 5s until Succeeded/Failed (max 10 min)
8. Register env vars and return deployment artifacts
```

The `Package` and `Publish` methods delegate container building/packaging to
`azdClient.Container().Build/Package/Publish()` — the standard `azd` container pipeline.

---

## 5. Agent Definition Schema (`agent.yaml`)

The `agent.yaml` file in each service directory defines the agent. It is parsed by
`internal/pkg/agents/agent_yaml/`.

### Core Types

```go
// agent_yaml/yaml.go — AgentDefinition (base)
type AgentDefinition struct {
    Kind         AgentKind               // "prompt" | "hosted" | "workflow"
    Name         string
    DisplayName  *string
    Description  *string
    Metadata     *map[string]interface{}
    InputSchema  *PropertySchema
    OutputSchema *PropertySchema
}

// PromptAgent — prompt-based agent
type PromptAgent struct {
    AgentDefinition
    Model        Model
    Tools        *[]any      // FunctionTool, McpTool, BingGroundingTool, etc.
    Template     *Template
    Instructions *string
}

// ContainerAgent — hosted/containerized agent
type ContainerAgent struct {
    AgentDefinition
    Protocols            []ProtocolVersionRecord
    EnvironmentVariables *[]EnvironmentVariable
}
```

### Example `agent.yaml` (hosted)

```yaml
kind: hosted
name: my-agent
description: A containerized AI agent
protocols:
  - protocol: responses
    version: v1
environmentVariables:
  - name: MY_VAR
    value: ${AZURE_AI_PROJECT_ENDPOINT}
```

### Validation (`agent_yaml/parse.go`)

`ValidateAgentDefinition()` performs structural validation:
- Agent kind must be one of `prompt`, `hosted`, `workflow`
- Agent name must match `^[a-zA-Z0-9]([a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?$`
- Prompt agents require `model.id`
- Kind-specific unmarshaling and field validation

---

## 6. YAML-to-API Mapping (`agent_yaml/map.go`)

The mapping layer converts `agent.yaml` types into Azure AI Agent Service REST API request types.

### Builder Pattern

```go
// Build options for agent creation
request, err := agent_yaml.CreateAgentAPIRequestFromDefinition(
    agentDef,                              // agent_yaml.ContainerAgent
    agent_yaml.WithImageURL(fullImageURL), // Container image
    agent_yaml.WithCPU("1"),
    agent_yaml.WithMemory("2Gi"),
    agent_yaml.WithEnvironmentVariables(resolvedEnvVars),
)
```

Internally, `CreateAgentAPIRequestFromDefinition` routes by kind:

```go
switch agentDef.Kind {
case AgentKindPrompt:
    return CreatePromptAgentAPIRequest(promptDef, buildConfig)
case AgentKindHosted:
    return CreateHostedAgentAPIRequest(hostedDef, buildConfig)
}
```

### Hosted Agent → API Mapping

```go
func CreateHostedAgentAPIRequest(hostedAgent ContainerAgent, buildConfig *AgentBuildConfig) {
    // Map protocol versions (default: responses/v1)
    // Set CPU, memory, env vars from buildConfig
    // Create ImageBasedHostedAgentDefinition with container image URL
    // Wrap in CreateAgentRequest with name, description, metadata
}
```

### Tool Type Mapping (Prompt Agents)

YAML tool kinds map to API tool types:

| YAML `ToolKind`    | API `ToolType`                |
|--------------------|-------------------------------|
| `function`         | `function`                    |
| `webSearch`        | `web_search_preview`          |
| `bingGrounding`    | `bing_grounding`              |
| `fileSearch`       | `file_search`                 |
| `mcp`              | `mcp`                         |
| `openApi`          | `openapi`                     |
| `codeInterpreter`  | `code_interpreter`            |

---

## 7. Agent REST API Client (`agent_api/`)

`agent_api.AgentClient` wraps the Azure AI Agent Service REST API.

### Client Setup

```go
// agent_api/operations.go
func NewAgentClient(endpoint string, cred azcore.TokenCredential) *AgentClient {
    // Bearer token policy: scope "https://ai.azure.com/.default"
    // Custom policies: correlation headers, user agent
    // Azure SDK pipeline for HTTP execution
}
```

### Key Operations

| Method | HTTP | Endpoint |
|--------|------|----------|
| `CreateAgent` | POST | `/agents` |
| `GetAgent` | GET | `/agents/{name}` |
| `UpdateAgent` | POST | `/agents/{name}` |
| `DeleteAgent` | DELETE | `/agents/{name}` |
| `ListAgents` | GET | `/agents` |
| `CreateAgentVersion` | POST | `/agents/{name}/versions` |
| `GetAgentVersion` | GET | `/agents/{name}/versions/{version}` |
| `StartAgentContainer` | POST | `/agents/{name}/versions/{version}/containers/default:start` |
| `UpdateAgentContainer` | POST | `/agents/{name}/versions/{version}/containers/default:update` |
| `StopAgentContainer` | POST | `/agents/{name}/versions/{version}/containers/default:stop` |
| `GetAgentContainer` | GET | `/agents/{name}/versions/{version}/containers/default` |
| `GetAgentContainerOperation` | GET | `/agents/{name}/operations/{id}` |
| `CreateOrUpdateAgentEventHandler` | POST | `/agents/{name}/event_handlers/{handler}` |

All operations use `api-version=2025-05-15-preview`.

---

## 8. Lifecycle Event Handlers

The extension registers three project-level event handlers that run during `azd provision`
and `azd deploy`:

### `preprovision`

```go
func preprovisionHandler(ctx, azdClient, projectParser, args) error {
    projectParser.SetIdentity(ctx, args)   // Assign managed identity

    for _, svc := range args.Project.Services {
        switch svc.Host {
        case "azure.ai.agent":
            populateContainerSettings(ctx, azdClient, svc)  // Set CPU/memory/scale defaults
            envUpdate(ctx, azdClient, args.Project, svc)     // Set env vars for infra
        case "containerapp":
            containerAgentHandling(ctx, azdClient, args.Project, svc)  // Enable container agents
        }
    }
}
```

**`populateContainerSettings`** fills default resource limits:
- CPU: `"1"`, Memory: `"2Gi"`, MinReplicas: `1`, MaxReplicas: `3`

**`envUpdate`** sets environment variables consumed by Bicep/Terraform:
- `ENABLE_HOSTED_AGENTS=true` (for hosted agents)
- `AI_PROJECT_DEPLOYMENTS` (JSON array of model deployments)
- `AI_PROJECT_DEPENDENT_RESOURCES` (JSON array of external resources)

### `predeploy`

Runs the same `SetIdentity` + `populateContainerSettings` + `envUpdate` logic for
`azure.ai.agent` services to ensure environment is up-to-date before deployment.

### `postdeploy`

Handled by `projectParser.CoboPostDeploy` — performs post-deployment tasks for
container-based agents (defined in `parser.go`).

---

## 9. `init` Command — Project Initialization

`azd ai agent init` is a rich interactive wizard that:

1. **Connects to an AI Foundry project** (via `--project-id` flag or interactive selection)
2. **Selects an agent source**: local `agent.yaml`, GitHub URL, or agent registry catalog
3. **Configures model deployments** from the project's model catalog
4. **Updates `azure.yaml`** with agent service configuration
5. **Generates infrastructure** (Bicep templates) if needed

Key flags:

```go
type initFlags struct {
    projectResourceId string  // --project-id — AI Foundry project resource ID
    manifestPointer   string  // --from — agent manifest source (URL, file, or registry ref)
    src               string  // --src — agent source directory
    host              string  // --host — service host type (azure.ai.agent or containerapp)
    env               string  // --env — azd environment name
}
```

---

## 10. MCP Server

The extension also functions as an MCP (Model Context Protocol) server, enabling
AI coding tools to interact with the extension programmatically.

```go
// internal/cmd/mcp.go
func runMcpServer(ctx context.Context) error {
    s := server.NewMCPServer("AZD Microsoft Foundry Agents Extension MCP Server", "1.0.0")
    s.AddTools(tools.NewAddAgentTool())
    return server.ServeStdio(s)  // stdio transport
}
```

Currently exposes one tool: **`add_agent`** — adds a Foundry agent to the project from a
manifest file path or URL.

---

## 11. Azure SDK Clients

### Foundry Projects Client (`pkg/azure/foundry_projects_client.go`)

Talks to the AI Foundry project API (`https://{account}.services.ai.azure.com/api/projects/{project}/`):

```go
// Key operations:
GetPagedConnections(ctx) → *PagedConnection    // List project connections
GetConnectionWithCredentials(ctx, name) → *Connection  // Get connection secrets
GetAllConnections(ctx) → []Connection          // Paginated connection fetch
```

### Azure Client (`pkg/azure/azure_client.go`)

Wraps ARM subscriptions API for listing Azure locations.

### Registry Client (`pkg/agents/registry_api/`)

Accesses the Azure ML agent manifest registry for catalog-based agent creation:

```go
client := registry_api.NewRegistryAgentManifestClient(registryName, cred)
manifest, _ := client.GetManifest(ctx, manifestName, manifestVersion)
allManifests, _ := client.GetAllLatest(ctx)
```

---

## 12. Configuration Model

### Service Target Config (`project/config.go`)

Extra config in `azure.yaml` under a service's `config:` block:

```go
type ServiceTargetAgentConfig struct {
    Environment map[string]string  `json:"env"`
    Container   *ContainerSettings `json:"container"`
    Deployments []Deployment       `json:"deployments"`
    Resources   []Resource         `json:"resources"`
}

type ContainerSettings struct {
    Resources *ResourceSettings  // Memory, CPU
    Scale     *ScaleSettings     // MinReplicas, MaxReplicas
}

type Deployment struct {
    Name  string          // Deployment name
    Model DeploymentModel // Model name, format, version
    Sku   DeploymentSku   // SKU name, capacity
}
```

### Protobuf Marshaling

Configuration is exchanged with `azd` core via `structpb.Struct` (Protocol Buffers):

```go
// Unmarshal from gRPC
UnmarshalStruct(svc.Config, &foundryAgentConfig)

// Marshal back to gRPC
agentConfigStruct, _ := MarshalStruct(foundryAgentConfig)
svc.Config = agentConfigStruct
```

---

## 13. Directory Structure Summary

```
cli/azd/extensions/azure.ai.agents/
├── main.go                          # Entry point
├── extension.yaml                   # Extension manifest
├── internal/
│   ├── cmd/
│   │   ├── root.go                  # Cobra command tree
│   │   ├── init.go                  # azd ai agent init (interactive wizard)
│   │   ├── listen.go                # gRPC event listener (extension host)
│   │   ├── mcp.go                   # MCP server
│   │   ├── metadata.go              # Command metadata generation
│   │   ├── debug.go                 # Debug logging setup
│   │   └── version.go               # Version command
│   ├── pkg/
│   │   ├── agents/
│   │   │   ├── agent_api/
│   │   │   │   ├── models.go        # REST API request/response types
│   │   │   │   └── operations.go    # HTTP client for Agent Service API
│   │   │   ├── agent_yaml/
│   │   │   │   ├── yaml.go          # agent.yaml type definitions
│   │   │   │   ├── parse.go         # YAML parsing, validation, extraction
│   │   │   │   └── map.go           # YAML→API type mapping
│   │   │   └── registry_api/
│   │   │       ├── models.go        # Registry manifest types
│   │   │       └── operations.go    # Registry API client
│   │   └── azure/
│   │       ├── azure_client.go      # Azure subscriptions client
│   │       ├── foundry_projects_client.go  # AI Foundry project API
│   │       └── client_options.go    # Standard ARM client options
│   ├── project/
│   │   ├── config.go                # ServiceTargetAgentConfig types
│   │   ├── parser.go                # FoundryParser (identity, postdeploy)
│   │   └── service_target_agent.go  # ServiceTargetProvider implementation
│   ├── tools/
│   │   └── add_agent.go             # MCP tool: add_agent
│   └── version/                     # Build version info
├── schemas/                         # JSON schemas
└── tests/                           # Test files
```

---

## 14. Key Data Flow: `azd deploy` for a Hosted Agent

```
User runs: azd deploy
    │
    ├─► azd core starts extension binary with `listen` command
    │       extension connects via gRPC to azd host
    │       registers ServiceTarget("azure.ai.agent") + event handlers
    │       signals Ready
    │
    ├─► azd core fires "predeploy" event
    │       extension: SetIdentity → populateContainerSettings → envUpdate
    │
    ├─► azd core calls ServiceTarget.Initialize()
    │       extension: locate agent.yaml, get Azure creds, parse Foundry project
    │
    ├─► azd core calls ServiceTarget.GetTargetResource()
    │       extension: resolve Foundry project → CognitiveServices/accounts/projects
    │
    ├─► azd core calls ServiceTarget.Package()
    │       extension: delegate to azdClient.Container().Build() + Package()
    │
    ├─► azd core calls ServiceTarget.Publish()
    │       extension: delegate to azdClient.Container().Publish()
    │       → pushes container image to Azure Container Registry
    │
    ├─► azd core calls ServiceTarget.Deploy()
    │       extension:
    │         1. Read agent.yaml → ContainerAgent
    │         2. Find published image URL from artifacts
    │         3. Resolve ${VAR} env vars from azd environment
    │         4. Build API request (image, CPU, memory, protocols, env vars)
    │         5. POST /agents/{name}/versions → create agent version
    │         6. POST /agents/{name}/versions/{v}/containers/default:start
    │         7. Poll GET /agents/{name}/operations/{id} until Succeeded
    │         8. Set AGENT_<SVC>_NAME, AGENT_<SVC>_VERSION env vars
    │         9. Return playground URL + API endpoint as artifacts
    │
    └─► azd core displays deployment results with endpoints
```
