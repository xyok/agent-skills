---
name: idea-refine
description: 将模糊的想法转化为清晰的、可操作的规格说明。适用于对某个功能、产品或技术方向有初步想法，但在深入实现之前需要进一步打磨时使用。
---

# 想法提炼

## 概述

将模糊的直觉或粗糙的概念转化为清晰的、有充分论据的提案，以便于做决策或转交给实现流程。这不是头脑风暴——而是结构化地将想法从"感觉不错"推进到"我们可以对此进行构建"。

## 适用场景

- 你有功能或产品方向的初步想法
- 一个问题已被识别，但解决方案还不清楚
- 有多个竞争性方案需要评估
- 想法需要向他人传达以获得认同
- 在实现之前需要验证假设

**不适用场景：** 当需求已经清晰，你需要的是 `spec-driven-development` 而不是提炼；当需要分解任务，你需要的是 `planning-and-task-breakdown`。

## 三阶段流程

```
Phase 1: DIVERGE
  └── Generate multiple interpretations and angles
      └── Goal: breadth, not depth

Phase 2: EVALUATE
  └── Test each interpretation against constraints and goals
      └── Goal: identify the strongest signal

Phase 3: SHARPEN
  └── Articulate the chosen direction precisely
      └── Goal: actionable clarity
```

## 第一阶段：发散

**目标：** 在聚焦之前先拓宽思维

从这些问题开始：

```
1. What problem are we really solving?
   └── "Build a task filter" → "Help users find tasks quickly in a long list"

2. Who has this problem?
   └── All users? Power users? New users? Specific roles?

3. What are the boundaries?
   └── What's in scope? What's explicitly out of scope?

4. What are the constraints?
   └── Technical limitations? Timeline? Dependencies?

5. What does "success" look like?
   └── How would we know this worked?
```

**产出：** 2-4 种不同的思考方式或解决方案方向，不带判断地列出。

## 第二阶段：评估

**目标：** 对每个方向进行压力测试

对每个候选方向提问：

```
Feasibility: Can we actually build this?
  └── Technical complexity, team capacity, dependencies

Impact: Does it solve the core problem?
  └── For whom? By how much? Are there better solutions?

Risk: What could go wrong?
  └── Technical risks, user adoption, edge cases, reversibility

Cost: What does it require?
  └── Engineering time, infrastructure, maintenance burden

Tradeoffs: What do we give up?
  └── Simplicity? Flexibility? Performance? Other features?
```

**产出：** 每个方向的优劣势，以及关于最有前景方向的建议。

## 第三阶段：锐化

**目标：** 将所选方向提炼为清晰、可操作的表述

### 提炼模板

```markdown
## Refined Idea: [Name]

### Problem
[One paragraph: what's broken, who it affects, and what the cost of not solving it is]

### Proposed Solution
[One paragraph: what we'll build and how it solves the problem]

### Who It's For
[The specific user or user segment who benefits most]

### Success Criteria
[How we'll know it worked — specific, measurable]

### Key Constraints
[Technical, timeline, or resource constraints that shape the solution]

### Out of Scope
[What this explicitly does NOT include]

### Open Questions
[What still needs to be answered before or during implementation]

### Risks
[What could go wrong, and mitigation ideas]
```

### 示例

```markdown
## Refined Idea: Quick Task Filter

### Problem
Users with 50+ tasks struggle to find specific tasks. The current list
requires scrolling through everything. Support tickets show "find my task"
as a top complaint for users with large task loads.

### Proposed Solution
Add a real-time filter bar above the task list. As users type, tasks that
don't match the query are hidden. Matches are highlighted. Works on title
and description.

### Who It's For
Power users managing 20+ tasks simultaneously (roughly 15% of active users).

### Success Criteria
- Users find a specific task in < 5 seconds when using the filter
- Engagement with the filter on lists > 20 tasks > 60% within 30 days

### Key Constraints
- Must not slow down initial render (client-side filter, not an API call)
- Mobile must work with soft keyboard visible (filter bar stays accessible)

### Out of Scope
- Label filtering (separate feature)
- Saved searches / filter presets
- Cross-list searching (only filters the current list)

### Open Questions
- Should the filter persist on navigation (URL param) or reset on leave?
- What's the minimum list size to show the filter bar?

### Risks
- Users may expect search to work across all lists (scope creep pressure)
  → Mitigate: clear placeholder text "Filter this list..."
```

## 互动对话模式

当想法仍然模糊时，通过提问来探索：

**揭示真正问题的问题：**
- "告诉我一个用户遇到这个问题时的实际场景"
- "如果你不实现这个功能，最坏的情况是什么？"
- "你看到这需要被构建的信号是什么？"

**揭示约束的问题：**
- "是什么让这在今天难以实现？"
- "谁需要先完成什么才能实现这个？"
- "如果我们下周就要交付，我们会砍掉什么？"

**阐明成功的问题：**
- "六个月后，你怎么知道这是否成功了？"
- "哪一类用户最会受益？他们会如何描述改进？"
- "我们怎么衡量这个？"

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "想法很清楚，我们只需要构建它" | 开始构建之前，你意外发现了需求——之后再考虑会更昂贵。 |
| "我们现在不需要提炼，我们只需要行动" | 没有清晰度的行动建造了错误的东西。提炼需要几个小时，错误的实现需要几周。 |
| "只有一种方法可以做到这件事" | 当你真正探索时，几乎总有多个方向。探索它们。 |
| "我们以后可以弄清细节" | 范围蔓延和重新工作来自于之后才弄清楚细节。 |

## 危险信号

- 开始实现没有清晰问题陈述的功能
- 成功标准无法测量
- 范围没有"不包含"部分
- 未解答的关键假设（"我们假设用户想要……"）
- 解决方案提案出现在对问题有充分理解之前

## 验证

想法提炼之后：

- [ ] 可以用一句话描述问题
- [ ] 可以用一句话描述解决方案
- [ ] 成功标准是可测量的
- [ ] 不在范围内的内容已明确定义
- [ ] 已识别关键约束
- [ ] 主要风险已被考虑
- [ ] 准备好转入 `spec-driven-development`（如果继续进行实现）
