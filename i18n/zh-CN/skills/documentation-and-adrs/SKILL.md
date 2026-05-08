---
name: documentation-and-adrs
description: 编写项目文档和架构决策记录（ADR）。适用于记录重大架构决策、创建或更新 README，以及编写代码内联文档时使用。
---

# 文档与 ADR

## 概述

用最少的努力捕获最重要的上下文。好的文档不是尽可能多地写——而是写对的东西，让下一个开发者（或六个月后的你）理解为什么代码是这样的。

**文档的核心目标：**
- 解释代码本身无法告诉你的内容
- 记录不明显的决策及其原因
- 使新开发者能够快速上手

## 适用场景

- 做出重大架构或技术决策时
- 创建新项目或服务时
- 添加非显而易见的代码时
- 更新破坏现有行为的 API 时
- 设置有许多配置步骤的系统时

**不适用场景：** 为显而易见的代码添加注释（`// increment counter`），创建在代码更改时不会更新的注释。

## 架构决策记录（ADR）

ADR 是捕获重要架构决策的轻量级文档。在做出改变方向的决策时始终写 ADR：数据库选择、框架、认证策略、部署方法，或任何带有非显而易见权衡的设计选择。

### ADR 模板

```markdown
# ADR-{number}: {Short title}

**Date:** {YYYY-MM-DD}
**Status:** {Proposed | Accepted | Deprecated | Superseded by ADR-N}

## Context

{Describe the situation and the problem we were solving. What forces are at play?
What constraints exist? Why does a decision need to be made now?}

## Decision

{State the decision clearly. "We will use X to solve Y."}

## Rationale

{Explain why this option was chosen. What alternatives were considered?
Why were they rejected? What tradeoffs are accepted?}

## Consequences

### Positive
- {List the benefits}

### Negative
- {List the costs, risks, or tradeoffs accepted}

### Neutral
- {Changes that are neither good nor bad, but important to know}
```

### ADR 示例

```markdown
# ADR-007: Use PostgreSQL for primary data storage

**Date:** 2025-03-15
**Status:** Accepted

## Context

We need a database for the task management application. The app requires:
- ACID transactions (task creation + assignment must be atomic)
- Complex queries with filtering and sorting
- JSON storage for flexible task metadata
- Team is experienced with relational databases

## Decision

We will use PostgreSQL as the primary data store, accessed via Prisma ORM.

## Rationale

**Considered:**
- **PostgreSQL**: Strong ACID compliance, JSON support, great tooling, team expertise
- **MongoDB**: Flexible schema, but weaker transactions and the team has less experience
- **SQLite**: Simple, but doesn't scale to multi-user concurrent writes

PostgreSQL satisfies all requirements and the team can move fast with it.
Prisma provides type-safe access and simplifies migrations.

## Consequences

### Positive
- Strong consistency guarantees for financial/task operations
- Native JSON columns for flexible metadata
- Rich query capabilities for filtering/reporting
- Type-safe ORM reduces runtime errors

### Negative
- More complex setup than SQLite for local development
- Schema changes require migrations (but Prisma handles this well)
```

### 在哪里存储 ADR

```
docs/
  adr/
    README.md          # Index of all ADRs
    001-use-nextjs.md
    002-use-postgres.md
    003-auth-strategy.md
```

## README 结构

好的 README 让开发者在 10 分钟内开始工作。它不需要记录每一个功能——只需要让人能够运行它并理解它。

### 必需部分

```markdown
# Project Name

One sentence: what does this do and for whom?

## Quick Start

The minimal steps to go from nothing to running:

\`\`\`bash
git clone https://github.com/org/repo
cd repo
cp .env.example .env    # Configure your environment
npm install
npm run dev             # Start at http://localhost:3000
\`\`\`

## Development

\`\`\`bash
npm run dev          # Start development server
npm test             # Run tests
npm run build        # Production build
npm run lint         # Lint and format
\`\`\`

## Project Structure

\`\`\`
src/
  app/          # Next.js app directory
  services/     # Business logic
  db/           # Database access layer
  components/   # Reusable UI components
\`\`\`

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `NEXTAUTH_SECRET` | Yes | Auth token signing secret |
| `STRIPE_KEY` | No | Payment processing (optional in dev) |

## Architecture

Brief overview of key design decisions. Link to ADRs for details.

## Contributing

Link to CONTRIBUTING.md or brief instructions.
```

### 可选部分（按需添加）

```markdown
## Deployment
## API Reference
## Troubleshooting
## License
```

**规则：** 如果 README 超过 300 行，将内容提取到单独文件并在 README 中链接。

## 内联代码文档

### 注释什么

注释**为什么**，而不是**做什么**。代码已经告诉你它做什么——注释告诉你原因、上下文或非显而易见的权衡。

```typescript
// ✗ What (obvious from code):
// Increment the counter
counter++;

// ✓ Why (not obvious):
// Rate limit uses a sliding window of 60s rather than a fixed bucket
// to prevent burst abuse at window boundaries
const windowStart = Math.floor(timestamp / 60000) * 60000;
```

### 何时添加注释

```typescript
// Performance optimization with data to back it up
// Caching here reduced average response time from 450ms to 12ms
// Cache TTL is 5 minutes — acceptable staleness for task counts
const cachedCount = await cache.get(`task-count:${userId}`);

// Non-obvious dependency or ordering requirement
// Auth middleware must run before this — it sets req.user
// If auth is removed, this will throw "Cannot read property 'id' of undefined"
const tasks = await taskService.getTasksForUser(req.user.id);

// Intentional deviation from expected pattern
// We use a raw SQL query here because Prisma doesn't support
// the lateral join needed for this particular aggregation
const stats = await db.$queryRaw`...`;

// Known issue or future work
// TODO: This doesn't handle timezone differences.
// Users in non-UTC zones will see incorrect "due today" counts.
// Tracking: https://github.com/org/repo/issues/234
```

### JSDoc / TSDoc 用于公共 API

```typescript
/**
 * Creates a new task and notifies the assignee.
 *
 * @param input - Task creation parameters
 * @param createdBy - ID of the user creating the task
 * @returns The created task with server-generated fields
 * @throws {ValidationError} If input fails validation
 * @throws {NotFoundError} If assignee doesn't exist
 *
 * @example
 * const task = await createTask(
 *   { title: 'Review PR', assigneeId: 'user-123' },
 *   'user-456'
 * );
 */
async function createTask(
  input: CreateTaskInput,
  createdBy: string
): Promise<Task> {
  ...
}
```

为公共 API 表面使用 JSDoc——被项目外部消费者使用的函数、类和接口。不要为只在一个文件中使用的内部函数添加 JSDoc。

## 变更日志

记录对用户可见的变更，而不是内部实现细节：

```markdown
## [2.3.0] - 2025-04-01

### Added
- Task labels: assign up to 5 color-coded labels per task
- Bulk operations: select multiple tasks and complete/delete at once

### Changed
- Task list now loads 50% faster due to query optimization

### Fixed
- Fixed task due dates showing incorrectly in UTC+8 timezones

### Deprecated
- `GET /api/tasks/all` is deprecated. Use `GET /api/tasks` with pagination instead.
  Will be removed in v3.0.0 (2025-Q4).

## [2.2.1] - 2025-03-20

### Fixed
- Fixed crash when task title contained emoji characters
```

**规则：** 每次用户可见的变更都更新变更日志。内部重构和依赖更新不需要条目。

## 文档维护

过时的文档比没有文档更危险——它会主动误导。

**规则：**
- 更改行为时更新 README
- 添加/更改公共 API 时更新 JSDoc
- 当架构决策被推翻时，在旧 ADR 中添加 `Superseded by ADR-N`
- 删除在代码更改时变为错误的注释

```typescript
// ✗ Stale comment (the code doesn't do this anymore):
// This calls the legacy payment processor
const result = await newStripeProcessor.charge(amount);

// ✓ Updated to reflect reality:
// Charges via Stripe. For PayPal, see processPayPalPayment()
const result = await newStripeProcessor.charge(amount);
```

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "代码是自文档化的" | 代码告诉你_什么_，注释和 ADR 告诉你_为什么_——那才是真正难以记住的。 |
| "我们稍后再补文档" | 六个月后没有人记得为什么做了那个决定。现在写 ADR。 |
| "ADR 太形式化了，我们只是一个小团队" | 小团队尤其需要 ADR——他们没有大量的历史记录可以审查。 |
| "README 已经有了" | README 是最后一次更新于三个版本之前吗？更新它。 |

## 危险信号

- 没有 README 或 README 不能让你运行项目
- 没有记录数据库/框架选择的 ADR
- 解释了"做什么"但不解释"为什么"的注释
- README 中的过时命令
- 多个标记为"已接受"但相互矛盾的 ADR（没有"取代"链接）
- 有 `TODO: document this` 但从未完成

## 验证

添加或更新文档之后：

- [ ] README 中的快速入门经过测试并有效
- [ ] 所有环境变量都记录在案
- [ ] 重大架构决策有 ADR
- [ ] 非显而易见的代码有解释"为什么"的注释
- [ ] 公共 API 有 JSDoc 示例
- [ ] 变更日志为用户可见的变更更新
- [ ] 过时的文档已移除或更新
