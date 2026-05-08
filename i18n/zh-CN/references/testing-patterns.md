# 测试模式参考

跨技术栈的常用测试模式快速参考。与 `test-driven-development` 技能配合使用。

## 目录

- [测试结构（Arrange-Act-Assert）](#测试结构arrange-act-assert)
- [测试命名约定](#测试命名约定)
- [常用断言](#常用断言)
- [Mock 模式](#mock-模式)
- [React/组件测试](#react组件测试)
- [API / 集成测试](#api--集成测试)
- [E2E 测试（Playwright）](#e2e-测试playwright)
- [测试反模式](#测试反模式)

## 测试结构（Arrange-Act-Assert）

```typescript
it('describes expected behavior', () => {
  // Arrange: Set up test data and preconditions
  const input = { title: 'Test Task', priority: 'high' };

  // Act: Perform the action being tested
  const result = createTask(input);

  // Assert: Verify the outcome
  expect(result.title).toBe('Test Task');
  expect(result.priority).toBe('high');
  expect(result.status).toBe('pending');
});
```

## 测试命名约定

```typescript
// Pattern: [unit] [expected behavior] [condition]
describe('TaskService.createTask', () => {
  it('creates a task with default pending status', () => {});
  it('throws ValidationError when title is empty', () => {});
  it('trims whitespace from title', () => {});
  it('generates a unique ID for each task', () => {});
});
```

## 常用断言

```typescript
// Equality
expect(result).toBe(expected);           // Strict equality (===)
expect(result).toEqual(expected);        // Deep equality (objects/arrays)
expect(result).toStrictEqual(expected);  // Deep equality + type matching

// Truthiness
expect(result).toBeTruthy();
expect(result).toBeFalsy();
expect(result).toBeNull();
expect(result).toBeDefined();
expect(result).toBeUndefined();

// Numbers
expect(result).toBeGreaterThan(5);
expect(result).toBeLessThanOrEqual(10);
expect(result).toBeCloseTo(0.3, 5);      // Floating point

// Strings
expect(result).toMatch(/pattern/);
expect(result).toContain('substring');

// Arrays / Objects
expect(array).toContain(item);
expect(array).toHaveLength(3);
expect(object).toHaveProperty('key', 'value');

// Errors
expect(() => fn()).toThrow();
expect(() => fn()).toThrow(ValidationError);
expect(() => fn()).toThrow('specific message');

// Async
await expect(asyncFn()).resolves.toBe(value);
await expect(asyncFn()).rejects.toThrow(Error);
```

## Mock 模式

### Mock 函数

```typescript
const mockFn = jest.fn();
mockFn.mockReturnValue(42);
mockFn.mockResolvedValue({ data: 'test' });
mockFn.mockImplementation((x) => x * 2);

expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledWith('arg1', 'arg2');
expect(mockFn).toHaveBeenCalledTimes(3);
```

### Mock 模块

```typescript
// Mock an entire module
jest.mock('./database', () => ({
  query: jest.fn().mockResolvedValue([{ id: 1, title: 'Test' }]),
}));

// Mock specific exports
jest.mock('./utils', () => ({
  ...jest.requireActual('./utils'),
  generateId: jest.fn().mockReturnValue('test-id'),
}));
```

### 仅在边界处 Mock

```
应该 Mock 的：                 不应该 Mock 的：
├── 数据库调用                 ├── 内部工具函数
├── HTTP 请求                  ├── 业务逻辑
├── 文件系统操作               ├── 数据转换
├── 外部 API 调用              ├── 验证函数
└── 时间/日期（按需）          └── 纯函数
```

## React/组件测试

```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';

describe('TaskForm', () => {
  it('submits the form with entered data', async () => {
    const onSubmit = jest.fn();
    render(<TaskForm onSubmit={onSubmit} />);

    // Find elements by accessible role/label (not test IDs)
    await screen.findByRole('textbox', { name: /title/i });
    fireEvent.change(screen.getByRole('textbox', { name: /title/i }), {
      target: { value: 'New Task' },
    });
    fireEvent.click(screen.getByRole('button', { name: /create/i }));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({ title: 'New Task' });
    });
  });

  it('shows validation error for empty title', async () => {
    render(<TaskForm onSubmit={jest.fn()} />);

    fireEvent.click(screen.getByRole('button', { name: /create/i }));

    expect(await screen.findByText(/title is required/i)).toBeInTheDocument();
  });
});
```

## API / 集成测试

```typescript
import request from 'supertest';
import { app } from '../src/app';

describe('POST /api/tasks', () => {
  it('creates a task and returns 201', async () => {
    const response = await request(app)
      .post('/api/tasks')
      .send({ title: 'Test Task' })
      .set('Authorization', `Bearer ${testToken}`)
      .expect(201);

    expect(response.body).toMatchObject({
      id: expect.any(String),
      title: 'Test Task',
      status: 'pending',
    });
  });

  it('returns 422 for invalid input', async () => {
    const response = await request(app)
      .post('/api/tasks')
      .send({ title: '' })
      .set('Authorization', `Bearer ${testToken}`)
      .expect(422);

    expect(response.body.error.code).toBe('VALIDATION_ERROR');
  });

  it('returns 401 without authentication', async () => {
    await request(app)
      .post('/api/tasks')
      .send({ title: 'Test' })
      .expect(401);
  });
});
```

## E2E 测试（Playwright）

```typescript
import { test, expect } from '@playwright/test';

test('user can create and complete a task', async ({ page }) => {
  // Navigate and authenticate
  await page.goto('/');
  await page.fill('[name="email"]', 'test@example.com');
  await page.fill('[name="password"]', 'testpass123');
  await page.click('button:has-text("Log in")');

  // Create a task
  await page.click('button:has-text("New Task")');
  await page.fill('[name="title"]', 'Buy groceries');
  await page.click('button:has-text("Create")');

  // Verify task appears
  await expect(page.locator('text=Buy groceries')).toBeVisible();

  // Complete the task
  await page.click('[aria-label="Complete Buy groceries"]');
  await expect(page.locator('text=Buy groceries')).toHaveCSS(
    'text-decoration-line', 'line-through'
  );
});
```

## 测试反模式

| 反模式 | 问题 | 更好的做法 |
|--------|------|------------|
| 测试实现细节 | 重构后测试失败 | 测试输入/输出 |
| 对所有内容做快照 | 没人审查快照差异 | 断言具体值 |
| 共享可变状态 | 测试之间相互污染 | 每个测试独立设置/清理 |
| 测试第三方代码 | 浪费时间，不是你的 Bug | Mock 边界 |
| 跳过测试以通过 CI | 掩盖真实 Bug | 修复或删除测试 |
| 永久使用 `test.skip` | 死代码 | 删除或修复 |
| 断言过于宽泛 | 无法捕获回归 | 保持具体 |
| 无异步错误处理 | 错误被吞掉，产生假通过 | 始终 `await` 异步测试 |
