---
name: using-agent-skills
description: 理解并有效使用 agent skills 工作流。当你需要了解如何使用这些 skills、何时应用哪个 skill，或者为什么 skills 以这种方式结构化时使用。
---

# 使用 Agent Skills

## 概述

这些 skills 是高级软件工程师的工作流——经过深思熟虑、实战检验的流程，可防止常见的工程失败模式。每个 skill 代表了开发者多年来积累的关于如何正确完成工作的集体智慧。

**核心前提：** Agent 不会通过省略步骤来走捷径。如果某个 skill 适用，就要完全遵循它。

## Skill 发现图

```
User intent → Skill to use

New feature / new functionality
  └── spec-driven-development (define first)
      └── planning-and-task-breakdown (plan second)
          └── incremental-implementation (build third)
              └── test-driven-development (test throughout)

Bug / unexpected behavior / failing test
  └── debugging-and-error-recovery

Code review / PR review
  └── code-review-and-quality

Refactoring / simplification
  └── code-simplification

API design / interface design / module boundaries
  └── api-and-interface-design

UI work / components / accessibility
  └── frontend-ui-engineering

Security review / auth implementation
  └── security-and-hardening

Performance problems / optimization
  └── performance-optimization

CI/CD setup / automation / deployment
  └── ci-cd-and-automation

Git operations / versioning / releases
  └── git-workflow-and-versioning

Documentation / ADRs
  └── documentation-and-adrs

Deprecation / migration
  └── deprecation-and-migration

Vague idea / fuzzy requirement
  └── idea-refine (clarify first)

Using unfamiliar library / API
  └── source-driven-development

Setting up Claude Code / AI agent context
  └── context-engineering

Browser testing / debugging UI
  └── browser-testing-with-devtools

Shipping / launch / deployment
  └── shipping-and-launch
```

## 六个核心操作行为

### 行为 1：先应用 Skill，再实现

当用户请求属于某个 skill 范围内的任务时，在编写任何代码之前先遵循该 skill 的流程。

```
✗ Wrong: "Here's the implementation of the new API endpoint..."
         (jumping straight to code without spec or plan)

✓ Right: "This is a new feature. Following spec-driven-development:
          [Writes spec first]
          [Plans tasks]
          [Then implements incrementally]"
```

### 行为 2：完全执行 Skill，不要部分执行

Skill 是门控的——每个阶段在下一阶段开始之前必须完成。不能选择性地执行步骤。

```
✗ Wrong: Jumping from spec directly to code, skipping the planning phase

✓ Right: Spec → Plan → Tasks → Implement (all four phases, in order)
```

### 行为 3：识别何时应该链接多个 Skills

复杂任务可能需要多个 skills 按顺序执行：

```
New feature with unknown requirements:
  1. idea-refine (clarify the idea)
  2. spec-driven-development (write the spec)
  3. planning-and-task-breakdown (plan the work)
  4. incremental-implementation (build it)
  5. test-driven-development (throughout)
  6. code-review-and-quality (review before merging)

Bug fix in security-sensitive code:
  1. debugging-and-error-recovery (diagnose the bug)
  2. security-and-hardening (apply during the fix)
  3. test-driven-development (prove-it pattern)
```

### 行为 4：抵制"这太小了"的合理化

Skills 尤其对感觉太小而不需要它们的任务有价值：
- 小型特性有范围蔓延的边界情况
- 小型 API 变更有 Hyrum 定律影响
- 小型 bug 修复会导致回归

```
✗ Self-deception: "This is just a 2-line change, I don't need a spec"
✓ Reality check: "Even 2-line changes can have edge cases. What are the 
                  acceptance criteria?"
```

### 行为 5：先验证假设

在实现之前，验证你对代码库、框架或需求的理解是否正确。

```
Before implementing: "I'll fetch the Prisma v6 docs to verify the 
                      relation query syntax (source-driven-development)"

Before refactoring: "I'll run the existing tests to establish a baseline
                     (test-driven-development)"
```

### 行为 6：当感到卡住时，重新评估 Skill 适用性

如果在实现过程中遇到阻碍，这可能是一个信号，表明另一个 skill 需要先被应用：

```
Blocked because requirements are unclear?
  → Go back to spec-driven-development

Blocked because of a mysterious bug?
  → Pause and apply debugging-and-error-recovery

Blocked because code is too complex to modify?
  → Apply code-simplification before proceeding
```

## 生命周期序列

对于完整的功能开发，标准生命周期是：

```
1. DEFINE    → spec-driven-development
2. PLAN      → planning-and-task-breakdown
3. BUILD     → incremental-implementation + test-driven-development
4. VERIFY    → debugging-and-error-recovery (if issues arise)
5. REVIEW    → code-review-and-quality
6. SHIP      → shipping-and-launch
```

支持性 skills 在需要时随时应用：
- `source-driven-development`——当需要库文档时
- `security-and-hardening`——当构建认证/授权时
- `performance-optimization`——当遇到性能问题时
- `browser-testing-with-devtools`——当测试 UI 时

## 失败模式

这些是 skills 旨在防止的常见失败模式：

| 失败模式 | 对应的 Skill |
|---------|------------|
| 构建了用户没有要求的功能 | spec-driven-development |
| 集成时发现层级之间不匹配 | incremental-implementation |
| 发布时才发现 bug | test-driven-development |
| 调试花费了几个小时而不是几分钟 | debugging-and-error-recovery |
| 重构破坏了现有行为 | test-driven-development |
| 库 API 与假设不符 | source-driven-development |
| 安全漏洞在生产中被发现 | security-and-hardening |
| 代码无法理解，难以修改 | code-simplification |
| 部署导致生产中断 | shipping-and-launch |
| PR 太大，无法有效审查 | planning-and-task-breakdown |

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "这太小了，不需要 skill" | 最昂贵的 bug 往往来自被认为太小而不需要流程的变更。 |
| "我可以跳过规划直接构建" | 跳过规划会产生你在集成时才发现的耦合。 |
| "我了解这个框架，不需要文档" | 框架会随时间变化。验证，而不是假设。 |
| "测试会减慢速度" | 没有测试的调试比有测试的实现慢 10 倍。 |
| "我们稍后再清理代码" | 技术债复利。在每个 PR 中应用 code-simplification。 |

## 验证

使用 skills 时：

- [ ] 已识别适用于任务的 skill
- [ ] 完全遵循了 skill，没有省略步骤
- [ ] 当需要时链接了多个 skills
- [ ] 没有用"这太小了"来合理化省略步骤
- [ ] 当遇到阻碍时，重新评估了是否需要另一个 skill
