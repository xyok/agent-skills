---
name: performance-optimization
description: 系统性地测量和优化应用程序性能。在响应速度或加载时间成为问题时使用，或者在部署影响用户体验关键指标的变更之前使用。
---

# 性能优化

## 概述

在有证据表明性能有问题时才优化，而不是在假设某些东西会很慢时优化。过早的优化会使代码复杂化，并使真正重要的性能优化更难发现。衡量、识别热点，然后针对性地优化。

**首要原则：先测量。** 在没有数据的情况下不要优化。直觉在性能方面经常是错误的，而测量会告诉你真正的瓶颈在哪里。

## 适用场景

- 页面加载缓慢或交互感觉迟钝
- 数据库查询超时
- API 端点响应时间过长
- 构建时间变长
- 用户抱怨性能问题
- 在部署会影响 Core Web Vitals 的变更之前

**不适用场景：** 没有证据表明有性能问题时，或当架构变更更紧迫时。

## Core Web Vitals 目标

| 指标 | 好 | 需要改进 | 差 |
|------|-----|---------|-----|
| LCP (Largest Contentful Paint) | ≤ 2.5s | 2.5–4.0s | > 4.0s |
| INP (Interaction to Next Paint) | ≤ 200ms | 200–500ms | > 500ms |
| CLS (Cumulative Layout Shift) | ≤ 0.1 | 0.1–0.25 | > 0.25 |

这些是用户体验的现实信号，而不是任意的基准。

## 测量优先工作流

```
1. BASELINE
   └── Measure current performance
       └── Use browser DevTools, Lighthouse, or profiler

2. IDENTIFY
   └── Find the actual bottleneck (not the assumed one)
       └── What's the slowest part?

3. HYPOTHESIZE
   └── Why is it slow?
       └── Network? Computation? Rendering? Database?

4. OPTIMIZE
   └── Make targeted changes

5. MEASURE
   └── Did it improve? By how much?
       └── Compare against baseline

6. REPEAT
   └── Only optimize further if still below targets
```

**规则：** 如果优化没有使指标明显改善，撤销它。无原则的"优化"会增加复杂性而没有收益。

## 前端性能

### 减少包大小

```bash
# Analyze what's in your bundle
npx @next/bundle-analyzer

# Or with webpack-bundle-analyzer
npx webpack-bundle-analyzer stats.json
```

```typescript
// ✗ Import entire library
import _ from 'lodash';
const sorted = _.sortBy(tasks, 'title');

// ✓ Tree-shakeable import
import { sortBy } from 'lodash-es';
const sorted = sortBy(tasks, 'title');
```

```typescript
// ✗ Eager load large component
import { HeavyEditor } from './HeavyEditor';

// ✓ Lazy load on demand
const HeavyEditor = dynamic(() => import('./HeavyEditor'), {
  loading: () => <EditorSkeleton />,
});
```

### 优化图像

```typescript
// ✓ Next.js Image component handles optimization
import Image from 'next/image';

<Image
  src="/hero.jpg"
  alt="Hero image"
  width={1200}
  height={600}
  priority  // For above-the-fold images
  sizes="(max-width: 768px) 100vw, 1200px"
/>
```

**图像优化清单：**
- 使用现代格式（WebP、AVIF）而不是 JPEG/PNG
- 正确调整大小（不要提供 2000px 宽的图像用于 400px 宽的容器）
- 延迟加载折叠以下的图像
- 为首屏图像添加 `priority`

### 防止布局偏移（CLS）

```css
/* Reserve space for images before they load */
.image-container {
  aspect-ratio: 16/9;  /* Reserves the correct space */
}

/* Reserve space for dynamic content */
.card-skeleton {
  min-height: 120px;  /* Matches the height of loaded content */
}
```

```typescript
// ✗ Injecting content above existing content causes layout shift
if (hasNotification) {
  return <div><NotificationBanner />{children}</div>;
}

// ✓ Reserve space for the banner always
return (
  <div>
    <div className={hasNotification ? 'visible' : 'invisible'}>
      <NotificationBanner />
    </div>
    {children}
  </div>
);
```

### React 性能

```typescript
// ✗ New function reference on every render triggers child re-renders
function TaskList({ tasks, onComplete }) {
  return tasks.map(task => (
    <Task
      key={task.id}
      task={task}
      onComplete={() => onComplete(task.id)}  // New function every render
    />
  ));
}

// ✓ Stable function reference with useCallback
function TaskList({ tasks, onComplete }) {
  const handleComplete = useCallback(
    (taskId: string) => onComplete(taskId),
    [onComplete]
  );

  return tasks.map(task => (
    <Task
      key={task.id}
      task={task}
      onComplete={handleComplete}
    />
  ));
}
```

```typescript
// ✗ Expensive computation on every render
function TaskStats({ tasks }) {
  const stats = calculateExpensiveStats(tasks);  // Runs every render
  return <StatsDisplay stats={stats} />;
}

// ✓ Memoize expensive computation
function TaskStats({ tasks }) {
  const stats = useMemo(
    () => calculateExpensiveStats(tasks),
    [tasks]  // Only recalculate when tasks change
  );
  return <StatsDisplay stats={stats} />;
}
```

**`useMemo`/`useCallback` 的使用准则：**
- 不要在所有地方都使用它们——记忆化本身有成本
- 在性能分析证明有问题之前不要使用
- 主要用于经过性能分析证实的昂贵计算
- 用于传递给优化子组件的引用（`React.memo` 包装的组件）

## 数据库性能

### 查找 N+1 查询

N+1 是最常见的数据库性能问题：1 个查询返回 N 条记录，然后每条记录又有 N 个查询。

```typescript
// ✗ N+1: 1 query for tasks + N queries for task owners
const tasks = await db.tasks.findMany({ where: { userId } });
for (const task of tasks) {
  task.owner = await db.users.findUnique({ where: { id: task.ownerId } });
}

// ✓ 1 query with JOIN
const tasks = await db.tasks.findMany({
  where: { userId },
  include: { owner: true },
});
```

### 索引策略

```sql
-- Find slow queries
EXPLAIN ANALYZE SELECT * FROM tasks WHERE user_id = 123 AND status = 'pending';

-- If it says "Seq Scan", you need an index
CREATE INDEX idx_tasks_user_status ON tasks(user_id, status);

-- For queries with ORDER BY
CREATE INDEX idx_tasks_created_at ON tasks(created_at DESC);

-- For LIKE queries (prefix search only)
CREATE INDEX idx_tasks_title ON tasks(title text_pattern_ops);
```

**索引规则：**
- 对查询中频繁出现在 `WHERE` 子句中的列建立索引
- 复合索引中，最具选择性的列放在前面
- 不要过度索引——索引会降低写性能
- 测量每个索引的影响

### 分页和限制

```typescript
// ✗ Load all records (catastrophic at scale)
const tasks = await db.tasks.findMany({ where: { userId } });

// ✓ Paginate
const tasks = await db.tasks.findMany({
  where: { userId },
  skip: (page - 1) * pageSize,
  take: pageSize,
  orderBy: { createdAt: 'desc' },
});

// ✓ Or use cursor-based pagination for better performance
const tasks = await db.tasks.findMany({
  where: { userId },
  take: pageSize,
  cursor: cursor ? { id: cursor } : undefined,
  orderBy: { id: 'desc' },
});
```

### 缓存策略

```typescript
// Cache expensive queries with short TTL
async function getTaskCountForUser(userId: string): Promise<number> {
  const cacheKey = `task-count:${userId}`;

  const cached = await redis.get(cacheKey);
  if (cached) return parseInt(cached);

  const count = await db.tasks.count({ where: { userId } });

  // Cache for 1 minute — acceptable staleness for a count
  await redis.setex(cacheKey, 60, count.toString());
  return count;
}
```

**何时缓存：**
- 昂贵的查询，结果不经常变化
- 计算结果，重新计算代价高
- 外部 API 响应，速率限制适用

**何时不缓存：**
- 数据必须精确实时的情况
- 频繁变更使缓存失效代价昂贵的情况

## 后端 API 性能

### 并发执行独立操作

```typescript
// ✗ Sequential: total time = time(A) + time(B) + time(C)
const user = await getUser(userId);
const tasks = await getTasks(userId);
const notifications = await getNotifications(userId);

// ✓ Concurrent: total time = max(time(A), time(B), time(C))
const [user, tasks, notifications] = await Promise.all([
  getUser(userId),
  getTasks(userId),
  getNotifications(userId),
]);
```

### 响应流

```typescript
// For large responses, stream instead of loading all into memory
app.get('/api/export/tasks', async (req, res) => {
  res.setHeader('Content-Type', 'application/json');
  res.write('[');

  let first = true;
  const cursor = db.tasks.cursor({ where: { userId: req.user.id } });

  for await (const task of cursor) {
    if (!first) res.write(',');
    res.write(JSON.stringify(task));
    first = false;
  }

  res.write(']');
  res.end();
});
```

## 发现性能问题的工具

| 工具 | 用途 |
|------|------|
| Chrome DevTools Performance 面板 | JS 执行分析 |
| Chrome DevTools Network 面板 | 请求大小、时序 |
| Lighthouse | 整体 Web Vitals 评分 |
| `EXPLAIN ANALYZE` | PostgreSQL 查询计划 |
| `pg_stat_statements` | 识别频繁/慢查询 |
| React DevTools Profiler | React 组件渲染性能 |
| Next.js Bundle Analyzer | 客户端包大小 |

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "这个可能很慢，我应该优化它" | 测量它。感觉慢不代表它是瓶颈。 |
| "我会稍后添加索引" | 在测试数据上工作的查询在生产数据下崩溃。及早添加索引。 |
| "在所有地方都用 useMemo 让它更快" | 记忆化有成本。过度使用会使代码更复杂并减慢速度。 |
| "我们以后可以缓存" | 引入缓存的架构复杂性——等到你有证据表明需要它。 |
| "性能优化是过早的" | 有些事情应该从一开始就考虑：分页、索引策略、N+1 避免。 |

## 危险信号

- 没有分页限制的列表端点
- 循环中的数据库查询（N+1 的标志）
- 没有索引的大表
- 没有基准的"性能优化"
- 到处都是 `useMemo` 和 `useCallback`（滥用）
- 没有懒加载的大型第三方库
- 不必要地加载到内存的完整数据集

## 验证

性能工作之后：

- [ ] 测量了基准（优化前的指标）
- [ ] 测量了改善（优化后的指标）
- [ ] Core Web Vitals 在目标范围内（LCP < 2.5s, INP < 200ms, CLS < 0.1）
- [ ] 没有 N+1 查询（使用 `include` 或 `JOIN`）
- [ ] 列表端点已分页
- [ ] 高频查询已建立适当的数据库索引
- [ ] 没有无理由的记忆化（只在有测量证据时使用）
