---
name: ci-cd-and-automation
description: 自动化 CI/CD 流水线配置。适用于建立或修改构建和部署流水线的场景。当需要自动化质量门禁、在 CI 中配置测试运行器或建立部署策略时使用。
---

# CI/CD 与自动化

## 概述

自动化质量门禁，确保没有任何变更在未通过测试、lint、类型检查和构建的情况下进入生产环境。CI/CD 是所有其他技能的执行机制——它能持续、一致地捕获人工和 agent 遗漏的问题。

**左移原则：** 在流水线尽可能早的阶段捕获问题。在 lint 阶段发现的 bug 需要数分钟处理；同样的 bug 在生产环境中发现则需要数小时。将检查移到上游——静态分析在测试之前，测试在预发布之前，预发布在生产之前。

**越快越安全：** 更小的批次和更频繁的发布降低风险，而不是增加风险。包含 3 个变更的部署比包含 30 个变更的部署更容易调试。频繁发布会建立对发布流程本身的信心。

## 适用场景

- 为新项目建立 CI 流水线
- 添加或修改自动化检查
- 配置部署流水线
- 当变更应触发自动化验证时
- 调试 CI 失败

## 质量门禁流水线

每个变更在合并前都经过以下门禁：

```
Pull Request Opened
    │
    ▼
┌─────────────────┐
│   LINT CHECK     │  eslint, prettier
│   ↓ pass         │
│   TYPE CHECK     │  tsc --noEmit
│   ↓ pass         │
│   UNIT TESTS     │  jest/vitest
│   ↓ pass         │
│   BUILD          │  npm run build
│   ↓ pass         │
│   INTEGRATION    │  API/DB tests
│   ↓ pass         │
│   E2E (optional) │  Playwright/Cypress
│   ↓ pass         │
│   SECURITY AUDIT │  npm audit
│   ↓ pass         │
│   BUNDLE SIZE    │  bundlesize check
└─────────────────┘
    │
    ▼
  Ready for review
```

**任何门禁都不能跳过。** 如果 lint 失败，修复 lint——不要禁用规则。如果测试失败，修复代码——不要跳过测试。

## GitHub Actions 配置

### 基本 CI 流水线

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npx tsc --noEmit

      - name: Test
        run: npm test -- --coverage

      - name: Build
        run: npm run build

      - name: Security audit
        run: npm audit --audit-level=high
```

### 带数据库集成测试

```yaml
  integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: ci_user
          POSTGRES_PASSWORD: ${{ secrets.CI_DB_PASSWORD }}
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - name: Run migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://ci_user:${{ secrets.CI_DB_PASSWORD }}@localhost:5432/testdb
      - name: Integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://ci_user:${{ secrets.CI_DB_PASSWORD }}@localhost:5432/testdb
```

> **注意：** 即使是 CI 专用的测试数据库，也应使用 GitHub Secrets 存储凭证，而不是硬编码值。这培养了良好习惯，并防止测试凭证被意外复用到其他场景。

### E2E 测试

```yaml
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - name: Install Playwright
        run: npx playwright install --with-deps chromium
      - name: Build
        run: npm run build
      - name: Run E2E tests
        run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
```

## 将 CI 失败反馈给 Agent

CI 与 AI agent 结合的力量在于反馈闭环。当 CI 失败时：

```
CI fails
    │
    ▼
Copy the failure output
    │
    ▼
Feed it to the agent:
"The CI pipeline failed with this error:
[paste specific error]
Fix the issue and verify locally before pushing again."
    │
    ▼
Agent fixes → pushes → CI runs again
```

**关键模式：**

```
Lint failure → Agent runs `npm run lint --fix` and commits
Type error  → Agent reads the error location and fixes the type
Test failure → Agent follows debugging-and-error-recovery skill
Build error → Agent checks config and dependencies
```

## 部署策略

### 预览部署

每个 PR 都获得一个预览部署以进行手动测试：

```yaml
# Deploy preview on PR (Vercel/Netlify/etc.)
deploy-preview:
  runs-on: ubuntu-latest
  if: github.event_name == 'pull_request'
  steps:
    - uses: actions/checkout@v4
    - name: Deploy preview
      run: npx vercel --token=${{ secrets.VERCEL_TOKEN }}
```

### 功能标志

功能标志将部署与发布解耦。在标志后面部署未完成或有风险的功能，以便：

- **在不启用的情况下发布代码。** 尽早合并到 main，准备好时再启用。
- **无需重新部署即可回滚。** 禁用标志而不是回退代码。
- **金丝雀新功能。** 先为 1% 的用户启用，然后是 10%，再到 100%。
- **运行 A/B 测试。** 比较有无功能时的行为差异。

```typescript
// Simple feature flag pattern
if (featureFlags.isEnabled('new-checkout-flow', { userId })) {
  return renderNewCheckout();
}
return renderLegacyCheckout();
```

**标志生命周期：** 创建 → 启用测试 → 金丝雀 → 全量发布 → 删除标志和废弃代码。永久存在的标志会变成技术债务——创建时就设置清理日期。

### 分阶段发布

```
PR merged to main
    │
    ▼
  Staging deployment (auto)
    │ Manual verification
    ▼
  Production deployment (manual trigger or auto after staging)
    │
    ▼
  Monitor for errors (15-minute window)
    │
    ├── Errors detected → Rollback
    └── Clean → Done
```

### 回滚计划

每次部署都应该可以撤销：

```yaml
# Manual rollback workflow
name: Rollback
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to rollback to'
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - name: Rollback deployment
        run: |
          # Deploy the specified previous version
          npx vercel rollback ${{ inputs.version }}
```

## 环境管理

```
.env.example       → Committed (template for developers)
.env                → NOT committed (local development)
.env.test           → Committed (test environment, no real secrets)
CI secrets          → Stored in GitHub Secrets / vault
Production secrets  → Stored in deployment platform / vault
```

CI 永远不应该拥有生产环境密钥。为 CI 测试使用独立的密钥。

## CI 之外的自动化

### Dependabot / Renovate

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
```

### 构建值班角色

指定专人负责保持 CI 为绿色状态。当构建损坏时，值班员的职责是修复或回退——而不是造成问题的人。这防止了每个人都假设别人会修复问题的情况下，损坏的构建不断积累。

### PR 检查

- **必须审查：** 合并前至少 1 个批准
- **必须通过状态检查：** 合并前 CI 必须通过
- **分支保护：** main 分支禁止强制推送
- **自动合并：** 如果所有检查通过且已批准，则自动合并

## CI 优化

当流水线超过 10 分钟时，按影响大小依次应用以下策略：

```
Slow CI pipeline?
├── Cache dependencies
│   └── Use actions/cache or setup-node cache option for node_modules
├── Run jobs in parallel
│   └── Split lint, typecheck, test, build into separate parallel jobs
├── Only run what changed
│   └── Use path filters to skip unrelated jobs (e.g., skip e2e for docs-only PRs)
├── Use matrix builds
│   └── Shard test suites across multiple runners
├── Optimize the test suite
│   └── Remove slow tests from the critical path, run them on a schedule instead
└── Use larger runners
    └── GitHub-hosted larger runners or self-hosted for CPU-heavy builds
```

**示例：缓存和并行化**
```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci
      - run: npm run lint

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci
      - run: npx tsc --noEmit

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci
      - run: npm test -- --coverage
```

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "CI 太慢了" | 优化流水线（见 CI 优化），不要跳过它。一个 5 分钟的流水线可以防止数小时的调试。 |
| "这个变更很小，跳过 CI" | 微小的变更也会破坏构建。CI 对于微小的变更本来就很快。 |
| "测试不稳定，重新运行就好" | 不稳定的测试会掩盖真实的 bug 并浪费所有人的时间。修复不稳定性。 |
| "我们以后再添加 CI" | 没有 CI 的项目会积累损坏的状态。第一天就建立它。 |
| "手动测试就够了" | 手动测试无法扩展且不可重复。尽可能自动化。 |

## 危险信号

- 项目中没有 CI 流水线
- CI 失败被忽略或消音
- 测试在 CI 中被禁用以使流水线通过
- 生产部署没有预发布验证
- 没有回滚机制
- 密钥存储在代码或 CI 配置文件中（而不是密钥管理器）
- CI 时间很长但没有优化努力

## 验证

建立或修改 CI 后：

- [ ] 所有质量门禁都存在（lint、类型检查、测试、构建、安全审计）
- [ ] 流水线在每个 PR 和推送到 main 时运行
- [ ] 失败会阻塞合并（已配置分支保护）
- [ ] CI 结果反馈到开发循环中
- [ ] 密钥存储在密钥管理器中，而不是代码中
- [ ] 部署有回滚机制
- [ ] 测试套件的流水线运行时间在 10 分钟以内
