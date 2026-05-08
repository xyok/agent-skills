# OpenCode 配置指南

本指南介绍如何在 OpenCode 中使用 Agent Skills，以高度贴近 Claude Code 体验的方式运行（自动 skill 选择、生命周期驱动的工作流及严格的流程执行）。

## 概述

OpenCode 支持自定义 `/commands`，但没有像 Claude Code 那样的原生插件系统或自动 skill 路由。

我们通过以下方式实现功能对等：

- 强大的系统提示（`AGENTS.md`）
- 内置 `skill` 工具
- 从 `/skills` 目录持续进行 skill 发现

这创建了一个**由 Agent 驱动的工作流**，其中 skill 会被自动选择和执行。

虽然可以在 OpenCode 中重建 `/spec`、`/plan` 等命令，但此集成方案有意采用 Agent 驱动的方式：

- Skill 根据意图自动选择
- 工作流通过 `AGENTS.md` 执行
- 无需手动调用命令

这更接近 Claude Code 在实践中的行为方式——skill 自动触发，而非手动触发。

---

## 安装

1. 克隆仓库：

```bash
git clone https://github.com/addyosmani/agent-skills.git
```

2. 在 OpenCode 中打开该项目。

3. 确保工作区中存在以下文件：

- `AGENTS.md`（根目录）
- `skills/` 目录

无需额外安装。

---

## 工作原理

### 1. Skill 发现

所有 skill 存放在：

```
skills/<skill-name>/SKILL.md
```

OpenCode Agent 通过 `AGENTS.md` 接收指令，要求：

- 检测 skill 是否适用
- 调用 `skill` 工具
- 严格遵循 skill

### 2. 自动 Skill 调用

Agent 会评估每个请求并映射到对应的 skill。

示例：

- "构建一个功能" → `incremental-implementation` + `test-driven-development`
- "设计一个系统" → `spec-driven-development`
- "修复一个 bug" → `debugging-and-error-recovery`
- "审查这段代码" → `code-review-and-quality`

用户**无需**显式请求 skill。

### 3. 生命周期映射（隐式命令）

开发生命周期以隐式方式编码：

- DEFINE → `spec-driven-development`
- PLAN → `planning-and-task-breakdown`
- BUILD → `incremental-implementation` + `test-driven-development`
- VERIFY → `debugging-and-error-recovery`
- REVIEW → `code-review-and-quality`
- SHIP → `shipping-and-launch`

这取代了 `/spec`、`/plan` 等斜杠命令。

---

## 使用示例

### 示例一：功能开发

用户：
```
Add authentication to this app
```

Agent 行为：
- 检测到功能开发工作
- 调用 `spec-driven-development`
- 在编写代码前生成规范
- 进入规划和实现 skill

---

### 示例二：Bug 修复

用户：
```
This endpoint is returning 500 errors
```

Agent 行为：
- 调用 `debugging-and-error-recovery`
- 复现 → 定位 → 修复 → 添加防护措施

---

### 示例三：代码审查

用户：
```
Review this PR
```

Agent 行为：
- 调用 `code-review-and-quality`
- 进行结构化审查（正确性、设计、可读性等）

---

## Agent 要求（关键）

为使 OpenCode 正确运行，Agent 必须遵循以下规则：

- 在行动前始终检查 skill 是否适用
- 如果 skill 适用，**必须**使用它
- 不得跳过必要的工作流（spec、plan、test 等）
- 不得直接跳到实现阶段

这些规则通过 `AGENTS.md` 执行。

---

## 局限性

- 没有原生斜杠命令（通过意图映射处理）
- 没有插件系统（通过提示和结构处理）
- Skill 调用依赖模型的合规性

尽管如此，该工作流在实践中与 Claude Code 高度吻合。

---

## 推荐工作流

只需使用自然语言：

- "设计一个功能"
- "规划这个改动"
- "实现这个"
- "修复这个 bug"
- "审查这个"

Agent 将自动选择并执行正确的 skill。

---

## 总结

OpenCode 集成通过以下组合实现：

- 结构化 skill（本仓库）
- 强大的 Agent 规则（`AGENTS.md`）
- 通过推理自动调用 skill

这形成了一个**完全由 Agent 驱动的生产级工程工作流**，无需插件或手动命令。
