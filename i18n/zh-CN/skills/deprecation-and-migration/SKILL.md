---
name: deprecation-and-migration
description: 安全地弃用旧 API 和迁移代码。当需要移除功能、重命名接口或将用户从旧模式迁移到新模式时使用。
---

# 弃用与迁移

## 概述

安全地从系统中移除旧代码，不破坏用户和消费者，同时不使技术债无限期积累。弃用是一个过程，而不是一个时刻——它有明确的阶段、沟通要求，以及干净的完成定义。

## 适用场景

- 重命名或重构公共 API
- 用更好的替代方案替换旧功能
- 移除不再需要的功能
- 将代码库迁移到新的模式
- 清理已积累的技术债

**不适用场景：** 在单个 PR 中重命名内部函数——直接重命名，不需要弃用过程。弃用适用于有真实消费者的公共接口。

## 弃用决策

在弃用之前，回答这些问题：

```
Is there a better alternative?
  └── No → Fix or improve the existing API instead
      Yes → Continue

Who are the consumers?
  ├── Internal only → Migrate all at once in one PR
  └── External users → Follow the full deprecation process

What's the migration path?
  └── Must be clear BEFORE announcing deprecation

What's the timeline?
  └── Short (weeks)? Long (months)? Coordinate with consumers
```

## 弃用阶段

```
Phase 1: SOFT DEPRECATION
  ├── Add @deprecated annotation in code
  ├── Add deprecation warning in logs/headers
  ├── Document the migration path
  └── Communicate timeline to consumers

Phase 2: HARD DEPRECATION
  ├── Old code still works but actively warns
  ├── Migration guide is public and tested
  ├── Monitor adoption of new API
  └── Set a removal date

Phase 3: REMOVAL
  ├── Confirm all consumers have migrated
  ├── Remove old code in one clean PR
  └── Archive or close related issues/PRs
```

## 迁移模式

### Strangler Fig（绞杀无花果）模式

用于将大型旧系统迁移到新系统，无需大爆炸重写。

```
Old System          New System
    │                   │
    ▼                   ▼
Requests →→→ Router →→→ decides
                │
                ├── Old feature → Old System
                └── New feature → New System

Gradually move features from Old to New until Old is empty
```

```typescript
// Router pattern
class TaskRouter {
  async getTask(id: string): Promise<Task> {
    // Feature flag controls which system handles the request
    if (featureFlags.isEnabled('new-task-service', { taskId: id })) {
      return this.newTaskService.getTask(id);
    }
    return this.legacyTaskService.getTask(id);
  }
}
```

这对于以下情况有用：
- 迁移大型服务，而不是单个功能
- 需要渐进式推出以减少风险
- 旧系统和新系统需要并行运行一段时间

### 适配器模式

当新 API 具有不同的接口，但需要与期望旧接口的消费者向后兼容时。

```typescript
// New API: cleaner interface
interface TaskServiceV2 {
  createTask(input: CreateTaskInput): Promise<Task>;
  listTasks(params: ListTasksParams): Promise<Task[]>;
}

// Old API: original interface
interface TaskServiceV1 {
  addTask(title: string, description: string): Promise<{ taskId: string }>;
  getTasks(userId: string, page: number): Promise<{ tasks: Task[], total: number }>;
}

// Adapter: wraps V2 to satisfy V1 interface
class TaskServiceV1Adapter implements TaskServiceV1 {
  constructor(private v2: TaskServiceV2) {}

  async addTask(title: string, description: string) {
    const task = await this.v2.createTask({ title, description });
    return { taskId: task.id };
  }

  async getTasks(userId: string, page: number) {
    const tasks = await this.v2.listTasks({ userId, page });
    return { tasks, total: tasks.length };
  }
}
```

### 功能标志迁移

当变更需要在不同用户群中测试或分阶段推出时。

```typescript
// Control the rollout via feature flags
async function getTasksForUser(userId: string) {
  if (featureFlags.isEnabled('paginated-task-response', { userId })) {
    // New: paginated response
    return taskService.listWithPagination({ userId, page: 1, pageSize: 20 });
  }
  // Old: returns all tasks
  return taskService.listAll({ userId });
}
```

## 代码注释中的弃用标注

### TypeScript / JavaScript

```typescript
/**
 * @deprecated Use `createTask(input: CreateTaskInput)` instead.
 * This function will be removed in v3.0.0 (2025-Q3).
 * Migration guide: https://docs.example.com/migrate/v3
 */
function addTask(title: string, userId: string): Promise<Task> {
  console.warn(
    '[DEPRECATED] addTask() is deprecated. Use createTask() instead. ' +
    'See https://docs.example.com/migrate/v3 for migration guide.'
  );
  return createTask({ title, userId });
}
```

### Python

```python
import warnings

def add_task(title: str, user_id: str) -> Task:
    """
    Deprecated: Use create_task(input: CreateTaskInput) instead.
    Will be removed in v3.0.0 (2025-Q3).
    """
    warnings.warn(
        "add_task() is deprecated, use create_task() instead. "
        "See https://docs.example.com/migrate/v3",
        DeprecationWarning,
        stacklevel=2
    )
    return create_task(CreateTaskInput(title=title, user_id=user_id))
```

### REST API（HTTP 头部）

```
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 01 Jan 2026 00:00:00 GMT
Link: <https://api.example.com/v2/tasks>; rel="successor-version"
Warning: 299 - "This endpoint is deprecated. See https://docs.example.com/migrate/v2"
```

## 僵尸代码

僵尸代码是声明已弃用但从未真正移除的代码。它是最危险的技术债形式：

- **消耗维护时间：** 人们继续更新从未使用的代码
- **产生混淆：** 新开发者不知道是否应该使用它
- **制造虚假安全感：** 延迟的删除会一直延迟

**规则：** 弃用时设置一个日期——要么作为日历提醒，要么作为代码中的 TODO：

```typescript
// TODO: Remove this by 2025-10-01 — all consumers migrated to createTask()
// Tracking: https://github.com/org/repo/issues/456
/** @deprecated */
function addTask(title: string): Promise<Task> { ... }
```

## 迁移指南写作

好的迁移指南包含：

```markdown
## Migrating from addTask() to createTask()

### What changed
`addTask(title, userId)` has been replaced with `createTask(input)` which
supports additional fields and returns a richer Task object.

### Before
\`\`\`typescript
const task = await taskService.addTask("Buy groceries", userId);
// Returns: { taskId: string }
\`\`\`

### After
\`\`\`typescript
const task = await taskService.createTask({
  title: "Buy groceries",
  userId: userId,
});
// Returns: Task object with id, title, createdAt, updatedAt
\`\`\`

### Step-by-step migration
1. Find all calls to `addTask()` in your codebase
2. Replace with `createTask()` using the pattern above
3. Update any code that reads `.taskId` to read `.id` instead
4. Run your tests

### Need help?
Open an issue at https://github.com/org/repo/issues
```

## 版本控制

对于外部 API，使用语义化版本控制（SemVer）传达破坏性变更的重要性：

```
MAJOR version (v1 → v2): Breaking changes (removed features, changed interfaces)
MINOR version (v1.0 → v1.1): New features, backward-compatible
PATCH version (v1.0.0 → v1.0.1): Bug fixes, no interface changes
```

**弃用时序（外部 API）：**

```
v1.5.0 → Deprecate addTask(), introduce createTask()
v2.0.0 → Remove addTask() entirely
         (minimum 1 major version cycle between deprecation and removal)
```

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "我们以后再移除" | 以后永远不会到来。设置截止日期或接受它会永久存在。 |
| "可能有人还在用它" | 检查实际使用情况，不要假设。分析日志或搜索代码库。 |
| "迁移太复杂了" | 如果迁移复杂，新的 API 设计可能需要改进。先修复 API。 |
| "弃用注释就够了" | 不够。消费者需要文档、时间表和迁移指导。 |

## 危险信号

- 没有截止日期或跟踪 issue 的弃用标注
- 没有迁移指南的弃用
- 弃用代码被积极更改（说明它不会被移除）
- 在同一个版本中弃用和移除
- 消费者未知的弃用（他们需要通知）
- 永久存在的功能标志（不是临时的）

## 验证

弃用之后：

- [ ] 弃用注释包含替代方案和截止日期
- [ ] 运行时警告引导用户到迁移指南
- [ ] 迁移指南是公开的并经过测试
- [ ] 已通知消费者（PR 描述、变更日志、直接沟通）
- [ ] 截止日期已设置在日历中或跟踪 issue 中

移除之后：

- [ ] 旧代码完全移除（没有注释掉的残余）
- [ ] 相关测试更新
- [ ] 文档更新
- [ ] 跟踪 issue 已关闭
