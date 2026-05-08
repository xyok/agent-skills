---
name: code-review-and-quality
description: 对代码变更进行系统性的五轴审查：正确性、可读性、架构、安全性和性能。适用于在提交前或响应拉取请求时审查代码时使用。
---

# 代码审查与质量

## 概述

在将代码变更发布到生产环境之前，从五个维度对其进行系统性审查。这是人类审查者和 AI agent 的检查框架——它需要深度，而不是表面的浏览。一次好的代码审查会发现真实的 bug、识别架构问题，并使代码库更易于所有人维护。

## 适用场景

- 提交代码变更之前
- 响应 PR 审查请求时
- 评估重构方案时
- 在合并前理解变更影响时

**不适用场景：** 当你只需要快速检查语法或 lint 问题时——那用 linter。代码审查针对 linter 无法捕获的问题。

## 五轴审查框架

### 轴一：正确性

代码按预期运行吗？

**检查什么：**
- 逻辑错误和边界条件
- 空值/undefined 处理
- 并发问题（竞争条件、死锁）
- 数值错误（溢出、精度、舍入）
- 错误传播和恢复
- 数据变化（是否有意？）

**常见模式：**

```typescript
// ✗ Race condition: reads, then writes based on stale value
const count = await db.tasks.count({ where: { userId } });
if (count < 5) {
  await db.tasks.create({ data: { userId, ... } });
}

// ✓ Atomic operation: let the database handle concurrency
await db.$transaction(async (tx) => {
  const count = await tx.tasks.count({ where: { userId }, select: { id: true } });
  if (count >= 5) throw new Error('Task limit reached');
  return tx.tasks.create({ data: { userId, ... } });
});
```

```typescript
// ✗ Unhandled null: crashes when user doesn't have an address
const city = user.address.city;

// ✓ Defensive access with explicit fallback
const city = user.address?.city ?? 'Unknown';
```

**提问：**
- 这段代码是否处理了所有输入情况（空、null、零、负数、超大数）？
- 如果这个异步操作失败，会发生什么？
- 如果多个请求同时命中这段代码，会发生什么？

### 轴二：可读性

代码是否清晰表达了意图？

**检查什么：**
- 变量和函数命名是否揭示意图
- 函数/方法大小和聚焦度
- 注释是否解释了"为什么"而不是"做什么"
- 常量的使用替代魔法数字
- 流程的一致性

**常见模式：**

```typescript
// ✗ Cryptic: what do x, d, c mean?
function calc(x: number, d: number, c: number): number {
  return x * d * (1 - c / 100);
}

// ✓ Expressive: names explain the domain
function calculateDiscountedPrice(
  basePrice: number,
  quantity: number,
  discountPercent: number
): number {
  return basePrice * quantity * (1 - discountPercent / 100);
}
```

```typescript
// ✗ Magic numbers with no explanation
if (score > 750) { ... }
setTimeout(fn, 86400000);

// ✓ Named constants explain the meaning
const CREDIT_SCORE_EXCELLENT_THRESHOLD = 750;
const ONE_DAY_MS = 24 * 60 * 60 * 1000;

if (score > CREDIT_SCORE_EXCELLENT_THRESHOLD) { ... }
setTimeout(fn, ONE_DAY_MS);
```

**提问：**
- 我在没有上下文的情况下能理解这个函数的作用吗？
- 六个月后再读这段代码，它还清晰吗？
- 注释是否解释了为什么，而不仅仅是什么？

### 轴三：架构

变更是否符合系统设计目标？

**检查什么：**
- 关注点分离
- 模块耦合
- 抽象层级
- 依赖方向
- 职责分配
- 层级违规（UI 代码调用数据库，业务逻辑渗入 UI）

**常见模式：**

```typescript
// ✗ UI component directly calling the database
function TaskList() {
  const [tasks, setTasks] = useState([]);

  useEffect(() => {
    db.tasks.findMany({ where: { userId: currentUser.id } }).then(setTasks);
  }, []);
}

// ✓ UI component using a service layer
function TaskList() {
  const tasks = useTaskQuery({ userId: currentUser.id });
  // UI only knows about tasks, not how they're fetched
}
```

**提问：**
- 这个变更是否引入了不应该存在的层级依赖？
- 这是否让将来更改任一部分变得更容易还是更困难？
- 这是否创建了循环依赖？
- 这在正确的抽象层级上吗？

### 轴四：安全性

代码是否引入了漏洞？

**检查什么：**
- 注入攻击（SQL、命令、LDAP）
- 用户输入净化
- 认证和授权检查
- 敏感数据暴露
- 密钥/凭证处理
- 第三方依赖风险

**常见模式：**

```typescript
// ✗ SQL injection: user input directly in query
const users = await db.query(
  `SELECT * FROM users WHERE email = '${req.body.email}'`
);

// ✓ Parameterized query: safe from injection
const users = await db.query(
  'SELECT * FROM users WHERE email = $1',
  [req.body.email]
);
```

```typescript
// ✗ Missing authorization check
app.delete('/api/tasks/:id', async (req, res) => {
  await db.tasks.delete({ where: { id: req.params.id } });
  res.sendStatus(204);
});

// ✓ Verify ownership before allowing mutation
app.delete('/api/tasks/:id', authenticate, async (req, res) => {
  const task = await db.tasks.findUnique({ where: { id: req.params.id } });
  if (!task || task.userId !== req.user.id) {
    return res.sendStatus(403);
  }
  await db.tasks.delete({ where: { id: req.params.id } });
  res.sendStatus(204);
});
```

**提问：**
- 用户输入在使用前是否净化？
- 是否所有变更都经过认证和授权？
- 错误消息中是否暴露了敏感信息？
- 是否有新的依赖引入了已知漏洞？

详细安全审查，参见 `security-and-hardening`。

### 轴五：性能

变更是否引入了性能回归？

**检查什么：**
- N+1 查询
- 不必要的重新渲染（UI）
- 内存泄漏
- 昂贵操作在热路径上
- 缓存机会
- 大量内存分配

**常见模式：**

```typescript
// ✗ N+1 query: 1 query for tasks, then N queries for each user
const tasks = await db.tasks.findMany({ where: { completed: false } });
for (const task of tasks) {
  task.owner = await db.users.findUnique({ where: { id: task.userId } });
}

// ✓ Single query with join
const tasks = await db.tasks.findMany({
  where: { completed: false },
  include: { owner: true },
});
```

**提问：**
- 这段代码在规模下如何表现（1000 个用户，100 万行数据）？
- 是否有数据库调用或 API 请求可以合并或缓存？
- 这是否修改了用户在关键渲染路径上需要等待的内容？

详细性能审查，参见 `performance-optimization`。

## 变更大小指南

### 为什么大小很重要

大型 PR 是审查质量的最大威胁。研究表明，人类在变更超过 400 行后会开始失去检测 bug 的能力。好的审查需要深度专注——面对超大的 diff，这几乎是不可能的。

```
< 100 lines    → Complete review possible, high confidence
100–400 lines  → Full review achievable with effort
400–800 lines  → Review quality degrades, break it up
800+ lines     → Split the PR — it will get a rubber stamp
```

### 拆分大型 PR

如果变更超过 400 行（不含测试），寻找机会拆分：

1. **逻辑层次拆分：** 数据层先于 UI 层
2. **功能拆分：** 基础设施 PR 先于功能 PR
3. **提取公共变更：** 重命名/重构单独一个 PR
4. **分阶段功能：** 使用功能标志在合并后关闭未完成的功能

**技术债的信号：** 如果一个功能在不碰触大量无关代码的情况下无法实现，可能存在过度耦合——将修复提取到单独的重构 PR 中。

## 审查评论级别

并非所有评论都具有相同分量。明确标记严重性：

| 前缀 | 含义 | 是否阻塞合并 |
|------|------|------------|
| `bug:` | 必须修复的正确性问题 | 是 |
| `security:` | 安全漏洞 | 是 |
| `concern:` | 值得讨论的重要问题 | 是（直到解决） |
| `suggestion:` | 改进意见，非阻塞 | 否 |
| `nit:` | 次要格式/命名问题 | 否 |
| `question:` | 征询意见，非阻塞 | 否 |

非阻塞评论永远不应该阻止合并——否则审查者应该升级其严重性或删除它。

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "代码能运行，所以没问题" | 运行和正确是不同的。考虑边界情况、并发性和错误路径。 |
| "这是一个小 PR，不需要完整审查" | 小的 PR 经常包含微小的 bug。快速审查取决于代码，而不是大小。 |
| "我稍后会修复这个 TODO" | TODO 很少被修复。如果现在不重要，就删除它或创建一个跟踪 issue。 |
| "审查者太挑剔了" | 批评代码，不是人。每一条评论都应该解释为什么。 |
| "这只是重构，不需要测试" | 重构会破坏行为——这正是我们为重构编写测试的原因。 |
| "我了解代码，不需要审查" | 代码作者是最不适合审查自己代码的人。全力盲目地看自己的错误。 |

## 危险信号

- 跨多个不相关功能的 PR
- 测试覆盖率没有随之增加的新功能
- 未处理的错误情况
- 直接字符串拼接查询（SQL 注入风险）
- 用户修改操作缺少授权检查
- 注释解释"做什么"而不是"为什么"
- 硬编码密钥或凭证
- 没有服务层的 N+1 查询

## 验证

对变更进行审查之后：

- [ ] **正确性：** 验证了边界条件和错误路径
- [ ] **可读性：** 没有神秘的命名或魔法数字
- [ ] **架构：** 没有层级违规或意外耦合
- [ ] **安全性：** 输入净化，鉴权检查，无暴露密钥
- [ ] **性能：** 热路径中没有 N+1 查询或昂贵操作
- [ ] **大小：** PR 小于 400 行或合理地拆分
- [ ] **测试：** 新行为有测试覆盖
- [ ] 所有 `bug:` 和 `security:` 评论在合并前已解决
