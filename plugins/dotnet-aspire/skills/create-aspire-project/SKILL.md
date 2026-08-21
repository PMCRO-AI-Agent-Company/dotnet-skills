---
name: create-aspire-project
description: >
  Create a new .NET Aspire 13+ cloud-native application with Ollama GPU containers,
  MCP server integration, ServiceDefaults (health checks, resilience, OpenTelemetry),
  and CopilotKit frontend. Production-ready, enterprise-grade, bleeding-edge.
  USE FOR: creating Aspire app hosts, scaffolding distributed apps, adding Ollama
  to Aspire projects, wiring MCP servers into Aspire orchestration, setting up
  .NET Aspire with GPU containers.
  DO NOT USE FOR: modifying existing Aspire projects, simple Blazor-only projects
  (use create-blazor-project), MAF agent creation without Aspire (use create-maf-agent).
---

# Create a .NET Aspire Project

Scaffolds a production .NET Aspire 13.4+ application with the full stack:
Ollama GPU container with data persistence, MCP actuator servers (filesystem,
terminal, Playwright), CopilotKit frontend, and ServiceDefaults.

## Stack Overview (v2.1.0 — PMCR-O Runtime)

```
src/
├── ProjectName.AppHost/           ← Aspire AppHost — .NET 11 + Aspire 13.4
│   ├── AppHost.cs                 ← Orchestration: Ollama GPU + MCP servers + CopilotKit
│   └── ProjectName.AppHost.csproj ← Aspire SDK + Ollama hosting + MAF DevUI
├── ProjectName.ServiceDefaults/   ← Shared: health checks, resilience, OTEL
│   ├── Extensions.cs              ← .AddServiceDefaults<TBuilder>()
│   └── OllamaExtensions.cs        ← Keyed IChatClient for Ollama API
├── ProjectName.OrchestratorService/ ← MAF WorkflowBuilder — sequential P→M→C→R
├── ProjectName.OrchestratorApi/   ← HTTP facade (Scalar, CopilotKit AG-UI)
├── frontend/                      ← Next.js CopilotKit runtime
└── mcp/
    ├── ProjectName.Mcp.Filesystem/ ← MCP filesystem actuator
    ├── ProjectName.Mcp.Terminal/   ← MCP terminal actuator
    └── ProjectName.Mcp.Playwright/ ← MCP browser actuator
```

## Prerequisites

```shell
dotnet workload install aspire
# .NET 11 SDK required
# Docker Desktop for Ollama GPU container
```

## Scaffold the Full Stack

```shell
# Create solution
dotnet new sln -o {ProjectName}

# AppHost (Aspire SDK 13.4+)
dotnet new aspire-apphost -o {ProjectName}/src/{ProjectName}.AppHost

# ServiceDefaults
dotnet new aspire-servicedefaults -o {ProjectName}/src/{ProjectName}.ServiceDefaults

# OrchestratorService (MAF WorkflowBuilder)
dotnet new web -o {ProjectName}/src/{ProjectName}.OrchestratorService

# OrchestratorApi (HTTP facade)
dotnet new webapi -o {ProjectName}/src/{ProjectName}.OrchestratorApi

# MCP servers
dotnet new console -o {ProjectName}/mcp/{ProjectName}.Mcp.Filesystem
dotnet new console -o {ProjectName}/mcp/{ProjectName}.Mcp.Terminal
dotnet new console -o {ProjectName}/mcp/{ProjectName}.Mcp.Playwright
```

## AppHost.cs — Ollama GPU + MCP + CopilotKit

```csharp
// AppHost.cs
var builder = DistributedApplication.CreateBuilder(args);
var repoRoot = builder.AddParameter("repoRoot");

// Ollama GPU container — persistent, 16384 context
var ollama = builder.AddOllama("ollama-server")
    .WithGPUSupport(OllamaGpuVendor.Nvidia)
    .WithLifetime(ContainerLifetime.Persistent)
    .WithDataVolume("ollama-data")
    .WithEnvironment("OLLAMA_CONTEXT_LENGTH", "16384")
    .WithEnvironment("OLLAMA_FLASH_ATTENTION", "0");
var model = ollama.AddModel("model-orchestrator", "qwen3:8b");

// MCP servers
var mcpFilesystem = builder.AddProject<Projects.ProjectName_Mcp_Filesystem>("mcp-fs")
    .WithEnvironment("Filesystem__SandboxRoot", repoRoot);
var mcpTerminal = builder.AddProject<Projects.ProjectName_Mcp_Terminal>("mcp-term")
    .WithEnvironment("Parameters__working-root", repoRoot);
var mcpPlaywright = builder.AddProject<Projects.ProjectName_Mcp_Playwright>("mcp-pw")
    .WithEnvironment("Playwright__Headless", "false");

// OrchestratorService — full PMCR-O cycle in-process
var orchestrator = builder.AddProject<Projects.ProjectName_OrchestratorService>("orchestrator")
    .WithReference(ollama).WithReference(model)
    .WithReference(mcpFilesystem).WithReference(mcpTerminal)
    .WithEnvironment("Orchestrator__FileSystemRoot", repoRoot)
    .WaitFor(model).WaitFor(mcpFilesystem).WaitFor(mcpTerminal);

// OrchestratorApi — HTTP facade
var api = builder.AddProject<Projects.ProjectName_OrchestratorApi>("api")
    .WithReference(ollama).WithReference(model)
    .WithReference(mcpFilesystem).WithReference(mcpPlaywright).WithReference(mcpTerminal)
    .WithReference(orchestrator)
    .WaitFor(model).WaitFor(orchestrator);

// CopilotKit frontend (Next.js)
var frontend = builder.AddNextJsApp("frontend", "../frontend")
    .WithReference(orchestrator)
    .WithExternalHttpEndpoints()
    .WaitFor(orchestrator);

// DevUI Dashboard
var devUI = builder.AddDevUI("pmcro-devui");
devUI.WithAgentService(orchestrator);

builder.Build().Run();
```

## MAF WorkflowBuilder Integration

```csharp
// OrchestratorService — WorkflowBuilder with sequential P→M→C→R
var workflow = builder
    .AddAgent(Planner())
    .AddAgent(Maker())
    .AddAgent(Checker())
    .AddAgent(Reflector());

AIAgent Planner() => new("Planner")
    .SetDefaultModel(new OllamaChatClient("http://localhost:11434", "qwen3:8b"))
    .WithTools(agentFilesystem, agentTerminal);

AIAgent Maker() => /* TYPE1/TYPE2 boundary — HIL gated */
AIAgent Checker() => /* 3-dim scoring */
AIAgent Reflector() => /* ACCEPT/LOOP/ESCALATE */
```

## OllamaExtensions.cs — Keyed IChatClient

```csharp
// ServiceDefaults/OllamaExtensions.cs
public static class OllamaExtensions
{
    public static class Keys { public const string Orchestrator = "model-orchestrator"; }

    public static TBuilder AddOllamaClients<TBuilder>(this TBuilder builder)
        where TBuilder : IHostApplicationBuilder
    {
        builder.Services.AddKeyedSingleton<IChatClient>(Keys.Orchestrator, (sp, _) =>
        {
            var cs = config.GetConnectionString("ollama-server") ?? "http://localhost:11434";
            var model = config["Ollama:Models:Orchestrator"] ?? "qwen3:8b";
            var http = new HttpClient { BaseAddress = new Uri(cs), Timeout = Timeout.InfiniteTimeSpan };
            return new OllamaApiClient(http) { SelectedModel = model };
        });
        return builder;
    }
}
```

## After Scaffolding

1. `dotnet workload install aspire` (if not already)
2. `dotnet build` — verify all projects compile
3. `docker pull ollama/ollama:latest` — ensure Ollama container image exists
4. `dotnet run --project src/{ProjectName}.AppHost` — launches Aspire dashboard
5. Ollama auto-pulls `qwen3:8b` on first run

## Constraints

- Ollama with GPU support requires Nvidia GPU + Docker / WSL2
- `AddNextJsApp` is `[Experimental]` in Aspire 13 — suppress ASPIREJAVASCRIPT001
- OLLAMA_CONTEXT_LENGTH: 16384 minimum for full skill manifests
- OLLAMA_FLASH_ATTENTION: disable on RTX 4070 Mobile due to SIGSEGV (known issue)
- MCP Playwright intentionally excluded from `.WaitFor()` — lazy actuator, tolerated down

## Don'ts

- Don't use .NET < 11 — Aspire 13+ requires it
- Don't hardcode MaxLoops in AppHost — use appsettings.json (config precedence)
- Don't WaitFor Playwright MCP — cascading crash risk
- Don't expose health check endpoints in production without `HealthChecks:ExposeEndpoints`