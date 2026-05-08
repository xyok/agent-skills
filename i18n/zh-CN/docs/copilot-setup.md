# 在 GitHub Copilot 中使用 agent-skills

## 配置

### Copilot 指令

Copilot 支持在仓库的 `.github/skills`、`.claude/skills` 或 `.agents/skills` 目录中创建 Agent skill。

```bash
mkdir -p .github

# 为核心 skill 创建文件
cat /path/to/agent-skills/skills/test-driven-development/SKILL.md > .github/skills/test-driven-development/SKILL.md
cat /path/to/agent-skills/skills/code-review-and-quality/SKILL.md > .github/skills/code-review-and-quality/SKILL.md
```

更多详情，请参阅 [为 GitHub Copilot 创建 Agent Skills](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-skills)。

### Agent 角色（agents.md）

Copilot 支持专项 Agent 角色。使用 agent-skills 中的 Agent：

```bash
# 复制 Agent 定义
cp /path/to/agent-skills/agents/code-reviewer.md .github/agents/code-reviewer.md
cp /path/to/agent-skills/agents/test-engineer.md .github/agents/test-engineer.md
cp /path/to/agent-skills/agents/security-auditor.md .github/agents/security-auditor.md
```

在 Copilot Chat 中调用 Agent：
- `@code-reviewer Review this PR`
- `@test-engineer Analyze test coverage for this module`
- `@security-auditor Check this endpoint for vulnerabilities`

### 自定义指令（用户级别）

对于希望在所有仓库中使用的 skill：

1. 打开 VS Code → Settings → GitHub Copilot → Custom Instructions
2. 添加你最常用的 skill 摘要

## 推荐配置

### .github/copilot-instructions.md

GitHub Copilot 通过 `.github/copilot-instructions.md` 支持项目级指令。

```markdown
# Project Coding Standards

## Testing
- Write tests before code (TDD)
- For bugs: write a failing test first, then fix (Prove-It pattern)
- Test hierarchy: unit > integration > e2e (use the lowest level that captures the behavior)
- Run `npm test` after every change

## Code Quality
- Review across five axes: correctness, readability, architecture, security, performance
- Every PR must pass: lint, type check, tests, build
- No secrets in code or version control

## Implementation
- Build in small, verifiable increments
- Each increment: implement → test → verify → commit
- Never mix formatting changes with behavior changes

## Boundaries
- Always: Run tests before commits, validate user input
- Ask first: Database schema changes, new dependencies
- Never: Commit secrets, remove failing tests, skip verification
```

### 专项 Agent

在 Copilot Chat 中使用这些 Agent 进行有针对性的审查工作流。

## 使用建议

1. **保持指令简洁** — Copilot 指令在聚焦时效果最佳。总结关键规则，而非包含完整的 skill 文件。
2. **使用 Agent 进行审查** — code-reviewer、test-engineer 和 security-auditor Agent 专为 Copilot 的 Agent 模型设计。
3. **在 Chat 中引用** — 处理特定阶段时，将相关 skill 内容粘贴到 Copilot Chat 中提供上下文。
4. **结合 PR 审查使用** — 配置 Copilot 使用 code-reviewer Agent 角色审查 PR。
