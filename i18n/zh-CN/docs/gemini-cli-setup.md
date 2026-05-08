# 在 Gemini CLI 中使用 agent-skills

## 配置

### 方式一：安装为 Skills（推荐）

Gemini CLI 拥有原生的 skills 系统，可自动发现 `.gemini/skills/` 或 `.agents/skills/` 目录中的 `SKILL.md` 文件。每个 skill 会在与任务匹配时按需激活。

**从仓库安装：**

```bash
gemini skills install https://github.com/addyosmani/agent-skills.git --path skills
```

**或从本地克隆安装：**

```bash
git clone https://github.com/addyosmani/agent-skills.git
gemini skills install /path/to/agent-skills/skills/
```

**仅为特定工作区安装：**

```bash
gemini skills install /path/to/agent-skills/skills/ --scope workspace
```

工作区范围安装的 skill 会放入 `.gemini/skills/`（或 `.agents/skills/`）。用户级 skill 放入 `~/.gemini/skills/`。

安装完成后，通过以下命令验证：

```
/skills list
```

Gemini CLI 会自动将 skill 的名称和描述注入提示中。当识别到匹配的任务时，会在加载完整指令前请求许可以激活该 skill。

### 方式二：GEMINI.md（持久化上下文）

对于希望始终以持久化项目上下文加载（而非按需激活）的 skill，可将其添加到项目的 `GEMINI.md`：

```bash
# 创建包含核心 skill 作为持久化上下文的 GEMINI.md
cat /path/to/agent-skills/skills/incremental-implementation/SKILL.md > GEMINI.md
echo -e "\n---\n" >> GEMINI.md
cat /path/to/agent-skills/skills/code-review-and-quality/SKILL.md >> GEMINI.md
```

也可以通过导入方式进行模块化：

```markdown
# Project Instructions

@skills/test-driven-development/SKILL.md
@skills/incremental-implementation/SKILL.md
```

使用 `/memory show` 验证已加载的上下文，使用 `/memory reload` 在更改后刷新。

> **Skills 与 GEMINI.md 的区别：** Skills 是按需激活的专项能力，仅在相关时才激活，从而保持上下文窗口整洁。GEMINI.md 提供每次提示都会加载的持久化上下文。将 skills 用于阶段性工作流，将 GEMINI.md 用于始终在线的项目规范。

## 推荐配置

### 始终在线（GEMINI.md）

将以下内容作为每次会话的持久化上下文：

- `incremental-implementation` — 以小型可验证切片方式构建
- `code-review-and-quality` — 五维度代码审查

### 按需激活（Skills）

将以下内容安装为 skill，使其仅在相关时才激活：

- `test-driven-development` — 在实现逻辑或修复 bug 时激活
- `spec-driven-development` — 在启动新项目或新功能时激活
- `frontend-ui-engineering` — 在构建 UI 时激活
- `security-and-hardening` — 在安全审查期间激活
- `performance-optimization` — 在性能工作期间激活

## 高级配置

### MCP 集成

该 skill 包中的许多 skill 利用 [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) 工具与环境交互。例如：

- `browser-testing-with-devtools` 使用 `chrome-devtools` MCP 扩展。
- `performance-optimization` 可受益于与性能相关的 MCP 工具。

要启用这些功能，请确保在 Gemini CLI 配置（`~/.gemini/config.json`）中安装了相关的 MCP 扩展。

### 会话钩子

Gemini CLI 支持会话生命周期钩子。你可以使用这些钩子在会话开始时自动注入上下文或运行验证脚本。

要复制其他工具中的 `agent-skills` 体验，可以配置一个 `SessionStart` 钩子，提示可用的 skill 或加载元技能。

### 显式上下文加载

你可以在提示中使用 `@` 符号引用任意 skill，将其显式加载到当前会话：

```markdown
Use the @skills/test-driven-development/SKILL.md skill to implement this fix.
```

当你希望确保遵循特定工作流而不等待自动发现时，这非常有用。

## 斜杠命令

该仓库在 `.gemini/commands/` 下提供了 7 个斜杠命令，映射到开发生命周期。从项目根目录运行时，Gemini CLI 会自动发现它们。

| 命令 | 功能 |
|------|------|
| `/spec` | 在编写代码前编写结构化规范 |
| `/planning` | 将工作拆分为小型可验证任务 |
| `/build` | 以增量方式实现下一个任务 |
| `/test` | 运行 TDD 工作流——红、绿、重构 |
| `/review` | 五维度代码审查 |
| `/code-simplify` | 在不改变行为的情况下降低复杂度 |
| `/ship` | 通过并行角色扇出进行上线前检查 |

每个命令都会自动调用对应的 skill——无需手动加载 skill。

> **注意：** 请使用 `/planning` 而非 `/plan`——`/plan` 与 Gemini CLI 内部命令名称冲突。

## 使用建议

1. **优先使用 skills 而非 GEMINI.md** — Skills 按需激活，保持上下文窗口专注。只有在希望始终加载时，才将 skill 放入 GEMINI.md。
2. **Skill 描述至关重要** — 每个 SKILL.md 的 frontmatter 中都有一个 `description` 字段，告诉 Agent 何时激活它。本仓库中的描述经过优化，可在所有受支持的工具（Claude Code、Gemini CLI 等）中实现自动发现，清晰说明了 skill 的*功能*和*触发时机*。
3. **使用 Agent 进行审查** — 请求结构化代码审查时，复制 `agents/code-reviewer.md` 的内容。
4. **结合参考资料使用** — 在处理测试或性能等特定质量领域时，引用 `references/` 中的检查清单。
