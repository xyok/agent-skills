# 在 Cursor 中使用 agent-skills

## 配置

### 方式一：规则目录（推荐）

Cursor 支持通过 `.cursor/rules/` 目录设置项目专属规则：

```bash
# 创建规则目录
mkdir -p .cursor/rules

# 将所需的 skill 复制为规则文件
cp /path/to/agent-skills/skills/test-driven-development/SKILL.md .cursor/rules/test-driven-development.md
cp /path/to/agent-skills/skills/code-review-and-quality/SKILL.md .cursor/rules/code-review-and-quality.md
cp /path/to/agent-skills/skills/incremental-implementation/SKILL.md .cursor/rules/incremental-implementation.md
```

该目录中的规则会自动加载到 Cursor 的上下文中。

### 方式二：.cursorrules 文件

在项目根目录创建 `.cursorrules` 文件，并内联核心 skill 内容：

```bash
# 生成合并的规则文件
cat /path/to/agent-skills/skills/test-driven-development/SKILL.md > .cursorrules
echo "\n---\n" >> .cursorrules
cat /path/to/agent-skills/skills/code-review-and-quality/SKILL.md >> .cursorrules
```

## 推荐配置

### 核心 Skills（始终加载）

将以下内容添加到 `.cursor/rules/`：

1. `test-driven-development.md` — TDD 工作流和 Prove-It 模式
2. `code-review-and-quality.md` — 五维度代码审查
3. `incremental-implementation.md` — 以小型可验证切片方式构建

### 阶段专用 Skills（按需加载）

对于特定阶段的工作，可根据需要创建额外的规则文件：

- `spec-development.md` -> `spec-driven-development/SKILL.md`
- `frontend-ui.md` -> `frontend-ui-engineering/SKILL.md`
- `security.md` -> `security-and-hardening/SKILL.md`
- `performance.md` -> `performance-optimization/SKILL.md`

在处理相关任务时将其添加到 `.cursor/rules/`，完成后移除以管理上下文限制。

## 使用建议

1. **不要一次性加载所有 skill** — Cursor 有上下文限制。将 2-3 个核心 skill 设为规则，并按需添加阶段专用 skill。
2. **明确引用 skill** — 告诉 Cursor "按照 test-driven-development 规则来完成这个改动"，以确保它读取已加载的规则。
3. **使用 Agent 进行审查** — 复制 `agents/code-reviewer.md` 的内容，告诉 Cursor "使用这个代码审查框架来审查这个 diff"。
4. **按需加载参考资料** — 进行性能工作时，将 `performance.md` 添加到 `.cursor/rules/`，或直接粘贴检查清单内容。
