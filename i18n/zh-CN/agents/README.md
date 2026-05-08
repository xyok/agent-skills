# Agent Personas（智能体角色）

专家角色，每个角色承担单一职责，提供单一视角。每个 persona 是一个 Markdown 文件，由你的运行环境（Claude Code、Cursor、Copilot 等）作为系统提示使用。

| Persona | 角色 | 最适用于 |
|---------|------|----------|
| [code-reviewer](code-reviewer.md) | 资深 Staff 工程师 | 合并前的五维度审查 |
| [security-auditor](security-auditor.md) | 安全工程师 | 漏洞检测、OWASP 风格审计 |
| [test-engineer](test-engineer.md) | QA 工程师 | 测试策略、覆盖率分析、"证明它"模式 |

## Persona、技能与命令的关系

三个层次，各有其职责：

| 层次 | 定义 | 示例 | 组合角色 |
|------|------|------|----------|
| **技能（Skill）** | 包含步骤和退出标准的工作流 | `code-review-and-quality` | *如何做*——在 persona 或命令内部调用 |
| **角色（Persona）** | 具有某种视角和输出格式的角色 | `code-reviewer` | *谁来做*——采用某种观点，产出报告 |
| **命令（Command）** | 用户面向的入口点 | `/review`、`/ship` | *何时做*——组合 persona 和技能 |

用户（或 slash 命令）是编排者。**Persona 不调用其他 persona。** 技能是 persona 工作流中的强制性步骤。

## 何时使用各种方式

### 直接调用 persona
当你需要对当前变更进行单一视角的分析，且用户处于参与状态时选择此方式。

- "Review this PR" → 直接调用 `code-reviewer`
- "Are there security issues in `auth.ts`?" → 直接调用 `security-auditor`
- "What tests are missing for the checkout flow?" → 直接调用 `test-engineer`

### Slash 命令（背后是单一 persona）
当有一个你每次都需要重新解释的可重复工作流时选择此方式。

- `/review` → 使用项目审查技能包装 `code-reviewer`
- `/test` → 使用 TDD 技能包装 `test-engineer`

### Slash 命令（编排者——扇出）
仅当**独立**的调查可以并行运行并产生报告，由单一 agent 汇总时选择此方式。

- `/ship` → 并行扇出到 `code-reviewer` + `security-auditor` + `test-engineer`，然后将其报告综合为发布与否的决策

这是本仓库唯一认可的编排模式。完整的模式目录和反模式请参见 [references/orchestration-patterns.md](../references/orchestration-patterns.md)。

## 决策矩阵

```
工作是否是对单一制品的单一视角分析？
├── 是 → 直接调用 persona
└── 否  → 子任务是否相互独立（无共享可变状态，无顺序依赖）？
         ├── 是 → 带并行扇出的 slash 命令（如 /ship）
         └── 否  → 由用户顺序运行 slash 命令（/spec → /plan → /build → /test → /review）
```

## 有效编排示例

`/ship` 是本仓库中典型的扇出编排器：

```
/ship
  ├── (parallel) code-reviewer    → review report
  ├── (parallel) security-auditor → audit report
  └── (parallel) test-engineer    → coverage report
                  ↓
        merge phase (main agent)
                  ↓
        go/no-go decision + rollback plan
```

为什么这种方式有效：
- 每个子 agent 基于同一 diff，但产出**不同的视角**
- 它们之间没有依赖关系 → 真正的并行性，节省实际等待时间
- 每个都在独立的上下文窗口中运行 → 主会话保持清晰
- 合并步骤规模小且受益于完整上下文，因此保留在主 agent 中

## 无效编排示例（请勿构建）

一个 `meta-orchestrator` persona，其职责是"决定调用哪个其他 persona"：

```
/work-on-pr → meta-orchestrator
                  ↓ (decides "this needs a review")
              code-reviewer
                  ↓ (returns)
              meta-orchestrator (paraphrases result)
                  ↓
              user
```

为什么这种方式失败：
- 纯粹的路由层，没有任何领域价值
- 增加了两次转述跳转 → 信息损失 + 2 倍 token 消耗
- 用户本来就知道自己要做审查；让他们直接调用 `/review` 即可
- 重复了 slash 命令和 `AGENTS.md` 意图映射已经完成的工作

## Persona 规则

1. 一个 persona 承担单一角色，产出单一格式的报告。如果你发现自己在添加第二个角色，请创建第二个 persona。
2. **Persona 不调用其他 persona。** 组合是 slash 命令或用户的职责。在 Claude Code 上，这同时也是平台的硬性约束——*"subagents cannot spawn other subagents"*——因此规则会自动执行。
3. 一个 persona 可以调用技能（*如何做*）。
4. 每个 persona 文件以"组合使用"模块结尾，说明其适用位置。

## Claude Code 互操作性

本仓库中的 persona 设计为无需修改即可作为 Claude Code 子 agent 和 Agent Teams 队友使用：

- **作为子 agent：** 启用此插件时自动被发现（无需路径配置）。使用 Agent 工具，指定 `subagent_type: code-reviewer`（或 `security-auditor`、`test-engineer`）。`/ship` 是典型示例。
- **作为 Agent Teams 队友**（实验性，需要 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`）：在生成队友时引用相同的 persona 名称。Persona 的内容会**追加到**队友的系统提示中作为附加指令（而非替换），因此你的 persona 文本叠加在 lead 安装的团队协调指令（SendMessage、task-list 工具等）之上。

子 agent 只能向主 agent 汇报结果。Agent Teams 允许队友之间直接互发消息。当报告已足够时使用子 agent；当子 agent 需要相互质疑对方的发现时（例如竞争假设调试）使用 Agent Teams。完整映射关系请参见 [references/orchestration-patterns.md](../references/orchestration-patterns.md)。

插件 agent 不支持 `hooks`、`mcpServers` 或 `permissionMode` frontmatter 字段——这些字段会被静默忽略。在此处编写新 persona 时请避免依赖这些字段。

## 添加新 persona

1. 按照现有 persona 使用的 frontmatter 格式创建 `agents/<role>.md`。
2. 定义角色、范围、输出格式和规则。
3. 在底部添加**组合使用**模块（直接调用时机 / 通过以下方式调用 / 不要从其他 persona 中调用）。
4. 将该 persona 添加到本文件顶部的表格中。
5. 如果该 persona 启用了新的编排模式，请在 `references/orchestration-patterns.md` 中记录，而非在 persona 文件本身中发明该模式。
