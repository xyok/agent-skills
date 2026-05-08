---
name: code-simplification
description: 通过降低复杂度来简化代码，提升清晰度和可维护性，同时不改变行为。当代码变得难以理解、修改或扩展时使用。
---

# 代码简化

## 概述

将难以理解的代码转化为清晰、聚焦的代码，同时精确保留其行为。简单代码并不意味着短代码——它意味着读者第一次就能理解的代码，下周修改时不会感到害怕，以及新的 bug 难以藏匿其中的代码。

## 适用场景

- 当代码需要超过 5 分钟才能理解
- 当变更被同时触及多个不相关代码块所阻碍
- 当同一 bug 不断重复出现时
- 在功能开发后的清理阶段
- 当代码审查评论说"太复杂了"

**不适用场景：** 当代码运行良好且不需要更改时。不要为简化而简化——只在有理由时进行，例如将要修改代码、测试困难或发现 bug 较多。

## 五个简化原则

### 原则 1：消除重复

重复代码意味着多个更新位置——分歧的来源。

```typescript
// Before: three identical validation blocks
function createUser(email: string) {
  if (!email || !email.includes('@')) {
    throw new Error('Invalid email');
  }
  // ...
}

function updateUserEmail(email: string) {
  if (!email || !email.includes('@')) {
    throw new Error('Invalid email');
  }
  // ...
}

function verifyEmail(email: string) {
  if (!email || !email.includes('@')) {
    throw new Error('Invalid email');
  }
  // ...
}

// After: single validation function
function validateEmail(email: string): void {
  if (!email || !email.includes('@')) {
    throw new Error('Invalid email');
  }
}

function createUser(email: string) {
  validateEmail(email);
  // ...
}

function updateUserEmail(email: string) {
  validateEmail(email);
  // ...
}

function verifyEmail(email: string) {
  validateEmail(email);
  // ...
}
```

**规则：** 当一段代码在三个地方以上出现时，提取它。对于两处重复，判断是否会继续增长。

### 原则 2：每个函数做一件事

做多件事的函数难以命名、测试和修改。

```typescript
// Before: one function doing four things
async function processUserRegistration(req: Request, res: Response) {
  // 1. Validate input
  if (!req.body.email || !req.body.password) {
    return res.status(400).json({ error: 'Email and password required' });
  }
  if (req.body.password.length < 8) {
    return res.status(400).json({ error: 'Password too short' });
  }

  // 2. Create the user
  const hashedPassword = await bcrypt.hash(req.body.password, 10);
  const user = await db.users.create({
    data: { email: req.body.email, password: hashedPassword }
  });

  // 3. Send welcome email
  await emailService.send({
    to: user.email,
    subject: 'Welcome!',
    template: 'welcome',
    data: { name: user.name }
  });

  // 4. Return response
  res.status(201).json({ id: user.id, email: user.email });
}

// After: each function has a single responsibility
function validateRegistrationInput(body: unknown): RegistrationInput {
  const result = RegistrationSchema.safeParse(body);
  if (!result.success) throw new ValidationError(result.error);
  return result.data;
}

async function createUserRecord(input: RegistrationInput): Promise<User> {
  const hashedPassword = await bcrypt.hash(input.password, 10);
  return db.users.create({
    data: { email: input.email, password: hashedPassword }
  });
}

async function sendWelcomeEmail(user: User): Promise<void> {
  await emailService.send({
    to: user.email,
    subject: 'Welcome!',
    template: 'welcome',
    data: { name: user.name }
  });
}

async function handleRegistration(req: Request, res: Response) {
  const input = validateRegistrationInput(req.body);
  const user = await createUserRecord(input);
  await sendWelcomeEmail(user);
  res.status(201).json({ id: user.id, email: user.email });
}
```

**规则：** 如果你无法用一句话描述一个函数而不使用"并且"，就拆分它。

### 原则 3：将条件提取为命名函数

嵌套和复杂的条件很难推理。

```typescript
// Before: cryptic conditional
if (
  user.subscription === 'premium' &&
  user.trialEndsAt &&
  new Date(user.trialEndsAt) > new Date() &&
  !user.paymentFailed
) {
  grantAccess();
}

// After: named condition with clear meaning
function hasActivePremiumAccess(user: User): boolean {
  const trialActive = user.trialEndsAt
    ? new Date(user.trialEndsAt) > new Date()
    : false;
  return (
    user.subscription === 'premium' &&
    trialActive &&
    !user.paymentFailed
  );
}

if (hasActivePremiumAccess(user)) {
  grantAccess();
}
```

**规则：** 如果一个条件需要注释才能理解，将其提取为命名函数。

### 原则 4：偏好数据变换而非命令序列

一系列的变量变化很难跟踪。链式变换显示每一步的意图。

```typescript
// Before: imperative mutation
const result = [];
for (const task of tasks) {
  if (task.completed) {
    const user = getUserById(task.userId);
    if (user && user.role === 'admin') {
      result.push({
        id: task.id,
        title: task.title,
        completedBy: user.name
      });
    }
  }
}

// After: declarative transformation
const result = tasks
  .filter(task => task.completed)
  .map(task => ({ task, user: getUserById(task.userId) }))
  .filter(({ user }) => user?.role === 'admin')
  .map(({ task, user }) => ({
    id: task.id,
    title: task.title,
    completedBy: user!.name,
  }));
```

**规则：** 如果你有一个带有过滤和变换逻辑的循环，考虑用 `.filter()`, `.map()`, `.reduce()` 替代。

### 原则 5：将魔法数字和字符串替换为命名常量

数字和字符串字面量没有传达其含义。

```typescript
// Before: magic numbers
if (responseTime > 2000) {
  showSlowConnectionWarning();
}
const expiresAt = Date.now() + 86400000;

// After: named constants
const SLOW_CONNECTION_THRESHOLD_MS = 2000;
const ONE_DAY_MS = 24 * 60 * 60 * 1000;

if (responseTime > SLOW_CONNECTION_THRESHOLD_MS) {
  showSlowConnectionWarning();
}
const expiresAt = Date.now() + ONE_DAY_MS;
```

## 简化过程

```
1. UNDERSTAND
   └── Read the code and understand what it does (all of it)

2. TEST
   └── Ensure existing tests pass (or write them first if missing)
       └── No tests = no safety net for simplification

3. IDENTIFY
   └── Find specific complexity (which principle applies?)
   └── Don't simplify what doesn't need simplifying

4. SIMPLIFY ONE THING
   └── Apply exactly one change (extract function, remove duplication, etc.)

5. VERIFY
   └── Run tests — behavior must be identical

6. REPEAT
   └── If more complexity remains, go to step 3
```

**关键规则：** 每次只做一处改变并验证。多次改变会导致行为变化，且难以判断哪里出了问题。

## 何时停止

这些是简化到位的信号：

- 函数名称清楚地传达它们做什么
- 函数可以在不读取实现的情况下理解
- 变更只需触及一个地方
- 新的 bug 很难意外引入
- 测试覆盖了完整行为

## 语言特定指导

### TypeScript / JavaScript

```typescript
// Extract complex type guards
const isCompletedTask = (task: Task): task is CompletedTask =>
  task.status === 'completed' && task.completedAt !== null;

// Use optional chaining over nested null checks
// Before
const city = user && user.address && user.address.city;
// After
const city = user?.address?.city;

// Prefer exhaustive switch with never checks
function getStatusLabel(status: TaskStatus): string {
  switch (status) {
    case 'pending': return 'Pending';
    case 'active': return 'Active';
    case 'completed': return 'Completed';
    default: {
      const _exhaustive: never = status;
      throw new Error(`Unhandled status: ${_exhaustive}`);
    }
  }
}
```

### CSS

```css
/* Before: repeated values */
.button-primary { background: #3b82f6; padding: 0.5rem 1rem; }
.button-secondary { background: #3b82f6; padding: 0.5rem 1rem; border: 1px solid #3b82f6; }
.button-danger { background: #ef4444; padding: 0.5rem 1rem; }

/* After: CSS custom properties */
:root {
  --color-primary: #3b82f6;
  --color-danger: #ef4444;
  --button-padding: 0.5rem 1rem;
}

.button { padding: var(--button-padding); }
.button-primary { background: var(--color-primary); }
.button-secondary { background: var(--color-primary); border: 1px solid var(--color-primary); }
.button-danger { background: var(--color-danger); }
```

### Python

```python
# Before: complex list manipulation
results = []
for item in data:
    if item['active']:
        processed = item['value'] * 2
        if processed > threshold:
            results.append(processed)

# After: list comprehension
results = [
    item['value'] * 2
    for item in data
    if item['active'] and item['value'] * 2 > threshold
]
```

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "我理解它，所以它不复杂" | 作者总是理解自己的代码。六个月后或对新成员来说会怎样？ |
| "我们以后再清理" | 以后永远不会到来。简化应该在发布之前或迭代中进行。 |
| "这段代码很关键，不要碰它" | 关键代码更需要可以被理解和安全修改。先写测试，然后简化。 |
| "这只是微小的重构，不需要测试" | 没有测试的重构是调试，而不是重构。 |
| "更短意味着更简单" | 单行的 JavaScript 可以极其难以理解。简单性是关于清晰度，而不是字符数。 |

## 危险信号

- 多层嵌套的条件（3 层或以上）
- 函数超过 50 行
- 相同的代码块三次或以上
- 没有注释无法理解的变量名称
- 带有"Also"、"And"或"Plus"的注释
- 功能测试之外没有单元测试
- 修改一个功能需要在 5+ 个文件中修改代码

## 验证

简化代码之后：

- [ ] 所有现有测试仍然通过
- [ ] 函数名称在不读取实现的情况下可以理解
- [ ] 没有嵌套条件超过 2 层
- [ ] 没有函数超过 50 行
- [ ] 没有魔法数字或字符串字面量
- [ ] 重复的逻辑已提取到单一位置
- [ ] 代码审查者不需要提问"这是做什么的？"
