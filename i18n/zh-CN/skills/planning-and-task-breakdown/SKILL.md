---
name: planning-and-task-breakdown
description: 将功能分解为可依赖排序的、可验证的任务。适用于在开始实现之前规划工作时使用，尤其是对于跨越多个会话的、有外部依赖的，或需要与他人协调的功能。
---

# 规划与任务分解

## 概述

在实现之前将功能分解为清晰、可排序的任务，每个任务都有明确的完成标准。好的规划不是创建一份从不阅读的文档——而是揭示隐藏的依赖关系、识别风险，以及使跟踪进度成为可能。

**关键洞见：** 真正的挑战不是分解任务，而是以正确的顺序排序它们。错误的顺序会造成返工：你在有 schema 之前构建 UI，然后发现 schema 需要改变，现在 UI 也需要更改。

## 适用场景

- 跨越多个会话或天数的功能
- 需要多人协作的工作
- 有真实依赖（数据库、外部 API、其他功能）的功能
- 高风险变更（认证、支付、数据迁移）
- 范围不清晰或可能增长的功能

**不适用场景：** 可以在单个会话中完成的单文件变更——直接实现它。

## 分解过程

```
1. UNDERSTAND
   └── Read the spec or requirements
   └── Identify the core goal, not just the listed tasks

2. IDENTIFY COMPONENTS
   └── What data layer changes are needed?
   └── What API changes?
   └── What UI changes?
   └── What external dependencies?

3. BUILD DEPENDENCY GRAPH
   └── What must exist before each task can start?

4. SLICE TASKS
   └── Each task should be independently testable
   └── Each task should leave the system in a working state

5. ESTIMATE AND ORDER
   └── Assign rough sizes
   └── Order by dependencies + risk (risky/uncertain first)

6. WRITE ACCEPTANCE CRITERIA
   └── Each task needs a specific, testable "done" condition
```

## 依赖图

始终从依赖图开始。它揭示了工作的实际顺序：

```
Example: Add task labels feature

Database schema (Label model, TaskLabel join table)
    │
    ├── Label service (CRUD operations)
    │       │
    │       ├── Label API endpoints
    │       │       │
    │       │       └── Label UI: add/remove labels
    │       │               │
    │       │               └── Label filtering in task list
    │       │
    │       └── Label API tests
    │
    └── Task service (update to include labels)
            │
            └── Task API (include labels in response)
```

**规则：** 在图中存在依赖关系时，不要并行安排任务。

## 垂直切片（端到端）

在切分层级（仅数据库、仅 API）之前，考虑垂直切片：端到端的最小工作片段。

```
✗ Horizontal slices (risky — nothing works until all layers are done):
  Task 1: All database schema changes
  Task 2: All service layer changes
  Task 3: All API changes
  Task 4: All UI changes

✓ Vertical slices (each slice delivers working functionality):
  Task 1: Create a label (schema + service + API + minimal UI)
  Task 2: Delete a label
  Task 3: Assign a label to a task
  Task 4: Filter tasks by label
```

垂直切片使你能够在功能完成之前就增量交付价值，并在每个步骤之后获得反馈。

## 任务大小指南

| 大小 | 时间估算 | 特征 |
|------|---------|------|
| XS | < 30 分钟 | 单文件变更，功能是增量的 |
| S | 30 分钟 - 2 小时 | 少数文件，清晰的完成标准 |
| M | 2-4 小时 | 多个文件，少数依赖 |
| L | 4-8 小时（1 天）| 需要一些调查，有一定复杂性 |
| XL | > 1 天 | 需要分解为更小的任务 |

**规则：** XL 任务是一个信号，表明规划还没有深入到足够的层次。继续分解，直到没有任何任务大于 L。

## 验收标准

每个任务都需要具体、可测试的完成条件：

```
✗ Vague acceptance criteria:
  "Add label functionality to tasks"

✓ Specific, testable criteria:
  "Given a task exists, when I POST /api/tasks/:id/labels with
   { name: 'urgent', color: '#ff0000' }, then:
   - Response is 201 with the created label
   - GET /api/tasks/:id returns the label in task.labels[]
   - Duplicate label names per task return 409"
```

## 计划文档格式

```markdown
## Feature: Task Labels

### Goal
Allow users to organize tasks with color-coded labels.

### Dependencies
- External: None
- Internal: Task API must support labels in response

### Tasks

#### Phase 1: Data Layer
- [ ] **[S]** Add Label schema and migration
  - Acceptance: `npx prisma migrate dev` succeeds, Label table exists
  - Risk: Low

- [ ] **[S]** Add LabelService (create, delete, findByTask)
  - Acceptance: Unit tests pass for all three operations
  - Risk: Low

#### Phase 2: API
- [ ] **[S]** POST /api/tasks/:id/labels
  - Acceptance: Creates label, returns 201, validates name length < 50
  - Risk: Low

- [ ] **[M]** GET /api/tasks/:id includes labels
  - Acceptance: Task responses include labels[] array
  - Risk: Low (additive change)

#### Phase 3: UI
- [ ] **[M]** Add label UI to task detail view
  - Acceptance: Can add and remove labels, updates immediately
  - Risk: Medium (state management complexity)

- [ ] **[L]** Add label filter to task list
  - Acceptance: Selecting a label filters the visible task list
  - Risk: Medium (need to handle multiple label selection)

### Risks
- Label search at scale (if users create 100+ labels)
  → Mitigate: add server-side search in Phase 3 if needed
```

## 识别计划中的风险

对于每个任务，评估：

```
Technical risk: How well do we understand how to implement this?
  Low  → Standard CRUD, familiar patterns
  Med  → Some unknowns, may need investigation
  High → Significant unknowns, needs a spike first

Reversibility: If we build this wrong, how hard is it to change?
  Easy   → Code-only change, easy to refactor
  Medium → API change, needs versioning
  Hard   → Data migration, hard to reverse

Impact: What happens if this is wrong?
  Low  → Internal tool, limited users
  Med  → Some users affected
  High → Core functionality, all users affected
```

**规则：** 高风险任务排在前面。在你了解高风险部分之前构建低风险部分，如果调查产生惊喜，你就是在浪费时间。

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "我知道如何实现它，不需要规划" | 规划不是为了知道你在做什么——而是为了发现你尚未考虑的部分。 |
| "计划是浪费时间，我应该开始编码" | 无规划的实现经常产生需要重新工作的耦合。半小时的规划可以节省数天。 |
| "任务太小了，无需验收标准" | 清晰的验收标准使"完成"变得可测试，而不是主观的。 |
| "我可以并行做这些任务" | 检查依赖图。并行化依赖任务会创建集成瓶颈。 |

## 危险信号

- 在依赖层之前排序的任务
- 没有验收标准的任务
- 超过 1 天的任务（XL——需要进一步分解）
- 没有错误处理的计划
- 没有测试任务（测试应该在实现中，不是分开的任务）
- 忽略数据迁移的变更 schema

## 验证

完成规划之后：

- [ ] 依赖图已构建，任务已按依赖关系排序
- [ ] 没有 XL 任务（全部 L 或更小）
- [ ] 每个任务都有具体的、可测试的验收标准
- [ ] 高风险任务排在前面
- [ ] 已识别外部依赖，已知是否被阻塞
- [ ] 已知完成定义（功能什么时候算"完成"）
- [ ] 准备好开始 `incremental-implementation`
