# AGENTS.md

本文件为 AI 编程智能体（Claude Code、Cursor、Copilot、Antigravity 等）在本代码库中工作时提供指导。

## 代码库概述

这是一个面向资深软件工程师的 Claude.ai 和 Claude Code 技能集合。技能以打包好的指令和脚本形式呈现，用于扩展 Claude 和你的编程智能体的能力。

## OpenCode 集成

OpenCode 使用由 `skill` 工具和本仓库 `/skills` 目录驱动的**技能驱动执行模型**。

### 核心规则

- 如果某个任务匹配某个技能，你必须调用它
- 技能位于 `skills/<skill-name>/SKILL.md`
- 如果适用某个技能，切勿直接实现
- 始终严格遵循技能指令（不得部分应用）

### 意图 → 技能映射

智能体应自动将用户意图映射到技能：

- 功能 / 新功能 → `spec-driven-development`，然后是 `incremental-implementation`、`test-driven-development`
- 规划 / 分解 → `planning-and-task-breakdown`
- Bug / 失败 / 意外行为 → `debugging-and-error-recovery`
- 代码审查 → `code-review-and-quality`
- 重构 / 简化 → `code-simplification`
- API 或接口设计 → `api-and-interface-design`
- UI 工作 → `frontend-ui-engineering`

### 生命周期映射（隐式命令）

OpenCode 不支持 `/spec` 或 `/plan` 等斜杠命令。

因此，智能体必须在内部遵循以下生命周期：

- DEFINE → `spec-driven-development`
- PLAN → `planning-and-task-breakdown`
- BUILD → `incremental-implementation` + `test-driven-development`
- VERIFY → `debugging-and-error-recovery`
- REVIEW → `code-review-and-quality`
- SHIP → `shipping-and-launch`

### 执行模型

对于每个请求：

1. 判断是否有适用的技能（哪怕只有 1% 的可能性）
2. 使用 `skill` 工具调用相应技能
3. 严格遵循技能工作流
4. 仅在完成必要步骤（spec、plan 等）后才进行实现

### 反合理化

以下想法是错误的，必须忽略：

- "这个太小了，不需要用技能"
- "我可以直接快速实现这个"
- "我先收集上下文"

正确行为：

- 始终先检查并使用技能

这确保 OpenCode 的行为与 Claude Code 一致，完整执行工作流。

## 编排：角色、技能与命令

本仓库有三个可组合的层级，它们各司其职，不应混淆：

- **Skills**（`skills/<name>/SKILL.md`）——包含步骤和退出标准的工作流。解决*如何做*的问题。当意图匹配时强制触发。
- **Personas**（`agents/<role>.md`）——具有特定视角和输出格式的角色。解决*谁来做*的问题。
- **Slash commands**（`.claude/commands/*.md`）——面向用户的入口点。解决*何时做*的问题。编排层。

组合规则：**用户（或斜杠命令）是编排者。角色不调用其他角色。** 角色可以调用技能。

本仓库唯一认可的多角色编排模式是**并行扇出加合并步骤**——由 `/ship` 用于并发运行 `code-reviewer`、`security-auditor` 和 `test-engineer` 并综合其报告。不要构建一个决定调用哪个其他角色的"路由器"角色；那是斜杠命令和意图映射的职责。

决策矩阵见 [agents/README.md](agents/README.md)，完整模式目录见 [references/orchestration-patterns.md](references/orchestration-patterns.md)。

**Claude Code 互操作性：** `agents/` 中的角色既可作为 Claude Code 子智能体（从该插件的 `agents/` 目录自动发现），也可作为 Agent Teams 队友（通过名称引用来生成）。两个平台约束与我们的规则一致：子智能体不能生成其他子智能体，团队也不能嵌套。插件智能体会静默忽略 frontmatter 中的 `hooks`、`mcpServers` 和 `permissionMode` 字段。

## 创建新技能

### 目录结构

```
skills/
  {skill-name}/           # kebab-case 目录名
    SKILL.md              # 必需：技能定义
    scripts/              # 必需：可执行脚本
      {script-name}.sh    # Bash 脚本（首选）
  {skill-name}.zip        # 必需：打包用于分发
```

### 命名规范

- **技能目录**：`kebab-case`（例如 `web-quality`）
- **SKILL.md**：始终大写，始终使用此确切文件名
- **脚本**：`kebab-case.sh`（例如 `deploy.sh`、`fetch-logs.sh`）
- **Zip 文件**：必须与目录名完全一致：`{skill-name}.zip`

### SKILL.md 格式

```markdown
---
name: {skill-name}
description: {一句话描述何时使用此技能。包含触发短语，如"Deploy my app"、"Check logs"等。}
---

# {技能标题}

{技能功能的简要描述。}

## How It Works

{解释技能工作流的编号列表}

## Usage

```bash
bash /mnt/skills/user/{skill-name}/scripts/{script}.sh [args]
```

**Arguments:**
- `arg1` - 描述（默认为 X）

**Examples:**
{展示 2-3 种常见用法}

## Output

{展示用户将看到的示例输出}

## Present Results to User

{Claude 向用户呈现结果时的格式模板}

## Troubleshooting

{常见问题及解决方案，尤其是网络/权限错误}
```

### 上下文效率最佳实践

技能按需加载——启动时只加载技能名称和描述。完整的 `SKILL.md` 仅在智能体判断该技能相关时才加载到上下文中。为最小化上下文使用：

- **保持 SKILL.md 在 500 行以内**——将详细参考材料放在独立文件中
- **编写具体的描述**——帮助智能体准确知道何时激活该技能
- **使用渐进式披露**——引用仅在需要时才被读取的辅助文件
- **优先使用脚本而非内联代码**——脚本执行不消耗上下文（只有输出才会）
- **文件引用仅深入一级**——直接从 SKILL.md 链接到辅助文件

### 脚本要求

- 使用 `#!/bin/bash` shebang
- 使用 `set -e` 实现快速失败
- 将状态消息写入 stderr：`echo "Message" >&2`
- 将机器可读输出（JSON）写入 stdout
- 为临时文件包含清理 trap
- 将脚本路径引用为 `/mnt/skills/user/{skill-name}/scripts/{script}.sh`

### 创建 Zip 包

创建或更新技能后：

```bash
cd skills
zip -r {skill-name}.zip {skill-name}/
```

### 终端用户安装

为用户说明以下两种安装方式：

**Claude Code：**
```bash
cp -r skills/{skill-name} ~/.claude/skills/
```

**claude.ai：**
将技能添加到项目知识库，或将 SKILL.md 内容粘贴到对话中。

如果技能需要网络访问，请指导用户在 `claude.ai/settings/capabilities` 中添加所需域名。
