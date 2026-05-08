# agent-skills 入门指南

agent-skills 适用于任何接受 Markdown 指令的 AI 编程助手。本指南介绍通用使用方式。如需特定工具的配置说明，请参阅对应的专项指南。

## Skills 的工作原理

每个 skill 都是一个 Markdown 文件（`SKILL.md`），描述特定的工程工作流。将其加载到 Agent 的上下文后，Agent 会遵循该工作流——包括验证步骤、应避免的反模式以及退出条件。

**Skills 不是参考文档。** 它们是 Agent 需要执行的分步流程。

## 快速开始（适用于任意 Agent）

### 1. 克隆仓库

```bash
git clone https://github.com/addyosmani/agent-skills.git
```

### 2. 选择一个 skill

浏览 `skills/` 目录。每个子目录包含一个 `SKILL.md`，内容涵盖：
- **使用时机** — 触发该 skill 的条件
- **流程** — 分步工作流
- **验证** — 如何确认工作已完成
- **常见借口** — Agent 可能用来跳过步骤的理由
- **危险信号** — skill 被违反的迹象

### 3. 将 skill 加载到你的 Agent

将相关 `SKILL.md` 的内容复制到 Agent 的系统提示、规则文件或对话中。常见的方式有以下几种：

**系统提示：** 在会话开始时粘贴 skill 内容。

**规则文件：** 将 skill 内容添加到项目的规则文件（CLAUDE.md、.cursorrules 等）中。

**对话：** 在给出指令时引用 skill："按照 test-driven-development 流程来完成这个改动。"

### 4. 使用元技能进行发现

先加载 `using-agent-skills` skill。它包含一个流程图，将任务类型映射到对应的 skill。

## 推荐配置

### 最简配置（从这里开始）

将以下三个核心 skill 加载到规则文件中：

1. **spec-driven-development** — 用于定义要构建的内容
2. **test-driven-development** — 用于证明功能可以正常运行
3. **code-review-and-quality** — 用于在合并前验证质量

这三个 skill 覆盖了 AI 辅助开发中最关键的质量盲区。

### 完整生命周期

如需全面覆盖，可按阶段加载 skill：

```
项目启动：  spec-driven-development → planning-and-task-breakdown
开发过程：  incremental-implementation + test-driven-development
合并前：    code-review-and-quality + security-and-hardening
部署前：    shipping-and-launch
```

### 按需加载

不要一次性加载所有 skill——这会浪费上下文。只加载与当前任务相关的 skill：

- 处理 UI？加载 `frontend-ui-engineering`
- 调试问题？加载 `debugging-and-error-recovery`
- 配置 CI？加载 `ci-cd-and-automation`

## Skill 结构

每个 skill 遵循相同的结构：

```
YAML frontmatter（name、description）
├── Overview — 该 skill 的功能说明
├── When to Use — 触发条件
├── Core Process — 分步工作流
├── Examples — 代码示例和模式
├── Common Rationalizations — 借口及反驳
├── Red Flags — skill 被违反的迹象
└── Verification — 退出条件清单
```

完整规范请参见 [skill-anatomy.md](skill-anatomy.md)。

## 使用 Agents

`agents/` 目录包含预配置的 Agent 角色：

| Agent | 用途 |
|-------|------|
| `code-reviewer.md` | 五维度代码审查 |
| `test-engineer.md` | 测试策略与编写 |
| `security-auditor.md` | 漏洞检测 |

当需要专项审查时，加载对应的 Agent 定义。例如，让编程助手"使用 code-reviewer Agent 角色来审查这个改动"，并提供 Agent 定义文件。

## 使用命令

`.claude/commands/` 目录包含适用于 Claude Code 的斜杠命令：

| 命令 | 调用的 Skill |
|------|------------|
| `/spec` | spec-driven-development |
| `/plan` | planning-and-task-breakdown |
| `/build` | incremental-implementation + test-driven-development |
| `/test` | test-driven-development |
| `/review` | code-review-and-quality |
| `/ship` | shipping-and-launch |

## 使用参考资料

`references/` 目录包含补充性检查清单：

| 参考资料 | 配合使用的 Skill |
|---------|----------------|
| `testing-patterns.md` | test-driven-development |
| `performance-checklist.md` | performance-optimization |
| `security-checklist.md` | security-and-hardening |
| `accessibility-checklist.md` | frontend-ui-engineering |

当需要超出 skill 范围的详细模式时，加载对应的参考资料。

## Spec 与任务产物

`/spec` 和 `/plan` 命令会生成工作产物（`SPEC.md`、`tasks/plan.md`、`tasks/todo.md`）。在工作进行期间，请将它们视为**活文档**：

- 在开发过程中将其纳入版本控制，以便人与 Agent 共享同一信息来源。
- 当范围或决策发生变化时及时更新。
- 如果你的仓库不希望长期保留这些文件，可在合并前删除，或将该文件夹添加到 `.gitignore`——工作流不要求这些文件永久存在。

## 使用建议

1. **对于任何非简单工作，从 spec-driven-development 开始**
2. **编写代码时始终加载 test-driven-development**
3. **不要跳过验证步骤** — 这才是关键所在
4. **按需加载 skill** — 更多上下文并不总是更好
5. **使用 Agent 进行审查** — 不同视角能发现不同问题
