# Skill 结构解析

本文档描述 agent-skills skill 文件的结构与格式。贡献新 skill 或理解现有 skill 时，可将本文档作为参考指南。

## 文件位置

每个 skill 都存放在 `skills/` 下各自的目录中：

```
skills/
  skill-name/
    SKILL.md           # 必填：Skill 定义
    supporting-file.md # 可选：按需加载的参考资料
```

## SKILL.md 格式

### Frontmatter（必填）

```yaml
---
name: skill-name-with-hyphens
description: Guides agents through [task/workflow]. Use when [specific trigger conditions].
---
```

**规则：**
- `name`：小写，用连字符分隔。必须与目录名称一致。
- `description`：以第三人称描述 skill 的功能，然后列出一个或多个明确的"Use when"触发条件。同时包含*功能*和*时机*。最多 1024 个字符。

**为何重要：** Agent 通过读取 description 来发现 skill。description 会被注入系统提示，因此必须同时说明 skill 的功能和激活时机。不要在 description 中概述工作流——如果 description 包含流程步骤，Agent 可能会遵循摘要而不去读取完整的 skill。

### 标准章节（推荐模式）

```markdown
# Skill 标题

## Overview
一到两句话，说明该 skill 的功能及其重要性。

## When to Use
- 触发条件的列表（症状、任务类型）
- 不适用的场景（排除项）

## [Core Process / The Workflow / Steps]
主要工作流，拆分为编号步骤或阶段。
在有帮助的地方附上代码示例。
在存在决策点的地方使用流程图（ASCII）。

## [Specific Techniques / Patterns]
针对特定场景的详细指导。
代码示例、模板、配置。

## Common Rationalizations
| 借口 | 现实 |
|---|---|
| Agent 用来跳过步骤的理由 | 为何该理由是错误的 |

## Red Flags
- 表明 skill 被违反的行为模式
- 审查期间需要关注的事项

## Verification
完成 skill 流程后，确认以下内容：
- [ ] 退出条件清单
- [ ] 证明要求
```

## 各章节的作用

### Overview
Skill 的"电梯演讲"。应回答：该 skill 做什么，Agent 为何要遵循它？

### When to Use
帮助 Agent 和用户判断当前任务是否适用该 skill。同时包含正向触发条件（"Use when X"）和排除条件（"NOT for Y"）。

### Core Process
Skill 的核心。这是 Agent 要遵循的分步工作流。必须具体且可操作——而非模糊的建议。

**好的写法：** "运行 `npm test` 并验证所有测试通过"
**不好的写法：** "确保测试可以正常工作"

### Common Rationalizations
精心设计的 skill 最具特色的部分。这些是 Agent 用来跳过重要步骤的借口，以及对应的反驳。它们可以防止 Agent 为不遵循流程进行自我辩解。

想想 Agent 每次说"我以后再加测试"或"这个足够简单，可以跳过 spec"的情景——这些都应该写在这里，并附上事实性的反驳。

### Red Flags
Skill 被违反时可观察到的迹象。在代码审查和自我监控中很有用。

### Verification
退出条件。Agent 用于确认 skill 流程已完成的清单。每个复选框都应该可以用证据验证（测试输出、构建结果、截图等）。

## 辅助文件

仅在以下情况下创建辅助文件：
- 参考资料超过 100 行（保持主 SKILL.md 的专注性）
- 需要代码工具或脚本
- 清单足够长，需要单独的文件

内容不超过 50 行时，将模式和原则内联保留。

## 写作原则

1. **流程优先于知识。** Skill 是工作流，而非参考文档。是步骤，而非事实。
2. **具体优先于笼统。** "运行 `npm test`" 胜过 "验证测试"。
3. **证据优先于假设。** 每个验证复选框都需要证明。
4. **反自我辩解。** 每个值得跳过的步骤，在借口表中都需要有一个反驳。
5. **渐进式披露。** 主 SKILL.md 是入口。辅助文件仅在需要时加载。
6. **Token 意识。** 每个章节必须证明其存在的必要性。如果删除它不会改变 Agent 的行为，就删掉它。

## 命名规范

- Skill 目录：`lowercase-hyphen-separated`（小写，连字符分隔）
- Skill 文件：`SKILL.md`（始终大写）
- 辅助文件：`lowercase-hyphen-separated.md`
- 参考资料：存放在项目根目录的 `references/` 中，而非 skill 目录内

## 跨 Skill 引用

通过名称引用其他 skill：

```markdown
Follow the `test-driven-development` skill for writing tests.
If the build breaks, use the `debugging-and-error-recovery` skill.
```

不要在 skill 之间重复内容——使用引用和链接代替。
