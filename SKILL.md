---
name: kratos-skills
description: |
  Comprehensive knowledge base for go-kratos microservices framework.

  **Use this skill when:**
  - Building microservices with go-kratos (Transport → Service → Biz → Data architecture)
  - Creating HTTP/gRPC APIs with clean architecture
  - Implementing dependency injection and wire generation
  - Working with protobuf and protocol buffers
  - Adding middleware, logging, metrics, and tracing
  - Troubleshooting kratos issues or understanding framework conventions
  - Generating production-ready microservices code

  **Features:**
  - Complete pattern guides with ✅ correct and ❌ incorrect examples
  - Four-layer architecture enforcement (Transport → Service → Biz → Data)
  - Production best practices
  - Common pitfall solutions
license: MIT
allowed-tools:
  - Read
  - Grep
  - Glob
---

# go-kratos Skills for AI Agents

This skill provides comprehensive go-kratos microservices framework knowledge, optimized for AI agents helping developers build production-ready services. It covers HTTP/gRPC APIs, clean architecture, dependency injection, middleware, and troubleshooting.

## 🎯 When to Use This Skill

Invoke this skill when working with go-kratos:
- **Creating services**: HTTP APIs, gRPC services, or microservices architectures
- **Clean architecture**: Implementing Transport → Service → Biz → Data layers
- **Dependency injection**: Using Wire for dependency management
- **Protocol Buffers**: Defining .proto files and generating code
- **Production hardening**: Middleware, logging, metrics, tracing, error handling
- **Debugging**: Understanding errors, fixing configuration, or resolving issues
- **Learning**: Understanding kratos patterns and best practices

## 📚 Knowledge Structure

This skill organizes go-kratos knowledge into focused modules. **Load specific guides as needed** rather than reading everything at once:

### Quick Start Guide
**Link**: [Official go-kratos Documentation](https://go-kratos.dev/docs/getting-started/start/)
**Contains**: Installation, first service, basic commands, hello-world examples (refer to official docs)

### Pattern Guides (Detailed Reference)

#### 1. HTTP Transport Patterns
**File**: [references/http-patterns.md](references/http-patterns.md)
**When to load**: Creating HTTP endpoints, implementing REST APIs, adding HTTP middleware
**Contains**:
- HTTP server setup and configuration
- Router registration and handlers
- Request/response encoding/decoding
- HTTP middleware patterns
- Complete CRUD examples with ✅ correct vs ❌ incorrect patterns

#### 2. gRPC Transport Patterns
**File**: [references/grpc-patterns.md](references/grpc-patterns.md)
**When to load**: Building gRPC services, service-to-service communication
**Contains**:
- Protocol Buffers definition and code generation
- gRPC server and client setup
- Service discovery with etcd/consul/kubernetes
- Load balancing strategies
- gRPC interceptors and metadata

#### 3. Clean Architecture Patterns
**File**: [references/clean-architecture-patterns.md](references/clean-architecture-patterns.md)
**When to load**: Implementing business logic, understanding layer separation
**Contains**:
- Transport → Service → Biz → Data four-layer architecture
- Service layer implementation
- Business logic (Biz) layer patterns
- Data access layer patterns
- Dependency injection with Wire

#### 4. Database Patterns
**File**: [references/database-patterns.md](references/database-patterns.md)
**When to load**: Implementing data persistence, ORM integration
**Contains**:
- GORM integration patterns
- SQL operations with ent
- Redis caching strategies
- Transaction management
- Connection pooling and performance tuning

#### 5. Middleware Patterns
**File**: [references/middleware-patterns.md](references/middleware-patterns.md)
**When to load**: Adding cross-cutting concerns, request/response processing
**Contains**:
- HTTP middleware implementation
- gRPC interceptors
- Logging middleware
- Metrics and tracing middleware
- Authentication/Authorization middleware

#### 6. Kratos CLI Reference
**File**: [references/kratos-cli-commands.md](references/kratos-cli-commands.md)
**When to load**: Generating code with kratos CLI, setting up new services
**Contains**:
- kratos CLI installation and usage
- Project scaffolding commands
- Protocol buffer generation
- Wire generation commands
- Post-generation pipeline

### Supporting Resources

#### Best Practices
**File**: [best-practices/overview.md](best-practices/overview.md)
**When to load**: Production deployment, code review, optimization
**Contains**: Configuration management, logging, monitoring, security, performance

#### Troubleshooting
**File**: [troubleshooting/common-issues.md](troubleshooting/common-issues.md)
**When to load**: Debugging errors, configuration issues, runtime problems
**Contains**: Common error messages, solutions, configuration pitfalls, debugging tips

#### Claude Code Integration
**File**: [getting-started/claude-code-guide.md](getting-started/claude-code-guide.md)
**When to load**: Setting up Claude Code for kratos-skills usage
**Contains**: Installation, invocation methods, advanced features

## 🚀 Common Workflows

These workflows guide you through typical go-kratos development tasks:

### Creating a New HTTP API Service

**Steps:**
1. Create project: `kratos new my-service`
2. Define Protocol Buffers in `api/` directory
3. Generate code: `kratos proto client api/myapi.proto`
4. Implement service layer in `internal/service/`
5. Implement business logic in `internal/biz/`
6. Implement data layer in `internal/data/`
7. Wire dependencies with `wire`

**Detailed guide**: [references/http-patterns.md](references/http-patterns.md#complete-http-api-workflow)

### Creating a New gRPC Service

**Steps:**
1. Define service in `.proto` file with messages and RPCs
2. Generate code: `kratos proto server api/myapi.proto`
3. Implement service logic in `internal/service/`
4. Configure gRPC server in `internal/server/grpc.go`
5. Register interceptors and middleware

**Detailed guide**: [references/grpc-patterns.md](references/grpc-patterns.md#complete-grpc-workflow)

### Implementing Clean Architecture

**Steps:**
1. Define domain models in `internal/biz/`
2. Define repository interfaces in `internal/biz/`
3. Implement repositories in `internal/data/`
4. Implement use cases in `internal/biz/`
5. Implement service handlers in `internal/service/`
6. Wire everything together with Wire

**Detailed guide**: [references/clean-architecture-patterns.md](references/clean-architecture-patterns.md#complete-clean-architecture-workflow)

### Adding Middleware

**Steps:**
1. Create middleware function following kratos middleware signature
2. Implement pre/post processing logic
3. Register middleware in server configuration
4. Chain multiple middlewares if needed

**Detailed guide**: [references/middleware-patterns.md](references/middleware-patterns.md#middleware-implementation)

## ⚡ Key Principles

When generating or reviewing go-kratos code, always apply these principles:

### ✅ Always Follow

- **Four-layer separation**: Keep Transport → Service → Biz → Data distinct
- **Dependency injection**: Use Wire for managing dependencies
- **Interface-driven design**: Define interfaces in Biz layer, implement in Data layer
- **Context propagation**: Pass `ctx context.Context` through all layers for tracing and cancellation
- **Protocol Buffers**: Use .proto files for API definitions
- **Error handling**: Use kratos errors package with proper error codes
- **Configuration**: Use kratos config with environment-specific files

### ❌ Never Do

- Put business logic directly in transport layer (violates clean architecture)
- Skip interface definitions (tight coupling)
- Hard-code dependencies (makes testing difficult)
- Skip context propagation (loses tracing and cancellation)
- Mix layers (business logic in data layer, etc.)
- Bypass Wire for dependency management

## 📖 Progressive Learning Path

Follow this path based on your needs:

### 🟢 New to go-kratos?

1. **Start here**: [Official go-kratos Quick Start](https://go-kratos.dev/docs/getting-started/start/)
   Install kratos CLI, create your first service, understand basic concepts

2. **Learn clean architecture**: [references/clean-architecture-patterns.md](references/clean-architecture-patterns.md)
   Understand Transport → Service → Biz → Data layers

3. **Build your first API**: [references/http-patterns.md](references/http-patterns.md)
   Create HTTP endpoints with proper patterns

### 🟡 Building production services?

1. **Review best practices**: [best-practices/overview.md](best-practices/overview.md)
   Configuration, logging, monitoring, security checklist

2. **Add middleware**: [references/middleware-patterns.md](references/middleware-patterns.md)
   Logging, metrics, tracing, authentication

3. **Check common pitfalls**: [troubleshooting/common-issues.md](troubleshooting/common-issues.md)
   Avoid typical mistakes and know how to debug issues

### 🔵 Extending capabilities?

1. **Use with Claude Code**: [getting-started/claude-code-guide.md](getting-started/claude-code-guide.md)
   Learn advanced features and workflows

2. **Explore examples**: [examples/](examples/)
   Run demo projects to validate your environment

## 🔗 Integration with go-kratos Ecosystem

This skill is part of the go-kratos ecosystem for AI-assisted development:

| Tool | Purpose | Best For |
|------|---------|----------|
| **kratos-skills** (this repo) | Comprehensive knowledge base | All AI tools, deep learning, reference |
| **kratos CLI** | Project scaffolding and code generation | Creating new projects, generating proto code |
| **Wire** | Dependency injection | Managing complex dependency graphs |

The AI runs `kratos` CLI directly in the terminal for code generation — no separate MCP server needed. See [references/kratos-cli-commands.md](references/kratos-cli-commands.md) for the complete command reference.

### Usage in Claude Code
- This skill loads automatically when working with go-kratos projects
- Use `/kratos-skills` to invoke manually for kratos guidance
- AI runs kratos CLI commands directly in the terminal for code generation
- Reference specific pattern files when needed (Claude loads them on demand)

See [getting-started/claude-code-guide.md](getting-started/claude-code-guide.md) for detailed usage instructions.

## 🌐 Additional Resources

- **Official docs**: [go-kratos.dev](https://go-kratos.dev) - Latest API reference and guides
- **GitHub**: [go-kratos/kratos](https://github.com/go-kratos/kratos) - Source code and examples
- **Community**: Discussions, issues, and contributions welcome in the main repository

## 📝 Version Compatibility

- **Target version**: go-kratos 2.x+
- **Go version**: Go 1.18 or later recommended
- **Updates**: Patterns updated regularly to reflect framework evolution
- **Breaking changes**: Check official docs for API changes between versions

---

**Quick invocation**: Use `/kratos-skills` or ask "How do I [task] with go-kratos?"
**Need help?** Reference the specific pattern guide for detailed examples and explanations.
