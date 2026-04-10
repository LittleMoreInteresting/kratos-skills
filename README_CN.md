# go-kratos Skills for AI Agents

[English](README.md) | 简体中文

这是一个 [Agent Skill](https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)，包含结构化的知识和模式，帮助 AI 编码助手有效地使用 [go-kratos](https://github.com/go-kratos/kratos) 框架。

## 什么是 Skill？

Skills 是 AI 代理动态发现和加载的指令、脚本和资源文件夹，用于在特定任务上表现更好。这个 skill 教 AI 代理如何生成生产就绪的 go-kratos 微服务代码。

## 目的

这个 skill 使 AI 代理（Claude、GitHub Copilot、Cursor 等）能够：
- 按照框架约定生成准确的 go-kratos 代码
- 理解四层架构（Transport → Service → Biz → Data）
- 应用整洁架构原则
- 使用 Wire 进行依赖注入
- 应用微服务开发最佳实践
- 高效排查常见问题
- 构建生产就绪的应用程序

## 快速安装

只需询问你的 AI 代理：

```
从  https://github.com/LittleMoreInteresting/kratos-skills 安装 kratos-skills
```

或手动安装：

```bash
# 项目级（推荐）
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .claude/skills/kratos-skills

# 个人级（所有项目）
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git ~/.claude/skills/kratos-skills
```

## Agent Skill 结构

遵循 [Agent Skills 规范](https://github.com/anthropics/skills/blob/main/spec/agent-skills-spec.md) 和 [Claude Code skills 文档](https://code.claude.com/docs/en/skills)：

```
kratos-skills/
├── SKILL.md                    # 入口点，包含 YAML 前置信息
├── getting-started/            # 入门指南
│   ├── README.md               # 工具比较概述
│   ├── claude-code-guide.md    # Claude Code（推荐）
│   ├── cursor-guide.md         # Cursor IDE
│   ├── copilot-guide.md        # GitHub Copilot
│   └── windsurf-guide.md       # Windsurf IDE
├── references/                 # 详细模式文档
│   ├── http-api-patterns.md    # HTTP API 开发模式
│   ├── grpc-patterns.md        # gRPC 服务模式
│   ├── clean-architecture-patterns.md  # 整洁架构模式
│   ├── data-patterns.md        # 数据层、ORM、事务、缓存
│   ├── error-patterns.md       # 错误处理和错误码定义
│   ├── validation-patterns.md  # 基于 proto 的输入验证
│   ├── tracing-patterns.md     # OpenTelemetry 追踪和指标
│   ├── registry-patterns.md    # 服务发现和注册
│   ├── event-patterns.md       # 基于 Kafka 的事件驱动架构
│   ├── cqrs-patterns.md        # CQRS 模式实现
│   ├── middleware-patterns.md  # 中间件和拦截器
│   └── kratos-cli-commands.md  # Kratos CLI 参考
├── best-practices/             # 生产环境建议
├── troubleshooting/            # 常见问题和解决方案
├── skill-patterns/             # 高级 skill 示例（模板）
│   ├── analyze-project.md      # 项目分析示例
│   ├── generate-service.md     # 生成服务示例
│   └── plan-architecture.md    # 架构规划示例
└── examples/                   # 演示项目和验证
```

## 使用此 Skill

### 使用 Claude Code（推荐）

Claude Code 原生支持 [Agent Skills 规范](https://agentskills.io/)。此 skill 针对 Claude Code 进行了优化，具有高级功能：

#### 项目级安装（Git 子模块）
将 kratos-skills 添加到项目中以自动发现：

```bash
# 添加为 git 子模块
git submodule add  https://github.com/LittleMoreInteresting/kratos-skills.git .claude/skills/kratos-skills

# 或直接克隆
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .claude/skills/kratos-skills
```

Claude Code 自动发现 `.claude/skills/` 目录中的 skills。

#### 个人级安装
要在所有项目中使用，安装到个人 skills 目录：

```bash
# 克隆到个人 skills 目录
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git ~/.claude/skills/kratos-skills
```

#### 在 Claude Code 中使用
- **自动**：当你处理 go-kratos 文件时（`.proto`、包含 kratos 的 `go.mod`），Claude 会自动加载 skill
- **手动**：输入 `/kratos-skills` 直接调用 go-kratos 指导
- **带参数**：`/kratos-skills 创建一个用户管理 API` 用于特定任务
- **检查可用性**：询问 "有哪些 skills 可用？" 查看是否已加载

#### 高级功能
- **动态上下文**：Skills 可以执行 shell 命令收集实时项目数据
- **子代理**：使用 `context: fork` 进行隔离的分析或规划任务
- **工具限制**：`allowed-tools` 确保安全的只读操作
- 查看 [skill-patterns/](skill-patterns/) 了解高级模式和模板

### 使用 Claude Desktop

添加到 `claude_desktop_config.json`：
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

### 使用 GitHub Copilot

查看 [copilot-guide.md](getting-started/copilot-guide.md) 了解详细设置。快速开始：

```bash
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .ai-context/kratos-skills
```

然后创建 `.github/copilot-instructions.md` 引用这些模式。

### 使用 Cursor

查看 [cursor-guide.md](getting-started/cursor-guide.md) 了解详细设置。快速开始：

```bash
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .ai-context/kratos-skills
```

然后创建 `.cursorrules` 引用这些模式。

### 使用 Windsurf

查看 [windsurf-guide.md](getting-started/windsurf-guide.md) 了解详细设置。快速开始：

```bash
git clone  https://github.com/LittleMoreInteresting/kratos-skills.git .ai-context/kratos-skills
```

然后创建 `.windsurfrules` 引用这些模式。

## 与 go-kratos 生态系统集成

kratos-skills 是 go-kratos AI 辅助开发生态系统的一部分：

| 工具 | 目的 | 大小 | 最适合 |
|------|------|------|--------|
| **kratos CLI** | 项目脚手架和代码生成 | ~10MB | 创建新项目 |
| **kratos-skills**（此仓库） | 综合知识库 | ~50KB | 所有 AI 工具，深度学习 |

AI 直接在终端中运行 `kratos` CLI 进行代码生成 —— 不需要单独的工具或服务器。查看 [references/kratos-cli-commands.md](references/kratos-cli-commands.md) 获取完整的命令参考。

### 它们如何协同工作

```
┌─────────────────────────────────────────────────────────────┐
│                     AI 助手                                  │
│  (Claude Code, GitHub Copilot, Cursor, Windsurf)           │
└────────────┬─────────────────────┬──────────────────────────┘
             │                     │
             ├─ 工具层 ────────────┤
             │  kratos CLI         │  "生成代码" - 项目脚手架
             │  (~10MB)            │  Proto 生成，Wire
             │                     │
             └─ 知识层 ────────────┘
                kratos-skills        "如何和为什么" - 详细模式
                (~50KB)              + CLI 命令参考
                                     按需加载
```

### 使用场景

**场景 1：Claude Code 用户（最佳体验）**
- 使用：`kratos-skills`（此仓库）作为原生 skill
- 好处：
  - 来自模式指南的深度学习
   - AI 直接在终端中运行 kratos CLI 命令
  - 带有实时项目数据的动态上下文
  - 用于复杂任务的子代理工作流
- 调用：`/kratos-skills` 或在处理 kratos 时自动

**场景 2：GitHub Copilot 用户**
- 使用：kratos-skills（通过 `.github/copilot-instructions.md` 加载）
- 好处：快速内联建议，工作流指导，通过终端使用 kratos CLI

**场景 3：Cursor/Windsurf 用户**
- 使用：kratos-skills（在项目规则中）
- 好处：IDE 原生体验，带有 kratos 指导，通过终端使用 kratos CLI

查看 [入门指南](getting-started/) 了解每个工具的详细集成说明。

## 快速链接

**Skill 文档：**

- 📖 **[SKILL.md](SKILL.md)** - 主 skill 入口点和导航
- 📚 **[go-kratos 快速开始](https://go-kratos.dev/docs/getting-started/start/)** - 官方 go-kratos 框架教程
- 🎯 **[高级示例](skill-patterns/)** - 子代理，动态上下文等

**入门指南：**

- 💡 **[Claude Code](getting-started/claude-code-guide.md)** - 完整功能，子代理（推荐）
- 🖱️ **[Cursor](getting-started/cursor-guide.md)** - 使用 .cursorrules 的 IDE 集成
- 🤖 **[GitHub Copilot](getting-started/copilot-guide.md)** - VS Code 内联建议
- 🏄 **[Windsurf](getting-started/windsurf-guide.md)** - Cascade AI 集成
- 📋 **[工具比较](getting-started/README.md)** - 比较所有工具

## 贡献

欢迎贡献！请确保：
- 示例完整且经过测试
- 模式遵循官方 go-kratos 约定
- 内容结构适合 AI 消费
- 包含正确（✅）和错误（❌）的示例
- 遵循 [Agent Skills 规范](https://agentskills.io/)

## 许可证

MIT 许可证 - 与 go-kratos 框架相同
