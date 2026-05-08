# Agent Skills

**面向 AI 编程智能体的生产级工程技能集。**

技能（Skills）将资深工程师在软件开发中使用的工作流程、质量关卡和最佳实践编码固化，并以结构化方式打包，使 AI 智能体能够在开发的每个阶段始终如一地遵循这些规范。

```
  DEFINE          PLAN           BUILD          VERIFY         REVIEW          SHIP
 ┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐
 │ Idea │ ───▶ │ Spec │ ───▶ │ Code │ ───▶ │ Test │ ───▶ │  QA  │ ───▶ │  Go  │
 │Refine│      │  PRD │      │ Impl │      │Debug │      │ Gate │      │ Live │
 └──────┘      └──────┘      └──────┘      └──────┘      └──────┘      └──────┘
   /spec          /plan          /build        /test         /review       /ship
```

---

## 命令

7 个斜杠命令，对应开发生命周期的各个阶段。每个命令会自动激活对应的技能。

| 你正在做什么 | 命令 | 核心原则 |
|-------------|------|---------|
| 定义要构建的内容 | `/spec` | 先写规格，再写代码 |
| 规划如何构建 | `/plan` | 小而原子化的任务 |
| 增量构建 | `/build` | 每次一个切片 |
| 验证功能正确性 | `/test` | 测试即证明 |
| 合并前审查 | `/review` | 提升代码健康度 |
| 简化代码 | `/code-simplify` | 清晰胜于聪明 |
| 发布到生产环境 | `/ship` | 越快越安全 |

技能也会根据你的操作自动激活——设计 API 时触发 `api-and-interface-design`，构建 UI 时触发 `frontend-ui-engineering`，以此类推。

---

## 快速开始

<details>
<summary><b>Claude Code（推荐）</b></summary>

**通过 Marketplace 安装：**

```
/plugin marketplace add addyosmani/agent-skills
/plugin install agent-skills@addy-agent-skills
```

> **遇到 SSH 错误？** Marketplace 通过 SSH 克隆仓库。如果你没有在 GitHub 上配置 SSH 密钥，可以[添加 SSH 密钥](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)，或使用完整 HTTPS URL 强制通过 HTTPS 克隆：
> ```bash
> /plugin marketplace add https://github.com/addyosmani/agent-skills.git
> /plugin install agent-skills@addy-agent-skills
> ```

**本地 / 开发模式：**

```bash
git clone https://github.com/addyosmani/agent-skills.git
claude --plugin-dir /path/to/agent-skills
```

</details>

<details>
<summary><b>Cursor</b></summary>

将任意 `SKILL.md` 复制到 `.cursor/rules/`，或引用完整的 `skills/` 目录。详见 [docs/cursor-setup.md](docs/cursor-setup.md)。

</details>

<details>
<summary><b>Gemini CLI</b></summary>

以原生技能方式安装以支持自动发现，或添加到 `GEMINI.md` 以获得持久上下文。详见 [docs/gemini-cli-setup.md](docs/gemini-cli-setup.md)。

**从仓库安装：**

```bash
gemini skills install https://github.com/addyosmani/agent-skills.git --path skills
```

**从本地克隆安装：**

```bash
gemini skills install ./agent-skills/skills/
```

</details>

<details>
<summary><b>Windsurf</b></summary>

将技能内容添加到你的 Windsurf 规则配置中。详见 [docs/windsurf-setup.md](docs/windsurf-setup.md)。

</details>

<details>
<summary><b>OpenCode</b></summary>

通过 AGENTS.md 和 `skill` 工具使用智能体驱动的技能执行模型。

详见 [docs/opencode-setup.md](docs/opencode-setup.md)。

</details>

<details>
<summary><b>GitHub Copilot</b></summary>

将 `agents/` 中的智能体定义用作 Copilot 角色（personas），并将技能内容放入 `.github/copilot-instructions.md`。详见 [docs/copilot-setup.md](docs/copilot-setup.md)。

</details>

<details>
  <summary><b>Kiro IDE & CLI</b></summary>
  Kiro 的技能存放在 ".kiro/skills/" 目录下，可在项目级或全局级进行配置。Kiro 同样支持 Agents.md。详见 Kiro 文档：https://kiro.dev/docs/skills/
</details>

<details>
<summary><b>Codex / 其他智能体</b></summary>

技能均为纯 Markdown 格式——适用于任何接受系统提示词或指令文件的智能体。详见 [docs/getting-started.md](docs/getting-started.md)。

</details>

---

## 全部 20 个技能

上述命令是入口点。在底层，它们会激活以下 20 个技能——每个技能都是包含步骤、验证关卡和反合理化表格的结构化工作流。你也可以直接引用任意技能。

### Define - 明确要构建的内容

| 技能 | 功能说明 | 适用场景 |
|------|---------|---------|
| [idea-refine](skills/idea-refine/SKILL.md) | 结构化发散/收敛思维，将模糊想法转化为具体方案 | 你有一个需要深入探索的粗略概念 |
| [spec-driven-development](skills/spec-driven-development/SKILL.md) | 在编写任何代码之前，撰写涵盖目标、命令、结构、代码风格、测试和边界的 PRD | 启动新项目、新功能或重大变更时 |

### Plan - 分解任务

| 技能 | 功能说明 | 适用场景 |
|------|---------|---------|
| [planning-and-task-breakdown](skills/planning-and-task-breakdown/SKILL.md) | 将规格分解为带验收标准和依赖顺序的小型、可验证任务 | 你已有规格文档，需要拆分为可实施单元时 |

### Build - 编写代码

| 技能 | 功能说明 | 适用场景 |
|------|---------|---------|
| [incremental-implementation](skills/incremental-implementation/SKILL.md) | 细垂直切片——实现、测试、验证、提交。支持功能标志、安全默认值、易回滚变更 | 任何涉及多个文件的变更 |
| [test-driven-development](skills/test-driven-development/SKILL.md) | Red-Green-Refactor，测试金字塔（80/15/5），测试规模，DAMP 优先于 DRY，Beyonce Rule，浏览器测试 | 实现逻辑、修复 bug 或修改行为时 |
| [context-engineering](skills/context-engineering/SKILL.md) | 在正确的时机为智能体提供正确的信息——规则文件、上下文打包、MCP 集成 | 开启会话、切换任务或输出质量下降时 |
| [source-driven-development](skills/source-driven-development/SKILL.md) | 将每个框架决策都建立在官方文档的基础上——验证、引用来源、标注未经验证的内容 | 你希望获得有权威来源引用的框架或库代码时 |
| [frontend-ui-engineering](skills/frontend-ui-engineering/SKILL.md) | 组件架构、设计系统、状态管理、响应式设计、WCAG 2.1 AA 无障碍性 | 构建或修改面向用户的界面时 |
| [api-and-interface-design](skills/api-and-interface-design/SKILL.md) | 契约优先设计、Hyrum 定律、One-Version Rule、错误语义、边界验证 | 设计 API、模块边界或公共接口时 |

### Verify - 证明功能正确

| 技能 | 功能说明 | 适用场景 |
|------|---------|---------|
| [browser-testing-with-devtools](skills/browser-testing-with-devtools/SKILL.md) | Chrome DevTools MCP，获取实时运行时数据——DOM 检查、控制台日志、网络追踪、性能分析 | 构建或调试任何在浏览器中运行的内容时 |
| [debugging-and-error-recovery](skills/debugging-and-error-recovery/SKILL.md) | 五步分类法：复现、定位、缩小范围、修复、防护。停线规则、安全回退 | 测试失败、构建中断或出现意外行为时 |

### Review - 合并前的质量关卡

| 技能 | 功能说明 | 适用场景 |
|------|---------|---------|
| [code-review-and-quality](skills/code-review-and-quality/SKILL.md) | 五轴审查、变更规模（约 100 行）、严重级别标签（Nit/Optional/FYI）、审查速度规范、拆分策略 | 合并任何变更之前 |
| [code-simplification](skills/code-simplification/SKILL.md) | Chesterton's Fence、500 行规则，在保留完整行为的同时降低复杂度 | 代码可以运行，但比应有的更难阅读或维护时 |
| [security-and-hardening](skills/security-and-hardening/SKILL.md) | OWASP Top 10 防护、认证模式、密钥管理、依赖审计、三层边界系统 | 处理用户输入、认证、数据存储或外部集成时 |
| [performance-optimization](skills/performance-optimization/SKILL.md) | 测量优先方法——Core Web Vitals 目标值、性能分析工作流、包体积分析、反模式检测 | 存在性能要求或怀疑出现性能回退时 |

### Ship - 自信发布

| 技能 | 功能说明 | 适用场景 |
|------|---------|---------|
| [git-workflow-and-versioning](skills/git-workflow-and-versioning/SKILL.md) | 主干开发、原子提交、变更规模（约 100 行）、提交即存档点模式 | 进行任何代码变更时（始终适用） |
| [ci-cd-and-automation](skills/ci-cd-and-automation/SKILL.md) | Shift Left、越快越安全、功能标志、质量关卡流水线、失败反馈循环 | 搭建或修改构建和部署流水线时 |
| [deprecation-and-migration](skills/deprecation-and-migration/SKILL.md) | 代码即负债思维、强制性与建议性废弃、迁移模式、僵尸代码清除 | 移除旧系统、迁移用户或下线功能时 |
| [documentation-and-adrs](skills/documentation-and-adrs/SKILL.md) | 架构决策记录、API 文档、内联文档规范——记录*为什么* | 做出架构决策、修改 API 或发布功能时 |
| [shipping-and-launch](skills/shipping-and-launch/SKILL.md) | 上线前检查清单、功能标志生命周期、分阶段发布、回滚流程、监控配置 | 准备发布到生产环境时 |

---

## 智能体角色（Agent Personas）

预配置的专家角色，用于针对性审查：

| 智能体 | 角色 | 视角 |
|--------|------|------|
| [code-reviewer](agents/code-reviewer.md) | 资深员工工程师 | 五轴代码审查，以"Staff 工程师会批准吗？"为标准 |
| [test-engineer](agents/test-engineer.md) | QA 专家 | 测试策略、覆盖率分析和 Prove-It 模式 |
| [security-auditor](agents/security-auditor.md) | 安全工程师 | 漏洞检测、威胁建模、OWASP 评估 |

---

## 参考检查清单

技能在需要时会引用的快速参考材料：

| 参考文件 | 涵盖内容 |
|---------|---------|
| [testing-patterns.md](references/testing-patterns.md) | 测试结构、命名规范、Mock、React/API/E2E 示例、反模式 |
| [security-checklist.md](references/security-checklist.md) | 提交前检查、认证、输入验证、请求头、CORS、OWASP Top 10 |
| [performance-checklist.md](references/performance-checklist.md) | Core Web Vitals 目标值、前端/后端检查清单、测量命令 |
| [accessibility-checklist.md](references/accessibility-checklist.md) | 键盘导航、屏幕阅读器、视觉设计、ARIA、测试工具 |

---

## 技能的工作原理

每个技能遵循一致的结构：

```
┌─────────────────────────────────────────────────┐
│  SKILL.md                                       │
│                                                 │
│  ┌─ Frontmatter ─────────────────────────────┐  │
│  │ name: lowercase-hyphen-name               │  │
│  │ description: Guides agents through [task].│  │
│  │              Use when…                    │  │
│  └───────────────────────────────────────────┘  │                                                                                                
│  Overview         → What this skill does        │
│  When to Use      → Triggering conditions       │
│  Process          → Step-by-step workflow       │
│  Rationalizations → Excuses + rebuttals         │
│  Red Flags        → Signs something's wrong     │
│  Verification     → Evidence requirements       │
└─────────────────────────────────────────────────┘
```

**关键设计选择：**

- **流程，而非散文。** 技能是智能体遵循的工作流，而非阅读的参考文档。每个技能都有步骤、检查点和退出标准。
- **反合理化。** 每个技能都包含一张智能体用于跳过步骤的常见借口表（例如"我稍后再加测试"），并附有有据可查的反驳论据。
- **验证不可妥协。** 每个技能以证据要求作为结尾——测试通过、构建输出、运行时数据。"看起来没问题"永远不够。
- **渐进式披露。** `SKILL.md` 是入口点，支撑性参考文件仅在需要时加载，将 token 消耗降至最低。

---

## 项目结构

```
agent-skills/
├── skills/                            # 20 个核心技能（每个目录含 SKILL.md）
│   ├── idea-refine/                   #   Define
│   ├── spec-driven-development/       #   Define
│   ├── planning-and-task-breakdown/   #   Plan
│   ├── incremental-implementation/    #   Build
│   ├── context-engineering/           #   Build
│   ├── source-driven-development/     #   Build
│   ├── frontend-ui-engineering/       #   Build
│   ├── test-driven-development/       #   Build
│   ├── api-and-interface-design/      #   Build
│   ├── browser-testing-with-devtools/ #   Verify
│   ├── debugging-and-error-recovery/  #   Verify
│   ├── code-review-and-quality/       #   Review
│   ├── code-simplification/          #   Review
│   ├── security-and-hardening/        #   Review
│   ├── performance-optimization/      #   Review
│   ├── git-workflow-and-versioning/   #   Ship
│   ├── ci-cd-and-automation/          #   Ship
│   ├── deprecation-and-migration/     #   Ship
│   ├── documentation-and-adrs/        #   Ship
│   ├── shipping-and-launch/           #   Ship
│   └── using-agent-skills/            #   Meta: 如何使用本技能包
├── agents/                            # 3 个专家角色
├── references/                        # 4 个辅助检查清单
├── hooks/                             # 会话生命周期钩子
├── .claude/commands/                  # 7 个斜杠命令（Claude Code）
├── .gemini/commands/                  # 7 个斜杠命令（Gemini CLI）
└── docs/                              # 各工具的配置指南
```

---

## 为什么需要 Agent Skills？

AI 编程智能体默认选择最短路径——这往往意味着跳过规格说明、测试、安全审查，以及那些让软件可靠的工程实践。Agent Skills 为智能体提供结构化工作流，强制执行资深工程师在生产代码中所具备的同等纪律。

每个技能都编码了来之不易的工程判断力：*何时*撰写规格、*测试什么*、*如何*审查，以及*何时*发布。这些不是泛泛而谈的提示词——它们是区分生产质量与原型质量的那种有主见、流程驱动的工作流。

技能将 Google 工程文化的最佳实践内化其中——包括来自 [Software Engineering at Google](https://abseil.io/resources/swe-book) 和 Google [工程实践指南](https://google.github.io/eng-practices/) 的理念。你会在 API 设计中看到 Hyrum 定律，在测试中看到 Beyonce Rule 和测试金字塔，在代码审查中看到变更规模和审查速度规范，在简化中看到 Chesterton's Fence，在 git 工作流中看到主干开发，在 CI/CD 中看到 Shift Left 和功能标志，以及一个将代码视为负债的专用废弃技能。这些不是抽象原则——它们直接嵌入在智能体遵循的逐步工作流中。

---

## 贡献

技能应当具备**具体性**（可操作的步骤，而非模糊建议）、**可验证性**（带证据要求的清晰退出标准）、**经过实战检验**（基于真实工作流）和**最小化**（仅包含引导智能体所需的内容）。

格式规范见 [docs/skill-anatomy.md](docs/skill-anatomy.md)，贡献指南见 [CONTRIBUTING.md](CONTRIBUTING.md)。

---

## 许可证

MIT——在你的项目、团队和工具中自由使用这些技能。
