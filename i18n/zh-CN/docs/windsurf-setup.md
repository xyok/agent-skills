# 在 Windsurf 中使用 agent-skills

## 配置

### 项目规则

Windsurf 使用 `.windsurfrules` 存储项目专属的 Agent 指令：

```bash
# 从最重要的 skill 创建合并的规则文件
cat /path/to/agent-skills/skills/test-driven-development/SKILL.md > .windsurfrules
echo "\n---\n" >> .windsurfrules
cat /path/to/agent-skills/skills/incremental-implementation/SKILL.md >> .windsurfrules
echo "\n---\n" >> .windsurfrules
cat /path/to/agent-skills/skills/code-review-and-quality/SKILL.md >> .windsurfrules
```

### 全局规则

对于希望在所有项目中使用的 skill，将其添加到 Windsurf 的全局规则：

1. 打开 Windsurf → Settings → AI → Global Rules
2. 粘贴最常用 skill 的内容

## 推荐配置

将 `.windsurfrules` 聚焦于 2-3 个核心 skill，以保持在上下文限制内：

```
# .windsurfrules
# Essential agent-skills for this project

[Paste test-driven-development SKILL.md]

---

[Paste incremental-implementation SKILL.md]

---

[Paste code-review-and-quality SKILL.md]
```

## 使用建议

1. **有选择性** — Windsurf 的上下文有限。选择能解决最大质量隐患的 skill。
2. **在对话中引用** — 处理特定阶段时，将额外的 skill 内容粘贴到聊天中（例如，构建身份验证时粘贴 `security-and-hardening`）。
3. **将参考资料用作检查清单** — 粘贴 `references/security-checklist.md` 并让 Windsurf 逐项验证。
