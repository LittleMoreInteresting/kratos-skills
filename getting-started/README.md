# Getting Started with kratos-skills

This guide helps you choose and set up the right AI tool for working with go-kratos.

## Tool Comparison

| Feature | Claude Code | GitHub Copilot | Cursor | Windsurf |
|---------|-------------|----------------|--------|----------|
| **Best For** | Complex tasks, subagents | Inline suggestions | IDE-native experience | Cascade workflows |
| **Skill Support** | Native | Via instructions | Via rules | Via rules |
| **Terminal Integration** | Built-in | VS Code terminal | Built-in | Built-in |
| **Subagents** | ✅ Yes | ❌ No | ❌ No | ✅ Yes |
| **Dynamic Context** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Learning Curve** | Medium | Low | Low | Medium |

## Quick Setup

### Claude Code (Recommended)

```bash
# Project-level installation
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .claude/skills/kratos-skills

# Usage
/kratos-skills Create a user service
```

**Why Claude Code?**
- Native skill support with automatic discovery
- Subagents for complex architectural planning
- Dynamic context gathering from your project
- Direct terminal integration for running kratos CLI

### GitHub Copilot

```bash
# Clone to project
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .ai-context/kratos-skills

# Create .github/copilot-instructions.md
echo "# kratos-skills\n\nSee .ai-context/kratos-skills/" > .github/copilot-instructions.md
```

**Why Copilot?**
- Works in your existing IDE (VS Code, JetBrains)
- Inline suggestions as you type
- Great for quick code completions

### Cursor

```bash
# Clone to project
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .ai-context/kratos-skills

# Create .cursorrules
cat > .cursorrules << 'EOF'
# Kratos Skills

When working with go-kratos:
- Follow clean architecture (Transport → Service → Biz → Data)
- Use Wire for dependency injection
- Reference .ai-context/kratos-skills/ for patterns
EOF
```

**Why Cursor?**
- Native IDE experience
- Chat interface for complex queries
- Good balance of features

### Windsurf

```bash
# Clone to project
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .ai-context/kratos-skills

# Create .windsurfrules
cat > .windsurfrules << 'EOF'
# Kratos Skills

When working with go-kratos:
- Follow clean architecture (Transport → Service → Biz → Data)
- Use Wire for dependency injection
- Reference .ai-context/kratos-skills/ for patterns
EOF
```

**Why Windsurf?**
- Cascade AI for complex workflows
- Good for iterative development
- Modern IDE features

## Feature Deep Dive

### Subagents (Claude Code Only)

Subagents allow the AI to spawn isolated tasks for complex operations:

```
You: Plan a microservices architecture for an e-commerce system
Claude: Spawns subagent to analyze requirements...
        Spawns subagent to design service boundaries...
        Spawns subagent to plan data flow...
        Returns comprehensive architecture plan
```

Use cases:
- Architectural planning
- Codebase analysis
- Refactoring large codebases
- Security audits

### Dynamic Context (Claude Code Only)

The skill can execute shell commands to gather live project data:

```bash
# Skill automatically runs:
find . -name "*.proto" -type f  # Find proto files
grep -r "kratos" go.mod         # Check kratos version
ls internal/service/            # List existing services
```

This provides context-aware suggestions based on your actual project state.

### Terminal Integration (All Tools)

All tools support running kratos CLI directly:

```bash
# Generate proto code
kratos proto client api/user.proto

# Generate service
kratos proto server api/user.proto

# Run wire
cd internal && wire
```

## Choosing the Right Tool

### Use Claude Code If:
- You want the most advanced features
- You're working on complex microservices
- You need architectural planning assistance
- You want automatic skill discovery

### Use GitHub Copilot If:
- You want inline suggestions in your IDE
- You prefer minimal setup
- You mainly need code completions
- You're already using VS Code or JetBrains

### Use Cursor If:
- You want an IDE-native experience
- You prefer chat-based interactions
- You want a balance of features
- You're comfortable switching IDEs

### Use Windsurf If:
- You like Cascade AI workflows
- You want iterative development
- You prefer modern IDE features
- You're open to new tools

## Next Steps

1. Choose your tool based on the comparison above
2. Follow the specific setup guide:
   - [Claude Code Setup](claude-code-guide.md)
   - [GitHub Copilot Setup](copilot-guide.md)
   - [Cursor Setup](cursor-guide.md)
   - [Windsurf Setup](windsurf-guide.md)
3. Try the [examples](../examples/) to verify your setup
4. Start building with go-kratos!

## Getting Help

- **Tool-specific issues**: See the individual setup guides
- **Kratos questions**: Reference the [pattern guides](../references/)
- **Common problems**: Check [troubleshooting](../troubleshooting/)
- **Examples**: Explore the [examples directory](../examples/)
