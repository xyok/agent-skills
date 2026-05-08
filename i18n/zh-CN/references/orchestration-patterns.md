# 编排模式

本仓库认可的 agent 编排模式参考目录，以及应当避免的反模式。在新增协调多个 persona 的 slash 命令，或引入"包装"现有 persona 的新 persona 之前，请阅读本文件。

核心规则：**用户（或 slash 命令）是编排者。Persona 不调用其他 persona。** 技能是 persona 工作流中的强制性步骤。

---

## 认可的模式

### 1. 直接调用（无编排）

单一 persona，单一视角，单一制品。这是默认且成本最低的选项。

```
user → code-reviewer → report → user
```

**适用场景：** 工作是对单一制品的单一视角分析，且能用一句话描述。

**示例：**
- "Review this PR" → `code-reviewer`
- "Find security issues in `auth.ts`" → `security-auditor`
- "What tests are missing for the checkout flow?" → `test-engineer`

**成本：** 一次往返。这是所有编排模式应与之比较的基准。

---

### 2. 单 persona slash 命令

使用项目技能包装单一 persona 的 slash 命令。避免用户每次都重新解释工作流程。

```
/review → code-reviewer (with code-review-and-quality skill) → report
```

**适用场景：** 相同的单 persona 调用重复发生且每次设置相同。

**本仓库示例：** `/review`、`/test`、`/code-simplify`。

**成本：** 与直接调用相同。slash 命令只是一个已保存的提示词。

**反信号：** 如果 slash 命令的主体大多是"决定调用哪个 persona"，请删除它，让用户直接调用 persona。

---

### 3. 并行扇出并合并

多个 persona 同时对同一输入进行操作，各自生成独立报告。合并步骤（在主 agent 的上下文中）将这些报告综合为单一决策。

```
                    ┌─→ code-reviewer    ─┐
/ship → fan out  ───┼─→ security-auditor ─┤→ merge → go/no-go + rollback
                    └─→ test-engineer    ─┘
```

**适用场景：**
- 子任务真正相互独立（无共享可变状态，无顺序依赖）
- 每个子 agent 受益于独立的上下文窗口
- 合并步骤足够小，可以在主上下文中完成
- 实际等待时间至关重要

**本仓库示例：** `/ship`。

**成本：** N 个并行子 agent 上下文 + 一次合并。比直接调用成本更高，但墙钟时间更快，且由于每个子 agent 专注于单一视角，报告质量更好。

**采用前的验证清单：**
- [ ] 我能同时运行所有子 agent 而没有顺序问题吗？
- [ ] 每个 persona 产生的是不同**类型**的发现，而非从不同角度看同一发现？
- [ ] 合并步骤能放入主 agent 剩余的上下文中吗？
- [ ] 用户的等待时间足够长，以至于并行化确实有明显效果？

如果任何答案为"否"，回退到直接调用或单 persona 命令。

---

### 4. 用户驱动的顺序 slash 命令流水线

用户按照固定顺序运行 slash 命令，在命令之间传递上下文（或提交历史）。没有编排 agent——用户本身就是编排者。

```
user runs:  /spec  →  /plan  →  /build  →  /test  →  /review  →  /ship
```

**适用场景：** 工作流存在依赖（每个步骤需要上一步的输出），且各步骤之间的人工判断有价值。

**本仓库示例：** 整个 DEFINE → PLAN → BUILD → VERIFY → REVIEW → SHIP 生命周期。

**成本：** 每步一个子 agent 上下文。编排层零成本，因为没有编排 agent。

**为何不自动化：** LLM"生命周期编排器"会（a）因为需要在交接时总结而丢失各步骤间的细节，（b）跳过能早期发现方向错误的人工检查点，（c）因转述回合而使 token 成本翻倍。

---

### 5. 研究隔离（上下文保护）

当任务需要阅读大量不应污染主上下文的材料时，生成一个研究子 agent，让其仅返回摘要。

```
main agent → research sub-agent (reads 50 files) → digest → main agent continues
```

**适用场景：**
- 主会话需要专注于下游任务
- 调查结果远小于其消耗的输入
- 主 agent 在完成调查后有足够的上下文空间来思考

**示例：** "在 monorepo 中找出所有调用这个已废弃 API 的地方"，"总结这 30 份 ADR 中关于缓存的内容"。

**成本：** 一个隔离的子 agent 上下文。当替代方案是将数百个文件加载到主上下文时，这是值得的。

**在 Claude Code 上，使用内置的 `Explore` 子 agent**，而非定义自定义研究 persona。`Explore` 运行在 Haiku 上，禁用了写入/编辑工具，专为此模式设计。仅当 `Explore` 不适用时（例如你需要模型无法推断的特定领域系统提示）才定义自定义研究子 agent。

---

## Claude Code 兼容性

本目录与运行环境无关，但大多数读者将在 Claude Code 上运行。以下是每种模式如何映射到 Claude Code 原语——以及平台在哪些地方为我们执行了规则。

### Persona 存储位置

插件子 agent 放在插件根目录的 `agents/` 中。本仓库是一个插件（`.claude-plugin/plugin.json`），因此 `agents/code-reviewer.md`、`agents/security-auditor.md` 和 `agents/test-engineer.md` 在启用插件时会被自动发现。无需路径配置。

### 子 agent 与 Agent Teams

Claude Code 有两种并行原语。模式 3（并行扇出并合并）映射到**子 agent**。如果你需要队友之间互相通信，请使用 **Agent Teams**。

| | 子 agent | Agent Teams |
|--|----------|-------------|
| 协调方式 | 主 agent 扇出，子 agent 仅汇报结果 | 队友互相发消息，共享任务列表 |
| 上下文 | 每个子 agent 独立的上下文窗口 | 每个队友独立的上下文窗口 |
| 适用场景 | 产出报告的独立任务 | 需要讨论的协作工作 |
| 状态 | 稳定 | 实验性——需要 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` |
| 成本 | 较低 | 较高——每个队友是独立的 Claude 实例 |

**本仓库中的 persona 在两种模式下均可工作。** 当作为子 agent 生成（例如被 `/ship` 调用）时，它们向主会话汇报发现。当作为队友生成（`Spawn a teammate using the security-auditor agent type…`）时，它们可以直接相互质疑对方的发现。Persona 定义相同；只有生成上下文不同。

一个细节：persona 中的 `skills` 和 `mcpServers` frontmatter 字段在作为子 agent 运行时被遵守，但**作为队友运行时被忽略**——队友从你的项目和用户设置加载技能和 MCP 服务器，与常规会话相同。如果一个 persona 依赖特定技能或 MCP 服务器，请在会话级别配置它，以确保在两种模式下都可用。

### 平台强制执行的规则

本目录中有两条规则不仅仅是约定——Claude Code 会强制执行：

- **"子 agent 不能生成其他子 agent"**（原文来自文档）。反模式 B（persona 调用 persona）和反模式 D（深层 persona 树）在 Claude Code 上从构造上就无法存在。
- **"无嵌套团队"**——队友不能生成自己的团队。同样的反模式在团队层面也被阻止。

这意味着你可以采用本目录中的模式，而无需担心贡献者意外构建反模式。这些反模式根本无法加载。

### 需要了解的内置子 agent

在定义自定义子 agent 之前，先检查以下内置子 agent 是否能满足需求：

| 内置子 agent | 用途 |
|--------------|------|
| `Explore` | 只读代码库搜索和分析。用于模式 5（研究隔离）。 |
| `Plan` | 计划模式下的只读研究。 |
| `general-purpose` | 需要探索和修改的多步骤任务。 |

不要重新定义这些内置子 agent。在它们之上叠加你的专家 persona（code-reviewer、security-auditor、test-engineer）。

### 插件 agent 的 frontmatter 限制

插件子 agent **不支持** `hooks`、`mcpServers` 或 `permissionMode` frontmatter 字段——这些会被静默忽略。如果未来的 persona 需要这些字段，用户必须将文件复制到 `.claude/agents/` 或 `~/.claude/agents/` 中。

在插件 agent 中**有效**的字段为：`name`、`description`、`tools`、`disallowedTools`、`model`、`maxTurns`、`skills`、`memory`、`background`、`effort`、`isolation`、`color`、`initialPrompt`。如需优化成本，可为每个 persona 指定 `model`（例如，`test-engineer` 覆盖率扫描用 Haiku，`code-reviewer` 用 Sonnet，`security-auditor` 用 Opus）。

### 并行生成多个子 agent

在 Claude Code 中，并行扇出（模式 3）需要在**单次助手回合中发出多个 Agent 工具调用**。顺序回合会串行化执行。`/ship` 对此有明确说明。任何新的编排器命令都应做到同样的事情。

---

## 示例：Agent Teams 用于竞争假设调试

此示例展示何时应使用 **Agent Teams** 而非 `/ship` 的子 agent 扇出。从远处看这两种模式很相似——都生成相同的三个 persona——但价值来自不同的地方。

### 场景

> *结账流程偶尔会挂起约 30 秒才完成。大约每 50 个会话发生一次。日志中无报错。在上周发布后开始出现。*

可能的根本原因（互相排斥，且都符合症状）：

1. 新支付确认流程中的竞态条件
2. 偶尔退回到慢速同步网络调用的认证检查
3. 随购物车大小扩展的查询缺少索引
4. 第三方 API 不稳定，SDK 在超时前静默重试

单个 agent 会选择第一个合理的理论就停止调查。`/ship` 风格的子 agent 扇出会让每个 persona 独立汇报——但它们的报告从不相遇，因此没有什么能排除错误的理论。

这正是 Agent Teams 文档描述的情况：*"当多个独立调查者积极尝试推翻彼此的理论时，最终存活下来的理论更有可能是真正的根本原因。"*

### 为何这**不是** `/ship` 的工作

| | `/ship`（子 agent） | Agent Teams |
|--|---------------------|-------------|
| 子 agent 看到的 | 相同的 diff，不同的视角 | 共享的任务列表，彼此的消息 |
| 输出 | 三份独立报告 → 一次合并 | 对抗性辩论 → 共识根本原因 |
| 适用场景 | 你想对已知制品作出判断 | 你想在假设中**找到**制品 |

`/ship` 是裁决；Agent Teams 是调查。

### 一次性设置（按环境）

Agent Teams 是实验性功能。在 `~/.claude/settings.json` 中：

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

需要 Claude Code v2.1.32 或更高版本。本仓库中的 persona 会自动被识别——无需手动编写团队配置文件。

### 触发提示词

在 lead 会话中用自然语言输入：

```
Users report checkout hangs for ~30 seconds intermittently after last
week's release. No errors in logs.

Create an agent team to debug this with competing hypotheses. Spawn
three teammates using the existing agent types:

  - code-reviewer  — investigate race conditions and blocking calls
                     in the checkout code path
  - security-auditor — investigate auth checks, session handling,
                       and any synchronous network calls added recently
  - test-engineer  — propose tests that would distinguish between the
                     hypotheses and check coverage gaps in checkout

Have them message each other directly to challenge each other's
theories. Update findings as consensus emerges. Only converge when
two teammates agree they can disprove the others'.
```

lead 生成三个队友，引用现有的 persona 名称。Persona 的内容会**追加**到每个队友的系统提示中作为附加指令（叠加在 lead 安装的团队协调指令之上）；上述触发提示词成为他们的任务。

### 运行过程

1. 每个队友在各自的上下文窗口中运行，从自身视角探索代码库。
2. 队友使用 `message` 直接向彼此发送发现。lead 无需中继。
3. 共享任务列表显示谁在调查什么——随时可通过 `Ctrl+T`（进程内模式）或 tmux 窗格（分屏模式）查看。
4. 当 `code-reviewer` 发现一个本应顺序执行的 `Promise.all` 时，会向 `security-auditor` 发消息确认认证调用是否是竞争的一部分。`security-auditor` 检查并回复——要么确认竞态是真正的问题，要么提供反证。
5. `test-engineer` 为当前最有可能的理论提出专注的集成测试，团队用它来验证后再宣告达成共识。
6. lead 综合已收敛的发现并呈现给你。

你可以通过 `Shift+Down` 循环切换到任意队友并输入——这对于将走错路的调查者重新引导很有用。

### 清理时机

当调查确定根本原因后，告诉 lead：

```
Clean up the team
```

始终通过 lead 进行清理，而非直接通过队友（根据文档：队友缺乏用于清理的完整团队上下文）。

### 成本预期

三个 Sonnet 队友运行约 10–15 分钟的调查，成本明显高于 `/ship` 将同三个 persona 作为子 agent 生成的成本。其合理性在于**结论的质量**——对于错误修复代价高昂的生产调试，额外的 token 是划算的。对于常规 PR 审查，请继续使用 `/ship`。

### 此场景中的反模式

**不要**将此场景重建为一个扇出子 agent 的 `/debug` slash 命令。子 agent 无法互相发消息——你会失去使该模式奏效的对抗性辩论。如果某个工作流反复出现，请将上述触发提示词记录为代码片段，而非将其包装在误用子 agent 的 slash 命令中。

### 何时**不**使用 Agent Teams

- 对已知 diff 进行生产就绪裁决 → 使用 `/ship`（子 agent）。
- 对单一制品进行单一专家视角分析 → 直接调用 persona。
- 顺序生命周期（spec → plan → build）→ 用户驱动的 slash 命令（模式 4）。
- 输出小摘要的大量阅读研究 → 内置 `Explore` 子 agent。

只有当队友**确实需要**相互质疑才能得出正确答案时，才使用 Agent Teams。

---

## 反模式

### A. 路由 persona（"meta-orchestrator"）

一个 persona，其职责是决定调用哪个其他 persona。

```
/work → router-persona → "this needs a review" → code-reviewer → router (paraphrases) → user
```

**为何失败：**
- 纯粹的路由层，没有任何领域价值
- 增加了两次转述跳转 → 信息损失 + 大约 2 倍 token 成本
- 用户本来就知道自己要做审查；他们本可以直接调用 `/review`
- 重复了 slash 命令和 `AGENTS.md` 中意图映射已经完成的工作

**替代方案：** 添加或完善 slash 命令。在 `AGENTS.md` 中记录意图 → 命令的映射关系。

---

### B. 调用其他 persona 的 persona

一个 `code-reviewer`，当它看到认证代码时会内部调用 `security-auditor`。

**为何失败：**
- Persona 的设计目的是产出单一视角；链式调用破坏了这一点
- 调用方 persona 传递的摘要会丢失被调用方 persona 所需的上下文
- 失败模式倍增（哪个 persona 的输出格式优先？谁的规则适用？）
- 向用户隐藏成本

**替代方案：** 让调用方 persona 在其报告中*建议*进行后续审计。由用户或 slash 命令发起第二次审查。

---

### C. 转述内容的顺序编排器

一个代表用户依次调用 `/spec`、`/plan`、`/build` 等的 agent。

**为何失败：**
- 丢失了能早期发现方向错误的人工检查点
- 每次交接都要总结上下文——长流水线中累积漂移
- token 成本翻倍：每步都有编排器回合 + 子 agent 回合
- 在最需要判断力的节点上移除了用户的能动性

**替代方案：** 保持用户作为编排者。在 `README.md` 中记录推荐的顺序，让用户自己调用。

---

### D. 深层 persona 树

`/ship` 调用一个 `pre-ship-coordinator`，后者调用一个 `quality-coordinator`，再调用 `code-reviewer`。

**为何失败：**
- 每层都增加延迟和 token，却没有任何决策价值
- 调试变成多层次的调查
- 叶子 persona 因多次总结步骤而失去上下文

**替代方案：** 保持编排深度最多为 1（slash 命令 → persona）。合并步骤在主 agent 中完成。

---

## 决策流程

在考虑新的编排工作流时，按此流程操作：

```
工作是否是对单一制品的单一视角分析？
├── 是 → 直接调用。停止。
└── 否  → 相同的组合是否会重复出现？
         ├── 否  → 直接调用，临时处理。停止。
         └── 是  → 子任务是否相互独立？
                  ├── 否  → 用户运行顺序 slash 命令（模式 4）。
                  └── 是  → 并行扇出并合并（模式 3）。
                           对照上述清单进行验证。
                           如果任何检查失败 → 回退到单 persona 命令（模式 2）。
```

---

## 何时向本目录添加新模式

仅在以下条件全部满足后才添加新条目：

1. 你已在实际工作中至少使用该模式两次
2. 你能在本仓库中指出一个演示该模式的具体制品
3. 你能解释为何现有模式无法替代
4. 你能描述其反模式影子（人们会错误构建什么来代替）

过早的目录条目会变成无人遵循的愿景式文档。
