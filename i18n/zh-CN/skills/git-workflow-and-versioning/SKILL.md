---
name: git-workflow-and-versioning
description: 管理 Git 提交、分支和版本发布。适用于提交代码变更、合并特性分支以及标记发布版本时使用。
---

# Git 工作流与版本管理

## 概述

通过原子提交、简洁的历史记录和可预测的发布来保持代码库健康。好的 Git 实践使调试更快（`git bisect` 有效）、代码审查更简单（小型聚焦的 PR），以及回滚更安全。

## 适用场景

- 提交代码变更
- 创建特性分支
- 合并 PR
- 标记发布版本
- 恢复损坏的状态

## 核心工作流：基于主干的开发

```
main (always deployable)
  │
  ├── feature/add-task-labels     (short-lived, < 2 days)
  ├── fix/task-pagination-bug     (short-lived, < 1 day)
  └── chore/update-dependencies   (short-lived, < 1 day)
```

**规则：**
- `main` 始终可部署——永不直接推送损坏的代码
- 特性分支存活时间短（理想情况下不超过 2 天）
- 在 PR 审查通过并且 CI 通过之后合并
- 长期存活的分支会创建合并地狱——小型增量工作

### 为什么不用长期分支？

`feature/new-dashboard`（存活 3 周）的问题：
- 与 `main` 分歧，导致大型合并冲突
- PR 变得太大无法有效审查
- 其他功能被阻塞，等待大型合并完成
- 调试时无法追踪哪个变更导致了问题

**解决方案：** 用功能标志替代长期分支。在标志后面增量提交，合并到 `main`，当准备好时启用该功能。

## 原子提交

每个提交都应该恰好包含一个逻辑变更，所有测试通过。

```
✓ Atomic commits:
  abc123 "Add task label schema and migration"
  def456 "Add label API endpoints"
  ghi789 "Add label UI component"
  jkl012 "Add label filtering to task list"

✗ Non-atomic commits:
  abc123 "WIP"
  def456 "more stuff"
  ghi789 "fix tests and also add labels and update readme"
  jkl012 "finally done"
```

**为什么这很重要：**
- `git bisect` 能精确定位引入 bug 的提交
- `git revert <sha>` 能安全地撤销一个变更
- 代码审查可以逐提交进行，而不是审查 500 行 diff
- 提交历史作为项目演进的文档

## 提交消息规范

使用约定式提交格式：

```
type(scope): subject

[optional body]

[optional footer]
```

### 类型

| 类型 | 使用场景 |
|------|----------|
| `feat` | 新功能 |
| `fix` | Bug 修复 |
| `docs` | 文档变更 |
| `style` | 格式化（不影响功能的空白、分号等）|
| `refactor` | 既不修复 bug 也不添加功能的代码变更 |
| `test` | 添加或修正测试 |
| `chore` | 构建过程、依赖更新等维护性变更 |
| `perf` | 提升性能的变更 |

### 示例

```
feat(tasks): add color-coded labels to tasks

Labels allow users to categorize tasks by project, priority, or context.
Up to 5 labels can be assigned per task.

Implements: #123

---

fix(auth): prevent session token reuse after logout

Logout now invalidates the session token in the database.
Previously, a captured token could be replayed after logout.

Closes: #456

---

refactor(api): extract validation to dedicated middleware

Reduces duplication across route handlers. Validation errors
now return consistent 422 responses with field-level details.

---

feat!: rename createTask to createNewTask

BREAKING CHANGE: `createTask` is now `createNewTask`.
Update all usages before upgrading.
```

### 提交消息规则

```
✓ Use imperative mood: "Add", "Fix", "Remove" (not "Added", "Fixed")
✓ Subject line ≤ 72 characters
✓ Explain what and why, not how (the code shows how)
✓ Reference issues/PRs when relevant
✗ "WIP", "stuff", "asdfasdf"
✗ "Fixed bug" (which bug? what was wrong?)
✗ Mixing unrelated changes in one commit
```

## Git 工作区（Worktrees）

Git worktrees 允许你同时检出多个分支——无需存储更改：

```bash
# Create a new worktree for a feature branch
git worktree add ../repo-feature feature/add-labels

# Now you can work in both simultaneously:
# ~/repo          → main or current branch
# ~/repo-feature  → feature/add-labels

# List active worktrees
git worktree list

# Remove when done
git worktree remove ../repo-feature
```

**什么时候使用：**
- 调查 bug，同时不中断当前工作
- 在两个特性之间来回切换
- 需要在当前上下文中运行旧版本

## 有用的 Git 操作

### 准备提交

```bash
# Stage specific files
git add src/tasks/labels.ts src/tasks/labels.test.ts

# Stage specific lines within a file (interactive)
git add -p src/tasks/labels.ts

# See what's staged before committing
git diff --staged

# Amend the last commit (before pushing)
git commit --amend
```

### 调查历史

```bash
# Find when a specific line changed
git blame src/services/task.service.ts

# Search commit messages
git log --oneline --grep="pagination"

# See all commits that touched a file
git log --oneline -- src/services/task.service.ts

# See what changed between two points
git diff main...feature/add-labels

# Find the commit that introduced a bug (binary search)
git bisect start
git bisect bad HEAD
git bisect good v2.1.0
# Test each checkout, then:
# git bisect good / git bisect bad
git bisect reset
```

### 恢复更改

```bash
# Undo last commit, keep changes staged
git reset --soft HEAD~1

# Undo last commit, keep changes unstaged
git reset --mixed HEAD~1

# Undo last commit, discard changes (DESTRUCTIVE)
git reset --hard HEAD~1

# Revert a specific commit (creates a new revert commit)
git revert <commit-sha>

# Revert a specific commit without auto-committing
git revert --no-commit <commit-sha>
```

**规则：** 在已推送的提交上使用 `git revert` 而不是 `git reset`。Revert 会创建新的提交（安全），reset 会重写历史（危险，破坏其他人的工作）。

### Cherry-pick

```bash
# Apply a specific commit from another branch
git cherry-pick <commit-sha>

# Cherry-pick a range of commits
git cherry-pick <start-sha>^..<end-sha>
```

## 版本标签和发布

### 语义化版本

```
v{MAJOR}.{MINOR}.{PATCH}

MAJOR: Breaking changes (removed features, changed interfaces)
MINOR: New features, backward-compatible
PATCH: Bug fixes, no interface changes

Examples:
v1.0.0 → v1.1.0  (new feature)
v1.1.0 → v1.1.1  (bug fix)
v1.1.1 → v2.0.0  (breaking change)
```

### 创建发布

```bash
# Tag the current commit
git tag -a v2.3.0 -m "Release v2.3.0: Add task labels and bulk operations"

# Push the tag
git push origin v2.3.0

# Create a GitHub release (if using GitHub CLI)
gh release create v2.3.0 --title "v2.3.0" --notes-file CHANGELOG.md
```

## 保护 main 分支

配置分支保护：

```yaml
# In GitHub repository settings (or via GitHub API):
- Require pull request before merging
- Require at least 1 approving review
- Require status checks to pass (CI must be green)
- No force pushes to main
- No direct pushes to main
```

这确保了：
- `main` 中的每个变更都经过审查
- CI 通过后代码才能到达 `main`
- 历史不会被重写（保护了 `git bisect`）

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "我会之后整理提交" | 提交之后会被忘记。在工作时就写原子提交。 |
| "提交消息不重要" | 三个月后你会搜索 git log 来找到为什么做了某个决定。清晰的消息会有帮助。 |
| "长期特性分支有效" | 直到合并时——然后你花了一整天解决冲突。用功能标志代替。 |
| "我直接推送到 main，更快" | 直到一个 bug 绕过了审查进入生产环境。保护 main。 |
| "强制推送修复了问题" | 强制推送重写了历史，破坏了拉取分支的其他人的工作。使用 revert。 |

## 危险信号

- 直接推送到 `main`（无 PR 审查）
- 存活超过 3 天的特性分支
- 提交消息："WIP"、"fix"、"更多内容"、"asdf"
- 一个提交中包含多个无关变更
- `git reset --hard` 在已推送的提交上
- 标签没有遵循语义化版本
- 没有为发布版本创建标签

## 验证

提交工作之后：

- [ ] 每个提交恰好包含一个逻辑变更
- [ ] 提交消息遵循约定式提交格式
- [ ] 所有测试在每个提交后都通过
- [ ] PR 是小的和聚焦的（< 400 行）
- [ ] CI 在合并前为绿色
- [ ] 没有直接推送到 `main`
- [ ] 发布版本用语义化版本标签标记
