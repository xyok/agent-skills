---
name: debugging-and-error-recovery
description: 系统性地诊断和修复错误。当遇到 bug、失败的测试、未预期的行为或错误消息时使用。适用于不清楚立即解决方案的情况。
---

# 调试与错误恢复

## 概述

通过系统性调查诊断并修复错误，而不是试错式的猜测。调试是一个假设-验证循环：提出一个理论，通过证据测试它，根据你所学到的内容进行修正。好的调试比随机代码变更更快——它阻止了几层表象之下的问题。

## 适用场景

- 一个测试失败，原因不明显
- 功能在运行时以意外方式出现故障
- 出现错误消息，但来源不清楚
- 修复一个地方破坏了另一个地方
- 在调查之前不清楚需要更改什么

## 停止-排查规则

**在有理解之前不要更改代码。**

这是最重要的调试规则。随机的代码修改：
- 引入新 bug，同时掩盖原始问题
- 不系统地创建难以归因的差异
- 将调查延长数倍于必要时间

如果你不能解释 _为什么_ 一个变更会修复 bug，则它不是一个修复——它是一个猜测，以后会咬你一口。

## 分类检查单

在修改任何代码之前，按以下顺序进行调查：

```
1. REPRODUCE IT
   └── Can you trigger the bug consistently?
       └── If not, gather more data before attempting a fix

2. READ THE ERROR
   ├── Full error message and stack trace (not just the first line)
   ├── File name and line number where it originated
   └── Error type or code (e.g., TypeError, ECONNREFUSED, 404)

3. UNDERSTAND THE EXPECTED BEHAVIOR
   └── What should have happened? What happened instead?

4. FIND THE DELTA
   └── What changed recently? (git log, dependencies, config)

5. FORM A HYPOTHESIS
   └── "I think the bug is caused by X because Y"

6. TEST THE HYPOTHESIS
   ├── Add a log/breakpoint to confirm your theory
   └── Do NOT change logic yet — just observe

7. FIX
   └── Only now change code, targeting the confirmed root cause

8. VERIFY
   ├── Bug is gone
   ├── Tests pass
   └── Related behavior is still correct
```

## 完整读取错误消息

最常见的调试错误：只读取错误的第一行。

```
✗ Reading only: "TypeError: Cannot read properties of undefined"

✓ Reading the full stack trace:
  TypeError: Cannot read properties of undefined (reading 'email')
    at UserService.getEmail (src/services/user.service.ts:47:28)
    at TaskController.assignTask (src/controllers/task.controller.ts:89:32)
    at Router.handle (node_modules/express/lib/router/index.js:284:22)
  
  → This tells you: user is undefined at line 47
  → Look at how user gets its value at line 47
  → Check what calls UserService.getEmail (task.controller.ts:89)
  → Trace why user could be undefined there
```

栈跟踪是一张映射。从 _你的_ 代码中的最顶端帧开始（跳过 `node_modules`）。

## 特定错误模式

### 未定义 / 空值错误

```
TypeError: Cannot read properties of undefined (reading 'X')
TypeError: Cannot read properties of null (reading 'X')
```

调查：
```typescript
// Add logging to find where null enters
console.log('user at step 1:', user);
// Is user expected to always exist here?
// Where does user get its value?
// Is there a code path that doesn't set it?
```

常见原因：
- 异步操作在使用前未等待完成
- 可选链访问（`obj?.prop`）传播了意外的 undefined
- 数组查找（`.find()`）找不到匹配项，返回 undefined
- 数据库返回 null 而不是对象

### Promise / 异步错误

```
UnhandledPromiseRejection: ...
Error: Promise rejected with reason X
```

调查：
```typescript
// Check: is every async call awaited?
// Bad
function loadUser() {
  db.users.findUnique({ ... }); // Not awaited!
}

// Good
async function loadUser() {
  const user = await db.users.findUnique({ ... });
  return user;
}

// Check: are errors caught at the right level?
// Check: is the Promise chain complete?
```

### 网络/API 错误

```
Error: ECONNREFUSED
Error: 401 Unauthorized
Error: 404 Not Found
Error: CORS error
```

调查：
```bash
# 1. Is the service running?
curl http://localhost:3000/health

# 2. Is the URL correct?
console.log('Fetching:', url);  // Log the actual URL being called

# 3. Is the auth header being sent?
# Log the full request headers

# 4. For CORS: check the server allows this origin
# Check Access-Control-Allow-Origin header in response
```

### 类型错误（TypeScript）

```
Type 'X' is not assignable to type 'Y'
Property 'X' does not exist on type 'Y'
```

调查：
```typescript
// Read the full error message — TypeScript is usually precise
// Find where the type mismatch occurs
// Hover over variables to see inferred types
// Check if a runtime cast (as Type) is masking the real issue

// Don't just cast away the error:
// ✗ const user = getUser() as User;
// ✓ Fix the actual type mismatch or add proper validation
```

### 失败的测试

```
Expected: X
Received: Y
```

调查：
```typescript
// 1. Run only the failing test to isolate it
npm test -- --testNamePattern "the failing test name"

// 2. Read what was expected vs what was received
// 3. Trace where the actual value comes from
// 4. Check if test setup/teardown is correct
// 5. Is the test actually testing the right thing?

// Add targeted logging:
console.log('Value before assertion:', actualValue);
```

### 数据库错误

```
PrismaClientKnownRequestError: Unique constraint violated
PrismaClientKnownRequestError: Record to update not found
ERROR: column "X" does not exist
```

调查：
```bash
# 1. Is the database schema up to date?
npx prisma migrate status

# 2. Is the data in the expected state?
# Use Prisma Studio or psql to inspect the actual data

# 3. For unique constraint violations:
# Check if you're accidentally creating duplicates
# Check if a previous test left conflicting data

# 4. For missing column errors:
# Check if migration was run
npx prisma migrate dev
```

## 二分法技术

对于难以定位的 bug，使用二分法：

```
Bug reproduced → Add a check halfway through the code

Case A: bug appears before the midpoint
→ Focus on the first half, repeat

Case B: bug appears after the midpoint  
→ Focus on the second half, repeat

Continue until you've isolated the exact line
```

这对以下情况有效：
- 数据在某处变得不正确，但不知道在哪里
- 大型函数中的意外行为
- 多步流程中的错误

## 添加临时调试输出

有策略地添加日志以验证假设：

```typescript
// Add logging at key points to trace data flow
function processPayment(order: Order) {
  console.log('DEBUG processPayment input:', JSON.stringify(order, null, 2));

  const total = calculateTotal(order);
  console.log('DEBUG calculated total:', total);

  const result = chargeCard(total, order.paymentMethod);
  console.log('DEBUG charge result:', result);

  return result;
}
```

**重要：** 在修复之后删除或注释掉调试日志。提交调试日志是一种代码气味。

## 使用 git 查找回归

如果最近某次变更之前一切都正常：

```bash
# See recent commits
git log --oneline -20

# Check what changed in a specific commit
git show <commit-sha>

# Find when a specific line last changed
git blame src/services/user.service.ts

# Binary search for when the bug was introduced
git bisect start
git bisect bad HEAD
git bisect good <last-known-good-sha>
# git will checkout commits for you to test
# Run: git bisect good or git bisect bad after each test
git bisect reset  # when done
```

## 隔离测试问题

当测试失败时，首先确认是真实的 bug 还是测试本身有问题：

```bash
# Run a single test file
npx vitest src/services/user.service.test.ts

# Run a specific test case by name
npx vitest --testNamePattern="should return user by email"

# Run tests in watch mode (re-runs on file change)
npx vitest --watch

# Check if test is flaky (run it 10 times)
for i in {1..10}; do npx vitest run --reporter=verbose --testPathPattern="specific.test"; done
```

## 何时升级

如果你已经花了 30 分钟没有找到根本原因：

1. **彻底记录你所知道的**：错误消息、栈跟踪、观察到的行为、已排除的原因
2. **检查你是否在调查正确的层级**：错误是在这个函数中，还是在其调用者中？
3. **考虑一个完全不同的假设**：你最初的假设可能是错误的
4. **寻求帮助**：向他人展示完整的错误和上下文

"我只是随机改变东西，直到它工作"——这是停止的信号。

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "我就试试这个，看看会怎样" | 没有假设的代码更改会引入新的 bug 并延长调试时间。 |
| "错误消息不是很有用" | 它几乎总是有用——你只需要读完整的内容，包括栈跟踪。 |
| "这一定是框架的 bug" | 99% 的情况下是你的代码。先用尽自己的假设。 |
| "它在我的机器上运行" | 环境差异（版本、配置、数据）才是关键。隔离差异。 |
| "重启修复了它，所以没事了" | 这是一个间歇性 bug，你只是把它推迟了。找出根本原因。 |

## 危险信号

- 在不理解的情况下更改代码（随机修复）
- 只读取错误消息的第一行而不是完整的栈跟踪
- 通过类型断言（`as Type`）绕过类型错误
- 更改工作行为以使测试通过
- 使用 `try/catch` 吞掉错误而不处理它
- 跨多个文件更改许多内容而不是隔离原因

## 验证

调试之后：

- [ ] 已确认根本原因（不是随机修复）
- [ ] 可以解释为什么该修复有效
- [ ] 已删除调试日志
- [ ] 回归测试已添加或存在以捕获此 bug
- [ ] 所有其他测试仍然通过
- [ ] 该修复没有引入新的 bug（已通过检查相关代码）
