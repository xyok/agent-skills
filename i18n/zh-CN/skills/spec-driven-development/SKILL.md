---
name: spec-driven-development
description: 在实现之前编写结构化规格说明。适用于开始构建任何新功能、API 端点或系统组件时使用。在实现之前，先规格化、规划，然后再构建。
---

# 规格驱动开发

## 概述

在编写代码之前，先编写清晰的、经过验证的规格说明。规格说明是你和未来代码之间的契约——它定义了正确行为的含义，使测试成为可能，并防止范围蔓延。没有规格说明，你不知道何时完成。

## 适用场景

- 构建任何新功能或组件
- 设计新的 API 端点
- 重大重构，改变行为
- 需要协调多个开发者或跨越多个会话的工作

**不适用场景：** 微小的 bug 修复、单行更改，或在单个会话中可以完全完成的微小功能。

## 四阶段门控工作流

```
Phase 1: SPECIFY
  └── Write the spec (what, not how)
  └── Gate: spec is clear and approved before proceeding

Phase 2: PLAN
  └── Identify implementation tasks and dependencies
  └── Gate: tasks are ordered, sized, and unambiguous

Phase 3: TASKS
  └── Map tasks to increments with acceptance criteria
  └── Gate: each task has a testable done condition

Phase 4: IMPLEMENT
  └── Build incrementally (see incremental-implementation)
  └── Gate: all acceptance criteria met
```

每个阶段都是一个关卡——在前一阶段完成之前，不要进入下一阶段。

## 阶段 1：规格说明

规格说明描述**什么**，而不是**如何**。它定义了正确实现的含义。

### 规格说明模板

```markdown
## Feature: [Name]

### Goal
One sentence: what user problem does this solve?

### User Stories
As a [user type], I want to [do something] so that [benefit].

### Functional Requirements
1. [Specific behavior, testable]
2. [Specific behavior, testable]
3. [Specific behavior, testable]

### Non-Functional Requirements
- Performance: [specific target, e.g., "< 200ms response time"]
- Security: [specific requirement, e.g., "only accessible by owner"]
- Accessibility: [specific standard, e.g., "WCAG 2.1 AA"]

### Data Requirements
- Inputs: [what the feature receives]
- Outputs: [what it produces]
- Persistence: [what changes in the database]

### Edge Cases
- [What happens in edge case 1]
- [What happens in edge case 2]

### Out of Scope
- [Explicit exclusions to prevent scope creep]

### Acceptance Criteria
Given [precondition], when [action], then [expected result].
(Repeat for each testable behavior)
```

### 规格说明示例

```markdown
## Feature: Task Labels

### Goal
Allow users to organize tasks with color-coded labels for easier categorization.

### User Stories
As a project manager, I want to assign multiple labels to tasks
so that I can quickly filter and organize tasks by category or priority.

### Functional Requirements
1. Users can create labels with a name (max 50 chars) and a hex color
2. Up to 5 labels can be assigned per task
3. Labels are scoped to the workspace (shared across team members)
4. Users can filter the task list by one or more labels
5. Selecting multiple labels shows tasks with ANY of those labels (OR logic)

### Non-Functional Requirements
- Performance: label filtering must not increase task list load time > 50ms
- Security: users can only assign labels from their own workspace

### Data Requirements
- Inputs: label name, hex color code
- Outputs: labeled tasks in task list, label chips on task cards
- Persistence: new Label table, TaskLabel join table

### Edge Cases
- Deleting a label removes it from all tasks (cascade)
- Filtering by label that has 0 tasks shows empty state
- Searching + filtering labels work together (AND logic across filters)

### Out of Scope
- Label templates or presets
- Per-user private labels (workspace-only)
- Label ordering or priority

### Acceptance Criteria
Given a task exists, when I add the "urgent" label, then the label
appears on the task card and the task appears in the "urgent" label filter.

Given 5 labels on a task, when I try to add a 6th, then I see an error
"Maximum 5 labels per task".

Given labels A and B, when I filter by both, then I see tasks with A,
tasks with B, and tasks with both (OR logic).
```

## 阶段 2：规划

一旦规格说明清晰，识别实现任务：

```markdown
## Implementation Plan: Task Labels

### Dependencies
- Label schema must exist before LabelService
- LabelService must exist before label API
- Label API must exist before label UI

### Tasks (ordered by dependency)
1. [S] Add Label and TaskLabel schema + migration
2. [S] Implement LabelService (createLabel, deleteLabel, getLabelsForTask)
3. [S] Implement label API endpoints (CRUD)
4. [M] Add label UI to task detail view
5. [M] Add label filter to task list view
6. [S] Write integration tests for full flow
```

参见 `planning-and-task-breakdown` 了解详细的任务大小和依赖规划。

## 阶段 3：任务

将每个计划任务细化为带有验收标准的增量：

```markdown
## Increment 1: Label Schema

Implementation:
- Add Label model to Prisma schema
- Add TaskLabel join table
- Create and run migration

Acceptance Criteria:
- `npx prisma migrate dev` succeeds without errors
- Label and TaskLabel tables exist in the database
- Can manually insert a test label record

## Increment 2: LabelService

Implementation:
- createLabel(name, color, workspaceId) → Label
- deleteLabel(labelId, userId) → void (validates ownership)
- getLabelsForTask(taskId) → Label[]

Acceptance Criteria:
- Unit tests pass for all three functions
- deleteLabel returns error if user doesn't own the workspace
- createLabel validates name length ≤ 50 chars
```

## 阶段 4：实现

按照已规划的增量实现，每个增量后都提交：

参见 `incremental-implementation` 了解增量构建的详细指导。

**关键规则：** 在实现过程中如果规格说明改变，先更新规格说明，然后再更新代码。不要让代码和规格说明分歧。

## 规格说明如何防止常见问题

| 问题 | 规格说明如何解决 |
|------|----------------|
| 范围蔓延 | "不在范围内"部分明确排除功能 |
| 不知道何时完成 | 验收标准定义了"完成" |
| 处理边界情况的 bug | 边界情况在开始之前就被识别 |
| 不同理解导致的返工 | 规格说明是对齐的共同参考 |
| 测试覆盖率低 | 验收标准直接映射到测试用例 |

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "我已经了解了这个功能，不需要规格说明" | 了解一个功能≠能够测试它是否正确实现。写下验收标准。 |
| "规格说明是浪费时间" | 范围蔓延和返工浪费的时间是写规格说明的 10 倍。 |
| "我们可以在路上弄清楚细节" | 边界情况在实现过程中出现时会产生延迟。提前确定。 |
| "这只是一个小功能，不需要规格说明" | 小功能有"不在范围内"条目，这可以防止它们变成大功能。 |

## 危险信号

- 开始实现没有验收标准
- 代码完成之后编写规格说明（本末倒置）
- 没有"不在范围内"部分（范围蔓延的滋生地）
- 无法验证的验收标准（"功能运行良好"≠验收标准）
- 实现与规格说明分歧而没有更新规格说明

## 验证

开始实现之前：

- [ ] 规格说明包含目标、功能要求和验收标准
- [ ] "不在范围内"已定义
- [ ] 边界情况已识别并有预期行为
- [ ] 每个验收标准是具体且可测试的（Given/When/Then）
- [ ] 实现计划有依赖排序的任务
- [ ] 每个任务的大小已估算（都 ≤ L）
