---
name: create-maf-agent
description: >
  Create a Microsoft.Agents.AI agent using the MAF WorkflowBuilder pattern
  with OllamaChatClient, sequential phase dispatch, and integrated MCP tool
  access. Follows Anthropic Orchestrator-Workers pattern.
  USE FOR: creating MAF AIAgents, building agent workflows, wiring Ollama to
  MAF, designing multi-agent systems with MCP tool integration.
  DO NOT USE FOR: Aspire orchestration (use create-aspire-project), simple
  Blazor components (use create-blazor-project).
---

# Create a MAF Agent

Scaffolds a Microsoft.Agents.AI agent using the WorkflowBuilder pattern.
The agent runs the full PMCR-O cognitive loop (Plan→Make→Check→Reflect)
as a sequential graph, with each phase as its own AIAgent node.

## Architecture

```
OrchestratorService
├── PmcroLoop.cs              ← WorkflowBuilder sequential graph
│   ├── AIAgent("Planner")    ← Decompose seed intent
│   ├── AIAgent("Maker")      ← Execute plan (TYPE2 tools) + HIL gate (TYPE1)
│   ├── AIAgent("Checker")    ← 3-dim scoring (completeness, correctness, compliance)
│   └── AIAgent("Reflector")  ← ACCEPT/LOOP/ESCALATE verdict
├── Program.cs                ← DI wiring: OllamaChatClient + MCP tools
└── appsettings.json          ← MaxLoops, model config, trail paths
```

## MAF Packages

```xml
<PackageReference Include="Microsoft.Agents.AI" />
<PackageReference Include="Microsoft.Agents.AI.DevUI" />
<PackageReference Include="Microsoft.Extensions.AI.Ollama" />
```

## WorkflowBuilder — Sequential Phase Dispatch

```csharp
// PmcroLoop.cs
using Microsoft.Agents.AI;

public class PmcroLoop
{
    public static void Build(WorkflowBuilder builder)
    {
        builder
            .AddAgent(Planner())
            .AddAgent(Maker())
            .AddAgent(Checker())
            .AddAgent(Reflector());
    }

    private static AIAgent Planner() => new("Planner")
        .SetInstructions("""
            You are the Planner. Decompose the seed intent into one bounded,
            checkable unit of work. Output an ExecutionPlan in JSONL format.
            """)
        .SetDefaultModel(new OllamaChatClient("http://localhost:11434", "qwen3:8b"));

    private static AIAgent Maker() => new("Maker")
        .SetInstructions("""
            You are the Maker. Execute exactly what the plan specifies.
            TYPE2 tools: call directly. TYPE1 operations: stub-and-halt for HIL.
            Output a MakerFrame in JSONL format.
            """)
        .SetDefaultModel(new OllamaChatClient("http://localhost:11434", "qwen3:8b"))
        .WithTools(/* McpToolCache: filesystem, terminal */);

    private static AIAgent Checker() => new("Checker")
        .SetInstructions("""
            You are the Checker. Independently verify every success criterion
            across 3 dimensions: completeness, correctness, compliance.
            Threshold: aggregate > 0.80, compliance = 1.0.
            """)
        .SetDefaultModel(new OllamaChatClient("http://localhost:11434", "qwen3:8b"));

    private static AIAgent Reflector() => new("Reflector")
        .SetInstructions("""
            You are the Reflector. Decide ACCEPT/LOOP/ESCALATE.
            Crystallize recurring failures as EarnedConstraints (EC-007).
            Seal disposition.json on ACCEPT.
            """)
        .SetDefaultModel(new OllamaChatClient("http://localhost:11434", "qwen3:8b"));
}
```

## Program.cs — Service Registration

```csharp
// OrchestratorService/Program.cs
using ProjectName.ServiceDefaults;
using Microsoft.Agents.AI;

var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();
builder.AddOllamaClients();

// WorkflowBuilder registration
builder.Services.AddSingleton(sp =>
{
    var workflow = new WorkflowBuilder();
    PmcroLoop.Build(workflow);
    return workflow.Build();
});

// MCP tool cache
builder.Services.AddSingleton<McpToolCache>();

var app = builder.Build();
app.MapDefaultEndpoints();
app.Run();
```

## appsettings.json — PMCR-O Configuration

```json
{
  "MaxLoops": 5,
  "Checker": {
    "PassThreshold": 0.8,
    "LawComplianceThreshold": 1.0
  },
  "Ollama": {
    "Models": {
      "Orchestrator": "qwen3:8b"
    }
  },
  "Orchestrator": {
    "FileSystemRoot": "B:\\pmcro-cline"
  }
}
```

## TYPE1/TYPE2 Boundary (EC-002)

```csharp
// Maker AIAgent — tool boundary
.WithTools(tools =>
{
    // TYPE2: read-only — direct call, no HIL
    tools.AddMcpTool("read_file", cache.Get("filesystem"));
    tools.AddMcpTool("list_directory", cache.Get("filesystem"));
    tools.AddMcpTool("search_files", cache.Get("filesystem"));

    // TYPE1: write/mutate — HIL required
    // tools.AddMcpTool("write_file", cache.Get("filesystem")); // gated
    // tools.AddMcpTool("create_directory", cache.Get("filesystem")); // gated
})
```

## After Scaffolding

1. `dotnet add package Microsoft.Agents.AI`
2. Copy `PmcroLoop.cs` into `OrchestratorService/`
3. Wire MCP tool cache in `Program.cs`
4. Pull Ollama model: `ollama pull qwen3:8b`
5. `dotnet build && dotnet run`

## Constraints

- MAF WorkflowBuilder enforces SEQUENTIAL-001: phases run in order, never fan-out
- TYPE1/TYPE2 boundary is enforced at the tool cache level, not the agent prompt
- MaxLoops sourced from appsettings.json, NEVER hardcoded in code
- Ollama must be running locally or via Aspire container

## Don'ts

- Don't fan out agents — SEQUENTIAL-001 is non-negotiable
- Don't hardcode model paths or MaxLoops
- Don't skip the TYPE1 gate on write tools
- Don't use `SetDefaultModel` on the WorkflowBuilder — set per-AIAgent