# MMLib.AgentSkills

This repository contains a collection of Agent Skills for Cursor/Claude AI agents. Skills extend AI agent capabilities with specialized knowledge and workflows for .NET/C# development. All skills are stored in the `.cursor/skills/` directory.

## Instructions for AI Agent to Download Skills

To download all skills from this repository, use the following instructions:

```text
Download all Agent Skills from the repository https://github.com/Burgyn/MMLib.AgentSkills.
Skills are located in the .cursor/skills/ directory. For each skill, copy the entire directory
with all files (SKILL.md and any subdirectories scripts/, references/, assets/)
to the location where the IDE expects skills to be located.

**IMPORTANT**:
- For Cursor IDE: Download skills to `<workspace_root>/.cursor/skills/` directory (located in the workspace root)
- The IDE automatically looks for skills in this location - this is where Cursor expects to find them
- Do NOT download to the global user directory (`~/.cursor/skills/`)
- Always use the workspace root path, not the user's home directory
- The target path should be: `<workspace_root>/.cursor/skills/` where `<workspace_root>` is the root of the current project workspace
```

## Available Skills

### csharp-coding-conventions

C# coding style conventions and formatting rules. Use when writing, reviewing, or formatting C# code to ensure consistent indentation, naming, braces, line length, variable declarations, expression-bodied members, line wrapping, documentation, and code organization.

📄 [SKILL.md](.cursor/skills/csharp-coding-conventions/SKILL.md)

### dotnet-openapi-discovery

Discover current OpenAPI documentation for .NET WebAPI endpoints. Use when agent needs to find OpenAPI documentation, create requests in .http files, or test endpoints. The skill can start the service if not running, download OpenAPI JSON, and generate VS Code REST Client format requests or curl commands for testing.

📄 [SKILL.md](.cursor/skills/dotnet-openapi-discovery/SKILL.md)

### dotnet-technology-stack

Technology stack, conventions, and best practices for .NET/C# projects. Use when working with .NET 10+ projects, C# code, adding NuGet packages, selecting databases, setting up testing, or following project-specific conventions.

📄 [SKILL.md](.cursor/skills/dotnet-technology-stack/SKILL.md)

### dotnet-vertical-slice-architecture

Vertical Slice Architecture patterns and conventions for .NET Minimal API projects. Use when organizing endpoints, creating handlers, structuring modules, grouping endpoints, or implementing feature-based architecture. Covers endpoint organization, Request/Response DTOs, Setup.cs pattern, Results handlers with strongly-typed return values, MapGroup usage, and expression body syntax.

📄 [SKILL.md](.cursor/skills/dotnet-vertical-slice-architecture/SKILL.md)

### dotnet-webapi-auto-start

Automatically checks and starts .NET WebAPI service at the end of agent work. Use when agent finishes modifying a .NET WebAPI project - at the end of work, checks if the service is running based on launchSettings.json, if not, asks the user if they want to start it, and if yes, starts it using dotnet run with launch profile and opens browser to /scalar.

📄 [SKILL.md](.cursor/skills/dotnet-webapi-auto-start/SKILL.md)

### skill-creator

Guide for creating effective skills. This skill should be used when users want to create a new skill (or update an existing skill) that extends Claude's capabilities with specialized knowledge, workflows, or tool integrations.

📄 [SKILL.md](.cursor/skills/skill-creator/SKILL.md)
