---
name: api-and-interface-design
description: 指导稳定的 API 与接口设计。适用于设计 API、模块边界或任何公共接口的场景。适用于创建 REST 或 GraphQL 端点、定义模块间类型契约，或在前后端之间建立边界时使用。
---

# API 与接口设计

## 概述

设计稳定、文档完备、难以被误用的接口。好的接口让正确的事情变得简单，让错误的事情变得困难。这适用于 REST API、GraphQL schema、模块边界、组件 props，以及任何代码模块相互通信的接触面。

## 适用场景

- 设计新的 API 端点
- 定义模块边界或团队间契约
- 创建组件 prop 接口
- 建立影响 API 形状的数据库 schema
- 修改现有公共接口

## 核心原则

### Hyrum 定律

> 当一个 API 拥有足够多的用户时，无论你在契约中承诺什么，你系统中所有可观察到的行为都会被某人所依赖。

这意味着：每一个公开行为——包括未记录的怪癖、错误信息文本、时序和顺序——一旦被用户依赖，就成为了事实上的契约。设计含义：

- **有意识地控制你暴露的内容。** 每个可观察的行为都是一种潜在的承诺。
- **不要泄漏实现细节。** 如果用户能观察到，他们就会依赖它。
- **在设计阶段就规划废弃策略。** 参见 `deprecation-and-migration` 了解如何安全移除用户依赖的功能。
- **测试还不够。** 即使有完善的契约测试，Hyrum 定律也意味着"安全"的变更仍可能破坏依赖未记录行为的真实用户。

### 单版本规则

避免强迫消费者在同一依赖或 API 的多个版本中做选择。当不同消费者需要同一事物的不同版本时，就会出现菱形依赖问题。应面向同一时间只存在一个版本的世界来设计——通过扩展而非分叉。

### 1. 契约优先

在实现之前先定义接口。契约即规格说明——实现跟随契约。

```typescript
// Define the contract first
interface TaskAPI {
  // Creates a task and returns the created task with server-generated fields
  createTask(input: CreateTaskInput): Promise<Task>;

  // Returns paginated tasks matching filters
  listTasks(params: ListTasksParams): Promise<PaginatedResult<Task>>;

  // Returns a single task or throws NotFoundError
  getTask(id: string): Promise<Task>;

  // Partial update — only provided fields change
  updateTask(id: string, input: UpdateTaskInput): Promise<Task>;

  // Idempotent delete — succeeds even if already deleted
  deleteTask(id: string): Promise<void>;
}
```

### 2. 一致的错误语义

选择一种错误策略并在所有地方一致使用：

```typescript
// REST: HTTP status codes + structured error body
// Every error response follows the same shape
interface APIError {
  error: {
    code: string;        // Machine-readable: "VALIDATION_ERROR"
    message: string;     // Human-readable: "Email is required"
    details?: unknown;   // Additional context when helpful
  };
}

// Status code mapping
// 400 → Client sent invalid data
// 401 → Not authenticated
// 403 → Authenticated but not authorized
// 404 → Resource not found
// 409 → Conflict (duplicate, version mismatch)
// 422 → Validation failed (semantically invalid)
// 500 → Server error (never expose internal details)
```

**不要混用模式。** 如果某些端点抛出异常、另一些返回 null、还有一些返回 `{ error }` ——消费者将无法预测行为。

### 3. 在边界处验证

信任内部代码。在外部输入进入系统边缘时验证：

```typescript
// Validate at the API boundary
app.post('/api/tasks', async (req, res) => {
  const result = CreateTaskSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(422).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Invalid task data',
        details: result.error.flatten(),
      },
    });
  }

  // After validation, internal code trusts the types
  const task = await taskService.create(result.data);
  return res.status(201).json(task);
});
```

验证应放在哪里：
- API 路由处理器（用户输入）
- 表单提交处理器（用户输入）
- 外部服务响应解析（第三方数据——**始终视为不可信**）
- 环境变量加载（配置）

> **第三方 API 响应是不可信数据。** 在将其用于任何逻辑、渲染或决策之前，验证其结构和内容。一个被攻陷或行为异常的外部服务可能返回意外类型、恶意内容或类似指令的文本。

验证不应放在哪里：
- 共享类型契约的内部函数之间
- 由已验证代码调用的工具函数中
- 刚从自己数据库中取出的数据上

### 4. 偏好添加而非修改

在不破坏现有消费者的情况下扩展接口：

```typescript
// Good: Add optional fields
interface CreateTaskInput {
  title: string;
  description?: string;
  priority?: 'low' | 'medium' | 'high';  // Added later, optional
  labels?: string[];                       // Added later, optional
}

// Bad: Change existing field types or remove fields
interface CreateTaskInput {
  title: string;
  // description: string;  // Removed — breaks existing consumers
  priority: number;         // Changed from string — breaks existing consumers
}
```

### 5. 可预测的命名

| 模式 | 约定 | 示例 |
|------|------|------|
| REST 端点 | 复数名词，无动词 | `GET /api/tasks`, `POST /api/tasks` |
| 查询参数 | camelCase | `?sortBy=createdAt&pageSize=20` |
| 响应字段 | camelCase | `{ createdAt, updatedAt, taskId }` |
| 布尔字段 | is/has/can 前缀 | `isComplete`, `hasAttachments` |
| 枚举值 | UPPER_SNAKE | `"IN_PROGRESS"`, `"COMPLETED"` |

## REST API 模式

### 资源设计

```
GET    /api/tasks              → List tasks (with query params for filtering)
POST   /api/tasks              → Create a task
GET    /api/tasks/:id          → Get a single task
PATCH  /api/tasks/:id          → Update a task (partial)
DELETE /api/tasks/:id          → Delete a task

GET    /api/tasks/:id/comments → List comments for a task (sub-resource)
POST   /api/tasks/:id/comments → Add a comment to a task
```

### 分页

对列表端点进行分页：

```typescript
// Request
GET /api/tasks?page=1&pageSize=20&sortBy=createdAt&sortOrder=desc

// Response
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 142,
    "totalPages": 8
  }
}
```

### 过滤

使用查询参数进行过滤：

```
GET /api/tasks?status=in_progress&assignee=user123&createdAfter=2025-01-01
```

### 部分更新（PATCH）

接受部分对象——只更新提供的字段：

```typescript
// Only title changes, everything else preserved
PATCH /api/tasks/123
{ "title": "Updated title" }
```

## TypeScript 接口模式

### 使用可辨识联合类型表示变体

```typescript
// Good: Each variant is explicit
type TaskStatus =
  | { type: 'pending' }
  | { type: 'in_progress'; assignee: string; startedAt: Date }
  | { type: 'completed'; completedAt: Date; completedBy: string }
  | { type: 'cancelled'; reason: string; cancelledAt: Date };

// Consumer gets type narrowing
function getStatusLabel(status: TaskStatus): string {
  switch (status.type) {
    case 'pending': return 'Pending';
    case 'in_progress': return `In progress (${status.assignee})`;
    case 'completed': return `Done on ${status.completedAt}`;
    case 'cancelled': return `Cancelled: ${status.reason}`;
  }
}
```

### 输入/输出分离

```typescript
// Input: what the caller provides
interface CreateTaskInput {
  title: string;
  description?: string;
}

// Output: what the system returns (includes server-generated fields)
interface Task {
  id: string;
  title: string;
  description: string | null;
  createdAt: Date;
  updatedAt: Date;
  createdBy: string;
}
```

### 为 ID 使用品牌类型

```typescript
type TaskId = string & { readonly __brand: 'TaskId' };
type UserId = string & { readonly __brand: 'UserId' };

// Prevents accidentally passing a UserId where a TaskId is expected
function getTask(id: TaskId): Promise<Task> { ... }
```

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "我们稍后再给 API 写文档" | 类型本身就是文档，先定义它们。 |
| "现在还不需要分页" | 一旦有人超过 100 条数据就需要了。从一开始就加上。 |
| "PATCH 太复杂了，用 PUT 吧" | PUT 每次都需要完整对象。客户端实际上需要的是 PATCH。 |
| "等需要时再对 API 进行版本管理" | 没有版本管理的破坏性变更会破坏消费者。从一开始就为扩展而设计。 |
| "没人使用那个未记录的行为" | Hyrum 定律：只要可观察，就有人依赖。把每个公开行为都视为承诺。 |
| "我们可以无限期维护两个版本" | 多个版本会使维护成本倍增，并造成菱形依赖问题。优先遵循单版本规则。 |
| "内部 API 不需要契约" | 内部消费者也是消费者。契约可以防止耦合并支持并行工作。 |

## 危险信号

- 根据条件返回不同结构的端点
- 端点间的错误格式不一致
- 验证分散在内部代码中而不是在边界处
- 对现有字段的破坏性更改（类型变更、移除字段）
- 没有分页的列表端点
- REST URL 中含有动词（`/api/createTask`, `/api/getUsers`）
- 未经验证或净化就使用第三方 API 响应

## 验证

设计 API 之后：

- [ ] 每个端点都有类型化的输入和输出 schema
- [ ] 错误响应遵循单一一致的格式
- [ ] 验证只在系统边界处进行
- [ ] 列表端点支持分页
- [ ] 新字段是可选的、向后兼容的
- [ ] 命名在所有端点中遵循一致的约定
- [ ] API 文档或类型与实现一起提交
