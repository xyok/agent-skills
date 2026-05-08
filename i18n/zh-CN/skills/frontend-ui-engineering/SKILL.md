---
name: frontend-ui-engineering
description: 构建可访问、响应式和可维护的 UI。适用于创建新组件或页面，以及修改现有 UI 代码时使用。
---

# 前端 UI 工程

## 概述

构建适用于真实用户的用户界面——包括键盘导航用户、屏幕阅读器用户、移动设备用户和慢速网络用户。前端工程比视觉实现更多：它关乎可访问性、性能、状态管理和可维护性。

**关于"AI 美学"的警告：** AI 生成的 UI 往往有视觉上令人印象深刻但体验糟糕的可辨识特征：过多的渐变/动画、随机的玻璃态效果、充斥着不相关图标的界面、空洞的营销词语，以及不实际可用的"漂亮"设计。构建用于完成工作的 UI，而不是为了在截图中看起来好看。

## 适用场景

- 构建新的组件或页面
- 修改现有 UI 代码
- 修复 UI 的 bug 或可访问性问题
- 响应式设计工作
- 状态管理决策

## 组件架构

### 组件大小规则

```
< 50 lines     → Single focused component, likely no breakdown needed
50–150 lines   → Consider if it's doing one thing
150–300 lines  → Break into sub-components
300+ lines     → Definitely needs to be broken up
```

### 关注点分离

```
Page Component
  └── Orchestrates data fetching, layout, state
      └── Container Component
            └── Handles business logic, passes data to UI
                └── UI Component
                      └── Pure rendering, no business logic
                          └── Primitive Component
                                └── Button, Input, etc.
```

```typescript
// ✗ Mixed concerns: fetching + business logic + rendering in one component
function TaskPage() {
  const [tasks, setTasks] = useState([]);
  useEffect(() => {
    fetch('/api/tasks').then(r => r.json()).then(setTasks);
  }, []);
  const completedTasks = tasks.filter(t => t.status === 'done');
  const pendingTasks = tasks.filter(t => t.status === 'pending');
  return (
    <div>
      {/* 200 lines of rendering */}
    </div>
  );
}

// ✓ Separated concerns
function TaskPage() {
  const { tasks, isLoading, error } = useTaskQuery();
  if (isLoading) return <TaskListSkeleton />;
  if (error) return <ErrorMessage error={error} />;
  return <TaskList tasks={tasks} />;
}

function TaskList({ tasks }: { tasks: Task[] }) {
  const { completed, pending } = groupTasksByStatus(tasks);
  return (
    <>
      <TaskSection title="Pending" tasks={pending} />
      <TaskSection title="Completed" tasks={completed} />
    </>
  );
}
```

### 服务端组件优先（Next.js App Router）

```typescript
// Default: Server Component (no 'use client')
// - Renders on server, sends HTML to client
// - Can directly access DB, filesystem, env vars
// - Cannot use useState, useEffect, event handlers
async function TaskList() {
  const tasks = await db.tasks.findMany(); // Direct DB access
  return <ul>{tasks.map(t => <TaskItem key={t.id} task={t} />)}</ul>;
}

// Only add 'use client' when needed:
// - useState or useReducer
// - useEffect
// - Browser APIs (localStorage, window)
// - Event handlers that need interactivity
'use client';
function TaskCheckbox({ taskId, completed }: TaskCheckboxProps) {
  const [isChecked, setIsChecked] = useState(completed);
  return (
    <input
      type="checkbox"
      checked={isChecked}
      onChange={() => setIsChecked(!isChecked)}
    />
  );
}
```

## 可访问性（WCAG 2.1 AA）

可访问性不是可选的。它让残障用户能够使用你的产品，在许多地区是法律要求，并且改善所有用户的体验。

### 语义 HTML

```html
<!-- ✗ Div soup: no semantic meaning -->
<div class="header">
  <div class="nav">
    <div onclick="navigate('/tasks')">Tasks</div>
  </div>
</div>
<div class="main">
  <div class="article">...</div>
</div>

<!-- ✓ Semantic HTML: structure is meaningful -->
<header>
  <nav>
    <a href="/tasks">Tasks</a>
  </nav>
</header>
<main>
  <article>...</article>
</main>
```

### 焦点管理

```typescript
// Interactive elements must be focusable and have visible focus styles
// ✗ Removing focus outline without replacement
button:focus { outline: none; }

// ✓ Custom focus style that's visible
button:focus-visible {
  outline: 2px solid var(--color-focus);
  outline-offset: 2px;
}
```

```typescript
// Manage focus for dynamic content (modals, drawers, alerts)
function Modal({ isOpen, onClose, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (isOpen) {
      // Move focus into modal when opened
      modalRef.current?.focus();
    }
  }, [isOpen]);

  return isOpen ? (
    <div
      ref={modalRef}
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
      tabIndex={-1}
    >
      {children}
    </div>
  ) : null;
}
```

### ARIA 标签

```typescript
// ✗ Icon buttons with no accessible name
<button onClick={handleDelete}>
  <TrashIcon />
</button>

// ✓ Icon button with accessible label
<button onClick={handleDelete} aria-label={`Delete task: ${task.title}`}>
  <TrashIcon aria-hidden="true" />
</button>

// ✓ Or use visually hidden text
<button onClick={handleDelete}>
  <TrashIcon aria-hidden="true" />
  <span className="sr-only">Delete task: {task.title}</span>
</button>
```

```typescript
// Announce dynamic changes to screen readers
function TaskStatus({ status }: { status: string }) {
  return (
    <div aria-live="polite" aria-atomic="true">
      {status === 'saving' && 'Saving...'}
      {status === 'saved' && 'Saved!'}
      {status === 'error' && 'Failed to save. Please try again.'}
    </div>
  );
}
```

### 表单可访问性

```typescript
// ✗ Input with no label
<input type="email" placeholder="email@example.com" />

// ✓ Input with associated label
<label htmlFor="email">Email address</label>
<input
  id="email"
  type="email"
  aria-describedby="email-hint"
  aria-invalid={!!errors.email}
  aria-errormessage="email-error"
/>
<p id="email-hint" className="text-sm text-gray-500">
  We'll never share your email.
</p>
{errors.email && (
  <p id="email-error" role="alert" className="text-red-600">
    {errors.email.message}
  </p>
)}
```

### 颜色对比

```
Text on background: minimum 4.5:1 contrast ratio (WCAG AA)
Large text (18px+): minimum 3:1 contrast ratio
UI components: minimum 3:1 contrast ratio against adjacent colors
```

不要仅靠颜色传达信息：

```typescript
// ✗ Color-only indicator: colorblind users can't distinguish
<span style={{ color: isOverdue ? 'red' : 'green' }}>
  {task.title}
</span>

// ✓ Color + icon/text: multiple signals
<span
  className={isOverdue ? 'text-red-600' : 'text-green-600'}
  aria-label={isOverdue ? 'Overdue' : 'On track'}
>
  {isOverdue ? '⚠️ ' : '✓ '}
  {task.title}
</span>
```

## 响应式设计

### 移动优先策略

```css
/* Base styles apply to mobile */
.task-list {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

/* Enhance for larger screens */
@media (min-width: 768px) {
  .task-list {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (min-width: 1024px) {
  .task-list {
    grid-template-columns: repeat(3, 1fr);
  }
}
```

### 触摸目标大小

```css
/* Minimum 44x44px for touch targets (iOS HIG, WCAG 2.5.5) */
.button, .checkbox, .radio, a {
  min-height: 44px;
  min-width: 44px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}
```

## 状态管理

### 状态应该放在哪里

```
Local state (useState)
  └── UI state that only affects one component
      e.g., dropdown open/closed, hover state

Lifted state (parent component)
  └── State shared between sibling components
      e.g., form values affecting preview

Server state (React Query / SWR)
  └── Data fetched from API
      e.g., task list, user profile

URL state (searchParams)
  └── State that should be shareable via URL
      e.g., filters, pagination, active tab

Global state (Context / Zustand)
  └── Truly global state (use sparingly)
      e.g., auth user, theme, notifications
```

### 避免不必要的客户端状态

```typescript
// ✗ Derived state in useState (gets out of sync)
const [tasks, setTasks] = useState(allTasks);
const [completedTasks, setCompletedTasks] = useState(
  allTasks.filter(t => t.completed)
);

// ✓ Derive from source of truth
const [tasks, setTasks] = useState(allTasks);
const completedTasks = tasks.filter(t => t.completed); // Always in sync
```

## 加载状态和错误状态

每个异步操作都需要三个状态：

```typescript
function TaskList() {
  const { data: tasks, isLoading, error } = useTaskQuery();

  // 1. Loading state
  if (isLoading) {
    return <TaskListSkeleton />;  // Not a spinner — skeleton matches layout
  }

  // 2. Error state
  if (error) {
    return (
      <ErrorMessage
        title="Failed to load tasks"
        message="Check your connection and try again."
        onRetry={() => mutate()}  // Allow retry
      />
    );
  }

  // 3. Empty state
  if (tasks.length === 0) {
    return (
      <EmptyState
        title="No tasks yet"
        action={<CreateTaskButton />}
      />
    );
  }

  // 4. Success state
  return <ul>{tasks.map(t => <TaskItem key={t.id} task={t} />)}</ul>;
}
```

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "可访问性是后期优化" | 事后添加可访问性代价昂贵。从一开始就构建它——使用正确的语义 HTML。 |
| "看起来好看就够了" | 视觉上令人印象深刻≠功能良好。用户需要完成任务，而不是欣赏渐变。 |
| "移动设备以后再说" | 50% 以上的网络流量来自移动设备。先为移动端设计。 |
| "每件事都使用全局状态" | 全局状态会创建隐藏的依赖。尽量让状态保持本地。 |
| "我的屏幕上可以工作" | 用键盘测试。用屏幕阅读器测试。在限速网络上测试。 |

## 危险信号

- 移除焦点轮廓（`outline: none`），没有替代
- 交互元素没有可访问的标签
- 仅用颜色传达信息
- 对一切使用 `<div>` 而不是语义 HTML
- 没有加载状态、错误状态或空状态
- 客户端组件没有理由（不需要任何客户端功能）
- 带有深色背景的白色文本不通过对比检查
- 触摸目标小于 44x44px

## 验证

构建或修改 UI 之后：

- [ ] **可访问性：** 可以只用键盘操作整个 UI
- [ ] **可访问性：** 所有图像都有 alt 文本；所有图标按钮都有 aria-label
- [ ] **可访问性：** 文本通过 4.5:1 颜色对比检查
- [ ] **响应式：** 在 320px、768px 和 1280px 宽度下测试
- [ ] **状态：** 加载、错误和空状态都已实现
- [ ] **语义：** HTML 使用适当的元素（`button`、`nav`、`main` 等）
- [ ] **性能：** 客户端组件仅用于需要客户端功能的内容
- [ ] **可用性：** 动态内容变化通过 ARIA live regions 宣告
