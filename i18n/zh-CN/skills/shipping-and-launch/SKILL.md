---
name: shipping-and-launch
description: 将功能安全地发布到生产环境。在准备部署时使用——运行预发布清单、验证监控，以及协调分阶段发布或功能标志推出。
---

# 发布与上线

## 概述

系统性地将功能从完成状态带到用户可用状态，同时最小化风险。发布不是将代码合并到 `main`——那只是起点。发布是在完整的质量保证之后，以可监控的方式交付验证过的代码，并有明确的回滚计划。

## 适用场景

- 将新功能发布到生产环境
- 重大变更部署（数据库迁移、API 变更）
- 协调多服务发布
- 功能标志推出

## 发布前清单

在部署任何重大变更之前，完成以下所有步骤：

### 代码质量

```
- [ ] CI 通过（所有测试为绿色）
- [ ] 代码已审查并获得批准
- [ ] 没有控制台错误或警告
- [ ] 没有调试代码或 console.log 留在代码中
- [ ] 待处理的 TODO 已记录为 issue，而不是留在代码中
```

### 测试覆盖

```
- [ ] 新功能有单元测试
- [ ] 关键路径有集成测试
- [ ] 已手动测试关键用户流程
- [ ] 已测试边界情况和错误场景
- [ ] 已在与生产相似的环境中测试（预发布/staging）
```

### 数据库和迁移

```
- [ ] 迁移已在 staging 上运行并成功
- [ ] 迁移是向后兼容的（如果需要零停机时间）
- [ ] 备份已完成（如果是破坏性迁移）
- [ ] 大表迁移已规划停机时间或使用在线迁移工具
```

### 安全性

```
- [ ] npm audit 无关键或高漏洞
- [ ] 没有密钥或凭证暴露
- [ ] 新端点已进行认证和授权
- [ ] 用户输入已在边界处验证
```

### 监控和可观测性

```
- [ ] 错误监控已配置（Sentry 或类似工具）
- [ ] 关键操作有日志记录
- [ ] 性能指标已建立基准
- [ ] 已定义警报阈值（错误率、延迟）
```

### 文档

```
- [ ] README 已更新（如果添加了新功能）
- [ ] API 文档已更新（如果 API 已更改）
- [ ] 变更日志已更新
- [ ] 破坏性变更已沟通给消费者
```

## 零停机时间部署

对于需要数据库迁移的变更，按照"扩展-收缩"模式进行：

```
Phase 1: EXPAND (backward-compatible change)
  - Deploy code that works with both old and new schema
  - Run migration that adds new columns/tables (non-breaking)
  - New code uses new schema; old code still works

Phase 2: MIGRATE (data migration if needed)
  - Migrate existing data to new format
  - Done in background jobs, not in the migration itself for large tables

Phase 3: CONTRACT (remove old code)
  - Remove the compatibility code that handles the old schema
  - Remove old columns that are no longer needed
```

**示例：重命名列**

```sql
-- ✗ Breaking: old code breaks immediately
ALTER TABLE tasks RENAME COLUMN name TO title;

-- ✓ Non-breaking migration:
-- Step 1: Add new column
ALTER TABLE tasks ADD COLUMN title TEXT;
-- Step 2: Deploy code that writes to both columns
-- Step 3: Backfill data
UPDATE tasks SET title = name WHERE title IS NULL;
-- Step 4: Deploy code that only uses new column
-- Step 5: Drop old column
ALTER TABLE tasks DROP COLUMN name;
```

## 功能标志发布

对于有风险或复杂的功能，使用功能标志进行分阶段发布：

```
Stage 1: Internal testing (1% of users = your team)
  - Enable for your own accounts
  - Verify core functionality works

Stage 2: Canary (5-10% of users)
  - Monitor error rates
  - Monitor performance metrics
  - Gather early user feedback

Stage 3: Gradual rollout (10% → 25% → 50% → 100%)
  - Each stage: monitor for 24-48 hours
  - If metrics are clean, proceed to next stage
  - If issues emerge, roll back to previous stage

Stage 4: Full release (100%)
  - Remove feature flag
  - Clean up old code path
```

```typescript
// Feature flag implementation
async function handleTaskRequest(req: Request) {
  const flag = await featureFlags.isEnabled('new-task-engine', {
    userId: req.user.id,
  });

  if (flag) {
    return newTaskEngine.process(req);
  }
  return legacyTaskEngine.process(req);
}
```

## 监控关键指标

发布后，在前 15-30 分钟密切监控：

```
Error rate: Should not increase significantly after deployment
  - Alert threshold: > 1% error rate (or 2x baseline)

Response time: p95 should stay within normal range
  - Alert threshold: p95 > 2x baseline

Throughput: Requests/second should be stable
  - Alert threshold: drop > 20% (may indicate errors blocking traffic)

Business metrics: 
  - Key user actions completing successfully
  - No drop in conversion rates for critical flows
```

### 回滚决策标准

```
ROLLBACK immediately if:
  - Error rate > 5% (absolute)
  - Error rate > 3x baseline
  - Core functionality is broken (can't create/view primary resource)
  - Data is being corrupted

MONITOR and decide if:
  - Error rate is elevated but below threshold
  - Performance degraded but still within SLA
  - Edge case errors affecting < 1% of users

PROCEED if:
  - Metrics are clean
  - Known expected errors (from expected behavior changes) are not increasing
```

## 数据库迁移安全

对于生产数据库：

```bash
# Always test migrations on a copy of production data first
# 1. Take a snapshot
pg_dump production_db > backup_$(date +%Y%m%d_%H%M%S).sql

# 2. Run migration on snapshot
psql staging_db < backup.sql
npx prisma migrate deploy  # Test on staging

# 3. Verify data integrity
# Check row counts, spot-check critical records

# 4. Run on production with monitoring
npx prisma migrate deploy
```

**大表迁移（> 100 万行）：**
```sql
-- ✗ Locks the table, causes downtime
ALTER TABLE tasks ADD COLUMN priority INTEGER DEFAULT 0;

-- ✓ Non-locking alternatives:
-- For PostgreSQL: use concurrent index creation
CREATE INDEX CONCURRENTLY idx_tasks_priority ON tasks(priority);

-- For adding columns: nullable first, then default
ALTER TABLE tasks ADD COLUMN priority INTEGER;
UPDATE tasks SET priority = 0 WHERE priority IS NULL;  -- In batches
ALTER TABLE tasks ALTER COLUMN priority SET DEFAULT 0;
```

## 发布沟通

对于可见的用户影响变更：

```
Internal announcement (to your team, before launch):
  - What's launching
  - What to watch for
  - Who to contact if there are issues
  - Rollback plan

External announcement (to users, if applicable):
  - What's new and why it's valuable
  - Any action users need to take
  - Where to get help
  
Post-launch retrospective:
  - What went well
  - What could be improved
  - Any incidents and their root causes
```

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "CI 通过了，可以发布了" | CI 验证代码是否工作，而不是功能是否正确。手动测试是必需的。 |
| "发布后我会监控" | 在不知道你在监控什么的情况下监控是无效的。提前建立基准和警报。 |
| "我们可以快速修复，不需要回滚" | 在生产中修复时，更多用户会受到影响。为快速回滚而设计，而不是快速修复。 |
| "功能标志太过了，这只是一个小变更" | 即使是小变更也会破坏东西。功能标志使回滚变成配置更改而不是部署。 |
| "迁移只需几秒钟，不需要备份" | 迁移失败是有可能的。总是备份生产数据。 |

## 危险信号

- 在验证 staging 之前部署到生产环境
- 没有回滚计划的数据库迁移
- 没有监控或警报的发布
- 在没有分阶段推出的情况下发布给所有用户
- 在上班时间之外部署（使回滚更难）
- 将多个高风险变更合并为一次部署
- 代码中留有调试日志

## 验证

在发布之前：

- [ ] 发布前清单的所有条目已完成
- [ ] staging 环境已验证
- [ ] 数据库迁移已在 staging 上测试
- [ ] 监控和警报已到位
- [ ] 回滚计划已记录并可执行
- [ ] 团队已准备好在发布后 30 分钟内监控

在发布之后：

- [ ] 15 分钟内无错误率上升
- [ ] 性能指标在正常范围内
- [ ] 关键用户流程已手动验证
- [ ] 功能标志（如果使用）已按计划推进
