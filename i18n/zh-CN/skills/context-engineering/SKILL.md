---
name: context-engineering
description: 构建 AI agent 上下文，包括规则文件、MCP 集成和系统提示。适用于需要设置 Claude Code 或其他 AI agent 的项目时使用。
---

# 上下文工程

## 概述

系统性地塑造 AI agent 可以访问的信息——规则、工具、文档和示例——以提高其在特定代码库中的效果。上下文工程使 agent 表现得更像了解你项目的高级工程师，而不是从未见过代码库的泛化助手。

## 适用场景

- 为新项目设置 Claude Code 或其他 AI coding agent
- Agent 一遍又一遍地犯同样的错误
- Agent 不遵循你的技术栈或约定
- 你需要 agent 访问外部工具（数据库、API、文档）
- 设置团队共享的 agent 配置

## 上下文层次结构

```
SYSTEM LEVEL
└── Platform instructions (model training, built-in behaviors)

PROJECT LEVEL
├── CLAUDE.md (project rules and conventions)
├── .mcp.json (MCP server integrations)
└── Project knowledge base

SESSION LEVEL
├── Conversation history
├── Opened files
└── Session-specific instructions

TASK LEVEL
└── Immediate prompt and context
```

**关键洞见：** Agent 在同一时刻工作的上下文是有限的。明确提供关键信息比依赖 agent "记住"或推断要好得多。

## CLAUDE.md：项目规则文件

`CLAUDE.md` 是在每次 Claude Code 会话开始时自动加载的指令文件。将其理解为你想让每个开发者（人类或 AI）在第一天就了解的内容。

### 结构模板

```markdown
# Project: [Name]

## Overview
[2-3 sentence description of what this project does]

## Tech Stack
- Runtime: [e.g., Node.js 22 with TypeScript]
- Framework: [e.g., Next.js 15]
- Database: [e.g., PostgreSQL with Prisma]
- Testing: [e.g., Vitest + Playwright]
- Deployment: [e.g., Vercel]

## Project Structure
[Describe key directories and their purpose]

## Development Commands
\`\`\`bash
npm run dev          # Start development server
npm test             # Run unit tests
npm run test:e2e     # Run end-to-end tests
npm run build        # Production build
npm run lint         # Lint and format
\`\`\`

## Conventions

### Code Style
[Specific conventions: TypeScript strict mode, import order, etc.]

### Naming
[Naming conventions: components, variables, files]

### Architecture
[Key patterns: service layer, data access, etc.]

### Testing
[Testing philosophy, what to test, testing tools]

## Important Constraints
[Things that must NEVER happen: secrets in code, skipping migrations, etc.]

## External Services
[Key integrations and how they're configured]
```

### 有效 CLAUDE.md 的原则

**具体而非模糊：**

```markdown
# ✗ Too vague
Write clean code and follow best practices.

# ✓ Specific and actionable
Use TypeScript strict mode. All functions must have explicit return types.
Prefer named exports over default exports except for page components.
Use Zod for all external data validation.
```

**记录"为什么"，不只是"什么"：**

```markdown
# ✓ Explains context
Use server-side pagination for all list endpoints. Our primary table has 2M+ rows;
client-side filtering would exceed memory limits and timeout.
```

**仅包含持续相关的内容：**

```markdown
# ✗ Too granular (this belongs in code comments, not CLAUDE.md)
The createTask function takes title, description, and priority.

# ✓ Convention that applies everywhere
All API endpoints must validate input with Zod before processing.
```

### CLAUDE.md 应包含的内容

```
✓ Tech stack versions
✓ Project structure description
✓ Development commands
✓ Code conventions (naming, formatting, imports)
✓ Architecture patterns (service layers, data access)
✓ Testing philosophy and tools
✓ Hard constraints ("never commit secrets", "never skip migrations")
✓ Key external service integrations
✓ Common gotchas ("the auth middleware must be first in Express")
```

### CLAUDE.md 不应包含的内容

```
✗ Secrets or credentials
✗ One-time instructions ("for this PR, do X")
✗ Overly detailed implementation instructions
✗ Information already obvious from the code
✗ Things that change per task or session
```

## MCP 集成

模型上下文协议（MCP）服务器为 agent 提供工具和资源访问。配置 MCP 服务器使 agent 能够：

- 查询你的数据库
- 搜索文档
- 与外部 API 交互
- 访问文件系统操作
- 读取浏览器状态

### 配置 MCP 服务器

```json
// .mcp.json (项目级别)
{
  "mcpServers": {
    "database": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-postgres@latest"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": [
        "@modelcontextprotocol/server-filesystem@latest",
        "/path/to/project"
      ]
    },
    "github": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-github@latest"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

### 常用 MCP 服务器

| 服务器 | 功能 | 适用场景 |
|--------|------|----------|
| `@modelcontextprotocol/server-postgres` | 查询 PostgreSQL 数据库 | 数据库操作和调试 |
| `@modelcontextprotocol/server-filesystem` | 文件系统读写操作 | 项目代码访问 |
| `@modelcontextprotocol/server-github` | GitHub Issues、PR、代码搜索 | 代码库导航和 issue 管理 |
| `@anthropic/chrome-devtools-mcp` | Chrome DevTools 集成 | 浏览器测试和调试 |
| `@modelcontextprotocol/server-brave-search` | 网络搜索 | 文档查找和研究 |

**安全注意事项：** MCP 服务器可以访问真实系统——数据库、文件、API。通过环境变量注入凭证（而不是硬编码到配置中），只提供必要的访问权限，并对生产 MCP 访问格外谨慎。

## 通过示例进行上下文工程

当 agent 一次又一次地以同样的方式出错时，提供具体的反例示例：

```markdown
## Example: API Route Pattern

Always follow this pattern for API routes:

\`\`\`typescript
// ✓ CORRECT: Validate → Process → Return
app.post('/api/tasks', authenticate, async (req, res) => {
  const result = CreateTaskSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(422).json({ error: result.error.flatten() });
  }

  const task = await taskService.create(result.data, req.user.id);
  res.status(201).json(task);
});

// ✗ WRONG: Never put validation in the service layer
// ✗ WRONG: Never return 200 for resource creation (use 201)
// ✗ WRONG: Never skip authentication on mutation endpoints
\`\`\`
```

## 上下文窗口管理

Agent 具有有限的上下文窗口。当会话变长时，之前的上下文可能会"被遗忘"。

### 策略

**保持 CLAUDE.md 简洁：** 只包含始终相关的内容。15 分钟规则：如果一条规则不影响 15 分钟后的代码决策，它不属于这里。

**使用 MCP 而不是粘贴文档：** 如果你需要 agent 访问 API 文档，设置一个文档 MCP 服务器，而不是将文档粘贴到聊天中。

**提炼上下文：** 如果会话变长，对当前状态进行总结："这是迄今为止我们已经做到的和什么还需要做……"

**新会话重要变更：** 复杂的功能开发最好在新的会话中继续，其中总结了已完成的内容。

## 系统提示（适用于 Agent API）

在直接使用 Anthropic API 构建 agent 时，使用系统提示来提供持久上下文：

```typescript
const response = await anthropic.messages.create({
  model: 'claude-opus-4-5',
  system: `You are a senior engineer working on [Project Name].

Tech stack: TypeScript, Next.js 15, PostgreSQL, Prisma, Tailwind CSS.

Key conventions:
- All API routes validate with Zod
- Use server components by default, client components only when needed
- Database access only through the service layer, never directly in routes
- All sensitive operations require authentication
- Follow the existing patterns in /src/app/api/* for new endpoints

Current codebase summary:
[Brief description of key modules and their purposes]

When making changes:
1. Check existing patterns first
2. Follow the conventions above
3. Write tests for new functionality`,

  messages: [
    { role: 'user', content: 'Add an endpoint to archive tasks' }
  ]
});
```

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "Agent 应该能从代码中推断出我的约定" | Agent 每次会话都从空白状态开始。明确胜于隐式。 |
| "CLAUDE.md 太耗时了，我就快点告诉它" | 口头指令在会话间不持久。CLAUDE.md 一次写好，永久生效。 |
| "MCP 设置太复杂了" | 数据库 MCP 让 agent 在 5 分钟内就能调试实时数据，而不是猜测。 |
| "Agent 每次会话后都会记住" | 不会。每个会话都从新开始。将项目知识放在文件中，而不是依赖记忆。 |

## 危险信号

- CLAUDE.md 不存在或为空
- Agent 一再违背同样的约定
- 将凭证硬编码到 MCP 配置中
- 给 MCP 服务器过多的系统访问权限
- CLAUDE.md 包含过期或矛盾的信息
- 没有 MCP 集成的代码库使 agent 无法检查实时数据

## 验证

设置或更新上下文工程后：

- [ ] CLAUDE.md 存在并包含技术栈、约定和关键约束
- [ ] 所有命令都已测试并可正常工作
- [ ] 没有在 CLAUDE.md 中存储密钥或凭证
- [ ] MCP 服务器（如果配置）通过环境变量获取凭证
- [ ] 上下文文件已提交到版本控制
- [ ] 在新会话中测试：agent 是否遵循约定？
