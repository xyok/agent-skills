---
name: test-driven-development
description: 通过先写测试来构建正确的代码。适用于实现新功能、修复 bug，或重构现有代码时使用。对于 bug，使用"先证明再修复"的模式。
---

# 测试驱动开发

## 概述

在实现之前编写测试。这强制清晰地思考代码应该做什么，在早期捕获 bug，并建立一个可以安全重构的安全网。TDD 不仅仅是关于测试——它是一种产生更模块化、更易于测试的设计的设计实践。

**先证明 bug（Prove-It 模式）：** 对于 bug 修复，在修复之前编写一个重现 bug 的失败测试。这证明 bug 是真实的，防止回归，并验证修复有效。

## 适用场景

- 实现带有可测试验收标准的新功能
- 修复 bug（先证明再修复）
- 重构现有代码（测试防止回归）
- 构建公共 API 或服务层

**不适用场景：** 纯粹的 UI 外观/CSS 变更（改用视觉测试），快速原型（稍后测试），或用 E2E 测试覆盖更充分的集成点。

## TDD 循环

```
1. RED
   └── Write a failing test
   └── Run it to confirm it fails (don't skip this step)

2. GREEN
   └── Write the minimum code to make the test pass
   └── Resist the urge to write more than needed

3. REFACTOR
   └── Clean up code while keeping tests green
   └── No new behavior — only improve quality

4. REPEAT
   └── Back to step 1 for the next behavior
```

**关键规则：** 绿灯之前不要重构。红灯之前不要实现。

## 先证明 Bug（Prove-It 模式）

修复 bug 之前，写一个证明 bug 存在的测试：

```typescript
// Step 1: Write a test that reproduces the bug
it('should not allow deleting another user\'s task', async () => {
  const user1Task = await createTask({ userId: 'user-1' });

  // This should return 403, but the bug is it returns 204
  const response = await request(app)
    .delete(`/api/tasks/${user1Task.id}`)
    .set('Authorization', 'Bearer user-2-token');

  // This will FAIL first (bug exists — response is 204 instead of 403)
  expect(response.status).toBe(403);
});

// Step 2: Run the test, confirm it fails with the bug
// Step 3: Fix the bug
// Step 4: Run the test again — it should now pass
```

**为什么这很重要：**
- 证明 bug 是真实的，而不只是假设的
- 防止你修复了不同的问题却认为你修复了 bug
- 添加一个防止回归的测试
- 验证修复实际上解决了问题

## 测试类型

### 单元测试

测试单个函数或类，与依赖项隔离：

```typescript
// Tests the service function in isolation
describe('TaskService.createTask', () => {
  it('creates a task with the correct owner', async () => {
    const result = await taskService.createTask(
      { title: 'Test task' },
      'user-123'
    );

    expect(result.title).toBe('Test task');
    expect(result.userId).toBe('user-123');
    expect(result.id).toBeDefined();
  });

  it('throws ValidationError for empty title', async () => {
    await expect(
      taskService.createTask({ title: '' }, 'user-123')
    ).rejects.toThrow(ValidationError);
  });
});
```

**何时使用：** 服务层、工具函数、纯计算逻辑。

### 集成测试

测试多个组件一起工作：

```typescript
// Tests the API endpoint + service + database together
describe('POST /api/tasks', () => {
  it('creates a task and returns 201', async () => {
    const response = await request(app)
      .post('/api/tasks')
      .set('Authorization', `Bearer ${userToken}`)
      .send({ title: 'New task', priority: 'high' });

    expect(response.status).toBe(201);
    expect(response.body.title).toBe('New task');
    expect(response.body.id).toBeDefined();

    // Verify it was actually persisted
    const dbTask = await db.tasks.findUnique({
      where: { id: response.body.id }
    });
    expect(dbTask).not.toBeNull();
  });

  it('returns 401 without authentication', async () => {
    const response = await request(app)
      .post('/api/tasks')
      .send({ title: 'New task' });

    expect(response.status).toBe(401);
  });
});
```

**何时使用：** API 端点测试，涉及数据库的工作流程。

### E2E 测试

测试完整的用户工作流程，通过 UI：

```typescript
// Playwright: tests the full user journey in a browser
test('user can create and complete a task', async ({ page }) => {
  await page.goto('/tasks');

  // Create a task
  await page.getByRole('button', { name: 'New task' }).click();
  await page.getByLabel('Task title').fill('Buy groceries');
  await page.getByRole('button', { name: 'Create' }).click();

  // Verify it appears
  await expect(page.getByText('Buy groceries')).toBeVisible();

  // Complete it
  await page.getByLabel('Complete: Buy groceries').click();

  // Verify it moves to completed
  await expect(page.getByTestId('completed-tasks')).toContainText('Buy groceries');
});
```

**何时使用：** 关键用户工作流程，跨越多个 UI 组件的集成点。

## 测试金字塔

```
         /\
        /E2E\         ← Few (5-10): critical user flows only
       /------\
      /  Integr.\     ← More (20-50): API endpoints, key workflows
     /------------\
    /  Unit tests  \  ← Many (100+): functions, services, logic
   /----------------\
```

**规则：** 大多数测试应该是单元测试（快速、便宜），少数是集成测试（慢些），更少是 E2E（最慢、最贵维护）。

## 测试的内容

```
✓ Happy path (common correct usage)
✓ Error cases (invalid input, missing data, unauthorized)
✓ Boundary conditions (empty arrays, zero, max values, null)
✓ State transitions (pending → active → completed → archived)
✓ Security (unauthorized access, ownership verification)

✗ Implementation details (don't test HOW, test WHAT)
✗ External library internals (assume they work)
✗ Things that don't change behavior (logging, formatting)
```

## 测试设置模式

### 数据库测试（Jest + Prisma）

```typescript
// test/setup.ts
import { db } from '../src/db';

beforeAll(async () => {
  // Reset database to clean state before test suite
  await db.$executeRaw`TRUNCATE tasks, users, labels CASCADE`;
});

afterAll(async () => {
  await db.$disconnect();
});
```

```typescript
// Factories for test data
export async function createTestUser(overrides = {}) {
  return db.users.create({
    data: {
      id: faker.string.uuid(),
      email: faker.internet.email(),
      name: faker.person.fullName(),
      ...overrides,
    },
  });
}

export async function createTestTask(userId: string, overrides = {}) {
  return db.tasks.create({
    data: {
      id: faker.string.uuid(),
      title: faker.lorem.words(3),
      userId,
      ...overrides,
    },
  });
}
```

### 模拟外部依赖

```typescript
// Mock external services, not your own code
jest.mock('../src/services/email.service', () => ({
  emailService: {
    sendWelcomeEmail: jest.fn().mockResolvedValue({ success: true }),
  },
}));

// In tests:
it('sends welcome email on user creation', async () => {
  await createUser({ email: 'test@example.com' });

  expect(emailService.sendWelcomeEmail).toHaveBeenCalledWith({
    to: 'test@example.com',
    // ... expected parameters
  });
});
```

## 测试命名

好的测试名称读起来像规格说明：

```typescript
// ✗ Vague
it('works correctly', ...)
it('should pass', ...)
it('taskService test', ...)

// ✓ Behavior description
it('returns 404 when task does not exist', ...)
it('prevents users from viewing other users\' tasks', ...)
it('includes completed tasks when showCompleted is true', ...)
```

**格式：** `[subject] [condition] [expected behavior]`

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "我以后再写测试" | 以后就是永远不。没有测试，重构会破坏东西，你不知道在哪里。 |
| "代码太简单了，不需要测试" | 简单的代码也会出错，尤其是在集成的时候。边界情况在简单函数中出现。 |
| "写测试比实现花费更长时间" | 测试节省了调试时间。一个有覆盖的小时比一个没有覆盖的小时要好得多。 |
| "我先修复 bug，然后写测试" | 先证明 bug——否则你可能修复了不同的东西，而认为你修复了 bug。 |
| "E2E 测试覆盖了所有内容" | E2E 太慢、太脆弱，无法覆盖每个边界情况。需要单元+集成。 |

## 危险信号

- 没有测试的新功能（或只有不测试行为的表面测试）
- 测试测试实现细节而不是行为（当内部实现改变时会断裂）
- 跳过失败状态的测试（`it.skip`）
- 由于外部因素（时序、随机性）而不稳定的测试
- 测试在对代码库不了解的情况下无法独立运行
- 没有首先证明 bug 就修复 bug

## 验证

实现功能或修复 bug 之后：

- [ ] 新行为有单元测试
- [ ] API 端点有集成测试（happy path + 授权 + 错误情况）
- [ ] Bug 修复有先失败后通过的回归测试
- [ ] 测试名称描述了行为而不是实现
- [ ] 所有测试独立运行（没有测试之间的依赖）
- [ ] 在 CI 中没有不稳定的测试
- [ ] 通过 `npm test` 运行的测试套件通过
