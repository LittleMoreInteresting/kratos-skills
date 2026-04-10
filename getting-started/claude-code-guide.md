# Claude Code Integration Guide

Claude Code provides the best experience for using kratos-skills with native skill support, subagents, and dynamic context.

## Installation

### Project-Level Installation (Recommended)

Add kratos-skills to your project for automatic discovery:

```bash
# Add as git submodule
git submodule add  https://github.com/LittleMoreInteresting/kratos-skills.git .claude/skills/kratos-skills

# Or clone directly
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .claude/skills/kratos-skills
```

Claude Code automatically discovers skills in `.claude/skills/` directories.

### Personal-Level Installation

To use across all your projects:

```bash
# Clone to personal skills directory
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git ~/.claude/skills/kratos-skills
```

## Usage

### Automatic Invocation

Claude loads the skill automatically when you work with go-kratos files:
- Files with `.proto` extension
- `go.mod` containing kratos dependency
- Files in `internal/` directory with kratos patterns

### Manual Invocation

Type `/kratos-skills` to invoke directly:

```
/kratos-skills                    # General kratos guidance
/kratos-skills Create a user API  # Specific task
/kratos-skills How do I add middleware?  # Question
```

### With Arguments

Pass arguments for specific tasks:

```
/kratos-skills Create a user management service with CRUD operations
/kratos-skills Add JWT authentication to my API
/kratos-skills Implement database layer with GORM
```

## Advanced Features

### Subagents

Claude Code supports spawning subagents for complex tasks:

```
You: Plan a microservices architecture for an e-commerce system

Claude: I'll spawn subagents to analyze this comprehensively...

[Subagent 1] Analyzing requirements...
[Subagent 2] Designing service boundaries...
[Subagent 3] Planning data flow...

Result: Comprehensive architecture with:
- Service decomposition
- API definitions
- Data models
- Communication patterns
```

Use subagents for:
- **Architectural planning**: Design service boundaries and interactions
- **Codebase analysis**: Understand large existing projects
- **Refactoring**: Plan and execute large-scale changes
- **Security audits**: Review code for vulnerabilities

### Dynamic Context

The skill can execute shell commands to gather live project data:

```bash
# Automatically executed by skill:
find . -name "*.proto" -type f          # Find proto files
grep -r "go-kratos/kratos" go.mod       # Check kratos version
ls internal/service/ 2>/dev/null        # List existing services
cat api/*.proto 2>/dev/null | head -50  # Read API definitions
```

This enables context-aware suggestions based on your actual project state.

### Tool Restrictions

The skill specifies `allowed-tools` for safe operations:

```yaml
allowed-tools:
  - Read      # Read files
  - Grep      # Search content
  - Glob      # Find files
```

Only read-only operations are permitted, ensuring safety.

## Common Workflows

### Creating a New Service

```
You: /kratos-skills Create a new order service with HTTP and gRPC APIs

Claude:
1. Analyzing project structure...
2. Creating proto definition...
3. Generating code with kratos proto...
4. Implementing service layer...
5. Implementing biz layer...
6. Implementing data layer...
7. Wiring dependencies...
8. Testing the service...

✅ Order service created successfully!
```

### Adding a Feature

```
You: /kratos-skills Add user authentication to my API

Claude:
1. Reading existing API definitions...
2. Creating auth middleware...
3. Adding JWT validation...
4. Updating service layer...
5. Testing authentication flow...

✅ Authentication added successfully!
```

### Debugging

```
You: /kratos-skills Why is my service returning 500 errors?

Claude:
1. Analyzing error logs...
2. Checking service implementation...
3. Reviewing error handling...
4. Found issue: Missing error wrapping in biz layer

✅ Fix applied: Added proper error handling
```

## Best Practices

### 1. Be Specific

```
❌ "Fix my code"
✅ "Add input validation to the CreateUser handler"
```

### 2. Provide Context

```
❌ "Create an API"
✅ "Create a REST API for user management with CRUD operations, following the existing patterns in internal/service/"
```

### 3. Use References

```
❌ "How do I use middleware?"
✅ "Reference middleware-patterns.md and add logging middleware to my HTTP server"
```

### 4. Iterate

```
You: Create a basic user service
Claude: [Creates service]
You: Add email validation
Claude: [Adds validation]
You: Add password hashing
Claude: [Adds hashing]
```

## Troubleshooting

### Skill Not Loading

```bash
# Check skill directory
ls -la .claude/skills/kratos-skills/

# Verify SKILL.md exists
cat .claude/skills/kratos-skills/SKILL.md | head -20

# Restart Claude Code
```

### Commands Not Working

```bash
# Verify kratos CLI is installed
which kratos
kratos --version

# Check Go installation
go version

# Verify proto compiler
which protoc
protoc --version
```

### Skill Outdated

```bash
# Update skill
cd .claude/skills/kratos-skills
git pull origin main
```

## Tips and Tricks

### 1. Quick Reference

```
You: What patterns are available?

Claude: Available pattern guides:
- http-patterns.md - HTTP API patterns
- grpc-patterns.md - gRPC service patterns
- clean-architecture-patterns.md - Clean architecture
- database-patterns.md - Database operations
- middleware-patterns.md - Middleware patterns
```

### 2. Pattern Application

```
You: Apply the repository pattern to my user service

Claude: Reading database-patterns.md...
        Analyzing current implementation...
        Applying repository pattern...

✅ Repository pattern applied!
```

### 3. Code Review

```
You: Review my service implementation

Claude: Analyzing code against kratos patterns...
        Checking clean architecture compliance...
        Reviewing error handling...

Findings:
✅ Proper layer separation
⚠️ Missing input validation
⚠️ Context not propagated in one function
```

## Next Steps

1. Try the [examples](../examples/) to see kratos-skills in action
2. Explore the [pattern guides](../references/) for detailed patterns
3. Review [best practices](../best-practices/) for production code
4. Check [troubleshooting](../troubleshooting/) for common issues

## Resources

- [Claude Code Documentation](https://code.claude.com/docs)
- [Agent Skills Specification](https://agentskills.io/)
- [go-kratos Documentation](https://go-kratos.dev/)
