---
name: incremental-implementation
description: 以小的、可验证的步骤实现功能。适用于构建任何功能时使用，尤其是对代码库不熟悉、任务范围不清晰或风险高的情况下。
---

# 增量实现

## 概述

将功能分解成最小的可验证步骤，然后一个一个地构建它们。每个步骤都会使代码库处于工作状态，测试通过，代码已提交。这与"大爆炸"开发（构建所有内容然后一起调试）相反，后者会在集成时产生难以解开的问题。

**核心原则：** 在每一步之后，代码库必须处于可工作状态。不允许"一半实现"的代码。

## 适用场景

- 构建任何重要的新功能
- 重构现有代码
- 处理不熟悉的代码库
- 任务范围不清晰或有高风险
- 当中断是可能的（你需要能够放下工作并在之后恢复）

**不适用场景：** 一行 bug 修复或微不足道的文本变更——那些直接提交。

## 增量循环

```
For each increment:

1. IDENTIFY the smallest next step
   └── "What's the least I can build to make forward progress?"

2. IMPLEMENT it
   └── Write only the code for this increment

3. TEST it
   └── Run tests, verify behavior manually if needed

4. COMMIT it
   └── Atomic commit, all tests passing

5. ASSESS
   └── Does the full feature need revision based on what you learned?

6. REPEAT
```

**停止标准：** 继续下一步之前，当前步骤的测试必须通过。如果无法使当前步骤工作，在继续之前调查原因。

## 切片策略

功能可以用多种方式分解：

### 数据层优先

```
1. Database schema + migration
2. Data access layer (repository/service)
3. Business logic
4. API endpoint
5. UI component
6. Integration tests
```

**何时使用：** 数据模型复杂或不确定时。早期进行数据建模会暴露会影响所有上层的假设。

### UI 优先（原型优先）

```
1. Static UI with hardcoded data
2. Add state management
3. Wire up to API calls
4. Implement API endpoint
5. Connect to database
```

**何时使用：** 用户体验不确定，或需要早期利益相关者反馈时。构建真实的 UI，而不是线框图。

### 功能切片（端到端）

```
1. Simplest case, end-to-end (one user, no edge cases)
2. Second case (handles another variant)
3. Error handling
4. Edge cases
5. Polish and optimization
```

**何时使用：** 大多数功能的默认策略。尽早交付可工作的端到端流程。

## 五条实现规则

### 规则 1：在测试之前构建正确的抽象

不要在验证你在正确的层级进行操作之前构建框架。特别是：
- 不要在数据库模式有效之前构建服务层
- 不要在服务层工作之前构建 API
- 不要在 API 正确之前构建 UI

```
✗ Build everything at once and debug the integrated mess
✓ Get each layer working in isolation, then connect them
```

### 规则 2：先让它工作，再让它优雅

初始实现可以是直接的。重构在之后进行，当你有测试来保护时：

```
Step 1: Make it work (even if the code is ugly)
Step 2: Make it correct (edge cases, errors)
Step 3: Make it clean (refactor with test coverage)
Step 4: Make it fast (optimize only if there's evidence it's needed)
```

### 规则 3：不要让功能标志半打开

当一个功能在进行中时，它要么完全在标志后面（用户看不到），要么完全发布。"半实现的 UI"会迷惑用户并使调试变得复杂。

```typescript
// Feature is behind a flag until complete
if (featureFlags.isEnabled('new-task-filter')) {
  return <TaskFilterV2 />;
}
return <TaskFilterV1 />;
```

### 规则 4：测试自己的假设

在构建某些东西之前，验证你的理解是正确的：

```typescript
// Assumption: task.updatedAt is always set
// Test it before building logic that depends on it
console.log(await db.tasks.findFirst()); // Check the actual data shape
```

### 规则 5：立即检测集成问题

越早集成，越容易发现接口不匹配。不要等到最后才把所有东西连接起来。

## 示例：添加任务标签功能

完整功能：用户可以在任务上添加标签。

**不好的方式：** 一次性构建所有内容（模式、服务、API、UI）

**好的方式：**

```
Increment 1: Database schema
  - Add Label model to schema
  - Create and run migration
  - Verify: `npx prisma studio` shows the table
  - Commit: "Add label schema and migration"

Increment 2: Data access
  - Add createLabel(), getLabelsByTask() functions
  - Write unit tests for these functions
  - Verify: tests pass
  - Commit: "Add label data access functions"

Increment 3: API endpoints
  - POST /api/tasks/:id/labels
  - GET /api/tasks/:id/labels
  - Write integration tests
  - Verify: curl requests return expected responses
  - Commit: "Add label API endpoints"

Increment 4: UI - add label form
  - Build AddLabelForm component (no styling yet)
  - Verify: form submits and calls the API
  - Commit: "Add label form component"

Increment 5: UI - display labels
  - Build TaskLabel component
  - Show labels on TaskItem
  - Verify: labels appear after refresh
  - Commit: "Display labels on task items"

Increment 6: Polish
  - Add loading states
  - Add error handling
  - Add optimistic updates
  - Style according to design system
  - Verify: full user flow works
  - Commit: "Polish label UI interactions"
```

**结果：** 六个清晰的、可工作的检查点，而不是一个大集成噩梦。

## 认识卡顿

有些情况表明需要停下来重新评估：

```
Signs you're stuck in a dead end:
├── You've been debugging the same thing for > 30 minutes
├── Each fix creates two new problems
├── You've lost track of what state the code is in
└── You're not sure what "working" would even look like

What to do:
1. Stop the current approach
2. Revert to the last known-good state (git checkout)
3. Re-examine your assumptions
4. Try a different slicing strategy
5. Break the increment into even smaller steps
```

`git reset --hard HEAD`（或 `git stash`）是你的朋友。明确的起点比模糊的进展要好。

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "我有整个特性在脑子里，我可以一次全部构建" | 集成 bug 总是在你把所有东西连接起来时出现。逐步构建可以更早发现它们。 |
| "更大的提交比 50 个微小提交更清晰" | 提交不需要微小——它们需要是原子的（一件事）。5 个良好的增量提交是完美的。 |
| "增量实现太慢了" | 一次性构建调试一个大型集成需要更长时间。增量开发更快，即使感觉更慢。 |
| "我无法增量部署它，所以我也无法增量构建" | 部署和构建是独立的。用功能标志增量构建，当准备好时部署一次。 |

## 危险信号

- 超过 24 小时不提交
- 测试失败但仍继续构建
- 一个提交混合了多个不相关的更改
- 在测试前一个东西之前构建下一个层（未经测试的堆叠）
- 构建所有东西然后在最后才进行集成
- "一半实现的"UI 状态到达代码审查

## 验证

实现功能之后：

- [ ] 每个提交都是原子的（一件逻辑变更）
- [ ] 每个提交的所有测试都通过
- [ ] 没有 TODO 或半实现的代码路径（或者它们在功能标志后面）
- [ ] 每个层（数据、服务、API、UI）在单独连接前都经过单独测试
- [ ] 完整的端到端流程已验证
- [ ] 代码已审查，有清晰的分层实现历史
