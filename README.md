# .NET Agent Skills — PMCRO-AI-Agent-Company Domain Pack

**Repository for skills to assist AI coding agents with .NET and C#.**

This is the official .NET domain pack for the [PMCR-O Colony](https://github.com/PMCRO-AI-Agent-Company).  
It hosts production-grade skills and plugins that coding agents use for C#, ASP.NET Core, Blazor, Aspire, Entity Framework, MSBuild, diagnostics, testing, and Microsoft Agent Framework (MAF) workflows.

Upstream source: [dotnet/skills](https://github.com/dotnet/skills) (Microsoft .NET team).  
Colony home: [PMCRO-AI-Agent-Company](https://github.com/PMCRO-AI-Agent-Company).

## Colony Context

| Related Repo | Role |
|--------------|------|
| [pmcro-skills](https://github.com/PMCRO-AI-Agent-Company/pmcro-skills) | Cognitive loop (Orchestrator → Planner → Maker → Checker → Reflector) + skill-creator |
| [pmcro-runtime](https://github.com/PMCRO-AI-Agent-Company/pmcro-runtime) | .NET Aspire host, OrchestratorService, frontend (CopilotKit), MCP servers |
| [agent-skills](https://github.com/PMCRO-AI-Agent-Company/agent-skills) | Base template / agentskills.io layout |
| **dotnet-skills** (this repo) | .NET / C# domain skills |

Governed by Colony Laws (EC-SYS-001 Atomic Content, EC-SYS-002 one-file-per-cycle, MAF-NATIVE).  
Trails are written under the owning project’s `.pmcro/projects/.../trails/`.

## What's Included

| Plugin | Description |
|--------|-------------|
| [dotnet](plugins/dotnet/) | C# language server (LSP) integration for coding agents and high-level .NET development skills. |
| [dotnet-advanced](plugins/dotnet-advanced/) | Collection of .NET skills for handling specific .NET tasks for special scenarios. |
| [dotnet-data](plugins/dotnet-data/) | Skills for .NET data access and Entity Framework related tasks. |
| [dotnet-diag](plugins/dotnet-diag/) | Skills for .NET performance investigations, debugging, and incident analysis. |
| [dotnet-msbuild](plugins/dotnet-msbuild/) | Comprehensive MSBuild and .NET build skills: failure diagnosis, performance optimization, code quality, and modernization. |
| [dotnet-nuget](plugins/dotnet-nuget/) | NuGet and .NET package management: dependency management and modernization. |
| [dotnet-upgrade](plugins/dotnet-upgrade/) | Skills for migrating and upgrading .NET projects across framework versions, language features, and compatibility targets. |
| [dotnet-maui](plugins/dotnet-maui/) | Skills for .NET MAUI development: environment setup, diagnostics, and troubleshooting. |
| [dotnet-ai](plugins/dotnet-ai/) | AI and ML skills for .NET: technology selection, LLM integration, agentic workflows, RAG pipelines, MCP, and classic ML with ML.NET. |
| [dotnet-template-engine](plugins/dotnet-template-engine/) | .NET Template Engine skills: template discovery, project scaffolding, and template authoring. |
| [dotnet-test](plugins/dotnet-test/) | Skills for running, generating, analyzing, and improving .NET tests. |
| [dotnet-test-migration](plugins/dotnet-test-migration/) | Skills and orchestrator agent for migrating .NET test frameworks and platforms. |
| [dotnet-aspnetcore](plugins/dotnet-aspnetcore/) | ASP.NET Core web development skills including middleware, endpoints, real-time communication, and API patterns. |
| [dotnet-blazor](plugins/dotnet-blazor/) | Skills for Blazor development: component authoring, interactivity, and web application patterns. |
| [dotnet11](plugins/dotnet11/) | Skills for new .NET 11 APIs and language features. |

## Installation

### Claude Code / Copilot CLI

```
/plugin marketplace add PMCRO-AI-Agent-Company/dotnet-skills
/plugin install <plugin>@dotnet-skills
```

Or continue to use the upstream marketplace:

```
/plugin marketplace add dotnet/skills
```

### Cursor / Codex

Register this repository as a marketplace (see `.agents/plugins/marketplace.json` and `.cursor-plugin/marketplace.json`).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).  
New skills that are Colony-specific (e.g. MAF agent scaffolding aligned to `pmcro-runtime`) should be added here and referenced from `pmcro-skills` catalog when appropriate.

## License

See [LICENSE](LICENSE).
