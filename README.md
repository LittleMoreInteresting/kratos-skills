# go-kratos Skills for AI Agents

English | [简体中文](README_CN.md)

This is an [Agent Skill](https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) containing structured knowledge and patterns for AI coding assistants to help developers work effectively with the [go-kratos](https://github.com/go-kratos/kratos) framework.

## What is a Skill?

Skills are folders of instructions, scripts, and resources that AI agents discover and load dynamically to perform better at specific tasks. This skill teaches AI agents how to generate production-ready go-kratos microservices code.

## Purpose

This skill enables AI agents (Claude, GitHub Copilot, Cursor, etc.) to:
- Generate accurate go-kratos code following framework conventions
- Understand the four-layer architecture (Transport → Service → Biz → Data)
- Apply clean architecture principles
- Use Wire for dependency injection
- Apply best practices for microservices development
- Troubleshoot common issues efficiently
- Build production-ready applications

## Quick Install

Just ask your AI agent:

```
Install kratos-skills from  https://github.com/LittleMoreInteresting/kratos-skills
```

Or manually:

```bash
# Project-level (recommended)
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .claude/skills/kratos-skills

# Personal-level (all projects)
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git ~/.claude/skills/kratos-skills
```

## Agent Skill Structure

Following the [Agent Skills Spec](https://github.com/anthropics/skills/blob/main/spec/agent-skills-spec.md) and [Claude Code skills documentation](https://code.claude.com/docs/en/skills):

```
kratos-skills/
├── SKILL.md                    # Entry point with YAML frontmatter
├── getting-started/            # Getting started guides
│   ├── README.md               # Tool comparison overview
│   ├── claude-code-guide.md    # Claude Code (recommended)
│   ├── cursor-guide.md         # Cursor IDE
│   ├── copilot-guide.md        # GitHub Copilot
│   └── windsurf-guide.md       # Windsurf IDE
├── references/                 # Detailed pattern documentation
│   ├── http-api-patterns.md    # HTTP API development patterns
│   ├── grpc-patterns.md        # gRPC service patterns
│   ├── clean-architecture-patterns.md  # Clean architecture patterns
│   ├── data-patterns.md        # Data layer, ORM, transactions, caching
│   ├── error-patterns.md       # Error handling and error codes
│   ├── validation-patterns.md  # Input validation with proto
│   ├── tracing-patterns.md     # OpenTelemetry tracing and metrics
│   ├── registry-patterns.md    # Service discovery and registration
│   ├── event-patterns.md       # Event-driven architecture with Kafka
│   ├── cqrs-patterns.md        # CQRS pattern implementation
│   ├── middleware-patterns.md  # Middleware and interceptors
│   └── kratos-cli-commands.md  # Kratos CLI reference
├── best-practices/             # Production recommendations
├── troubleshooting/            # Common issues and solutions
├── skill-patterns/             # Advanced skill examples (templates)
│   ├── analyze-project.md      # Explore agent example
│   ├── generate-service.md     # Argument passing example
│   └── plan-architecture.md    # Plan agent example
└── examples/                   # Demo projects and verification
```

## Using This Skill

### With Claude Code (Recommended)

Claude Code natively supports the [Agent Skills specification](https://agentskills.io/). This skill is optimized for Claude Code with advanced features:

#### Project-Level Installation (Git Submodule)
Add kratos-skills to your project for automatic discovery:

```bash
# Add as git submodule
git submodule add  https://github.com/LittleMoreInteresting/kratos-skills.git .claude/skills/kratos-skills

# Or clone directly
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .claude/skills/kratos-skills
```

Claude Code automatically discovers skills in `.claude/skills/` directories.

#### Personal-Level Installation
To use across all your projects, install to your personal skills directory:

```bash
# Clone to personal skills directory
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git ~/.claude/skills/kratos-skills
```

#### Usage in Claude Code
- **Automatic**: Claude loads the skill when you work with go-kratos files (`.proto`, `go.mod` with kratos)
- **Manual**: Type `/kratos-skills` to invoke directly for kratos guidance
- **With arguments**: `/kratos-skills Create a user management API` for specific tasks
- **Check availability**: Ask "What skills are available?" to see if it's loaded

#### Advanced Features
- **Dynamic context**: Skills can execute shell commands to gather live project data
- **Subagents**: Use `context: fork` for isolated analysis or planning tasks
- **Tool restrictions**: `allowed-tools` ensures safe, read-only operations
- See [skill-patterns/](skill-patterns/) for advanced patterns and templates

### With Claude Desktop

Add to `claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "kratos-skills": {
      "command": "node",
      "args": ["/path/to/skill-server.js", "/path/to/kratos-skills"]
    }
  }
}
```

### With GitHub Copilot

See [copilot-guide.md](getting-started/copilot-guide.md) for detailed setup. Quick start:

```bash
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .ai-context/kratos-skills
```

Then create `.github/copilot-instructions.md` referencing the patterns.

### With Cursor

See [cursor-guide.md](getting-started/cursor-guide.md) for detailed setup. Quick start:

```bash
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .ai-context/kratos-skills
```

Then create `.cursorrules` referencing the patterns.

### With Windsurf

See [windsurf-guide.md](getting-started/windsurf-guide.md) for detailed setup. Quick start:

```bash
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .ai-context/kratos-skills
```

Then create `.windsurfrules` referencing the patterns.

## Integration with go-kratos Ecosystem

kratos-skills is part of the go-kratos ecosystem for AI-assisted development:

| Tool | Purpose | Size | Best For |
|------|---------|------|----------|
| **kratos CLI** | Project scaffolding and code generation | ~10MB | Creating new projects |
| **kratos-skills** (this repo) | Comprehensive knowledge base | ~50KB | All AI tools, deep learning |

The AI runs `kratos` CLI directly in the terminal for code generation — no separate tools or servers needed. See [references/kratos-cli-commands.md](references/kratos-cli-commands.md) for the complete command reference.

### How They Work Together

```
┌─────────────────────────────────────────────────────────────┐
│                     AI Assistant                            │
│  (Claude Code, GitHub Copilot, Cursor, Windsurf)           │
└────────────┬─────────────────────┬──────────────────────────┘
             │                     │
             ├─ Tool Layer ────────┤
             │  kratos CLI         │  "Generate code" - Project scaffolding
             │  (~10MB)            │  Proto generation, Wire
             │                     │
             └─ Knowledge Layer ───┘
                kratos-skills        "How & Why" - Detailed patterns
                (~50KB)              + CLI command reference
                                     Loaded when needed
```

### Usage Scenarios

**Scenario 1: Claude Code User (Best Experience)**
- Uses: `kratos-skills` (this repo) as native skill
- Benefits:
  - Deep knowledge from pattern guides
  - AI runs kratos CLI commands directly in terminal
  - Dynamic context with live project data
  - Subagent workflows for complex tasks
- Invocation: `/kratos-skills` or automatic when working with kratos

**Scenario 2: GitHub Copilot User**
- Uses: kratos-skills (loaded via `.github/copilot-instructions.md`)
- Benefits: Quick inline suggestions, workflow guidance, kratos CLI via terminal

**Scenario 3: Cursor/Windsurf User**
- Uses: kratos-skills (in project rules)
- Benefits: IDE-native experience with kratos guidance, kratos CLI via terminal

See [Getting Started Guides](getting-started/) for detailed integration instructions for each tool.

## Quick Links

**Skill Documentation:**

- 📖 **[SKILL.md](SKILL.md)** - Main skill entry point and navigation
- 📚 **[go-kratos Quick Start](https://go-kratos.dev/docs/getting-started/start/)** - Official go-kratos framework tutorial
- 🎯 **[Advanced Examples](skill-patterns/)** - Subagents, dynamic context, etc.

**Getting Started Guides:**

- 💡 **[Claude Code](getting-started/claude-code-guide.md)** - Full features, subagents (recommended)
- 🖱️ **[Cursor](getting-started/cursor-guide.md)** - IDE integration with .cursorrules
- 🤖 **[GitHub Copilot](getting-started/copilot-guide.md)** - VS Code inline suggestions
- 🏄 **[Windsurf](getting-started/windsurf-guide.md)** - Cascade AI integration
- 📋 **[Tool Comparison](getting-started/README.md)** - Compare all tools

## Contributing

Contributions are welcome! Please ensure:
- Examples are complete and tested
- Patterns follow official go-kratos conventions
- Content is structured for AI consumption
- Include both correct (✅) and incorrect (❌) examples
- Follow the [Agent Skills specification](https://agentskills.io/)

## License

MIT License - Same as go-kratos framework
