---
name: source-driven-development
description: 在实现代码之前先获取权威文档。当使用不熟悉的库、API 或框架，或者需要验证特定技术的当前最佳实践时使用。
---

# 源码驱动开发

## 概述

在实现之前获取权威的、最新的文档。避免使用可能过时的训练数据，通过引用真实来源来确保实现的准确性。这减少了因使用不存在的 API、过时的模式，或错误地假设行为方面的 bug。

**核心原则：** 实现应该基于文档的事实，而不是对库行为的假设。当有疑问时，先查阅文档。

## 适用场景

- 使用你不熟悉的库或框架
- 使用可能已经更新的 API（路由约定、配置格式等）
- 需要验证特定行为（身份验证流程、错误处理）
- 使用任何有版本化 API 的工具

**不适用场景：** 使用你已经在代码库中见过模式并且了解其工作方式的熟悉代码时——不要为已验证的模式获取文档。

## 源码层次结构

按优先级排序：

```
1. OFFICIAL DOCUMENTATION
   └── docs.framework.dev, api.library.com
   └── Versioned docs matching your installed version

2. OFFICIAL REPOSITORY
   └── GitHub source code + README
   └── Examples and test files are often most accurate

3. OFFICIAL CHANGELOGS / MIGRATION GUIDES
   └── Check what changed between versions
   └── Especially important for major version bumps

4. REPUTABLE COMMUNITY SOURCES
   └── Official blog posts, maintainer articles
   └── Avoid: outdated blog posts, unverified StackOverflow answers

5. TRAINING DATA (lowest trust)
   └── Use only when docs aren't available
   └── Always verify against runtime behavior
```

## 工作流程

### 第一步：检测技术栈

```bash
# Check installed versions
cat package.json | grep -E '"(next|react|prisma|express)'
# or
npm list --depth=0
```

总是实现与已安装版本匹配的文档，而不是最新版本。Next.js 13 和 Next.js 15 有根本不同的路由约定。

### 第二步：获取权威文档

使用相关工具获取文档（web_fetch、搜索等）：

```
For Next.js: https://nextjs.org/docs
For Prisma:  https://www.prisma.io/docs
For Express: https://expressjs.com/en/api.html
For React:   https://react.dev
```

**获取什么：**
- 要使用的特定 API 或功能
- 配置格式和选项
- 常见的陷阱和注意事项
- 版本特定的变更（如果从较旧版本迁移）

### 第三步：实现

基于获取的文档进行实现。如果文档与你的假设有出入，遵循文档。

### 第四步：引用来源

在代码中引用获取的文档：

```typescript
// See: https://next-auth.js.org/configuration/options#session
export const authOptions: AuthOptions = {
  session: {
    strategy: 'jwt',
    maxAge: 30 * 24 * 60 * 60,  // 30 days
  },
  // ...
};
```

这对未来的维护者（包括你）很有价值——他们知道这些配置选项来自哪里，并且可以核实它们。

## 版本感知的文档查找

不同版本的相同框架行为可能完全不同：

```
Next.js 12: pages/ directory, getServerSideProps
Next.js 13: app/ directory introduced as experimental
Next.js 14: app/ directory stable, server actions
Next.js 15: React 19 support, enhanced caching

→ Always check: which version do we have?
→ Then: navigate to that version's docs
```

```bash
# Find the installed version
npm list next
# → next@14.2.3

# Navigate to versioned docs
# https://nextjs.org/docs (latest)
# For older: https://nextjs.org/docs/14 or check docs for version selectors
```

## 已获取文档的示例

### 示例 1：Next.js App Router

**任务：** 在 Next.js 中添加服务器端数据获取

**获取：** `https://nextjs.org/docs/app/building-your-application/data-fetching/fetching`

**发现：**
- 服务器组件默认异步
- 使用 `fetch()` 内置缓存，而不是 `getServerSideProps`
- `cache: 'no-store'` 用于动态数据

**基于文档的实现：**
```typescript
// Source: https://nextjs.org/docs/app/building-your-application/data-fetching/fetching
async function TaskList() {
  const tasks = await fetch('/api/tasks', { cache: 'no-store' })
    .then(r => r.json());
  return <ul>{tasks.map(t => <li key={t.id}>{t.title}</li>)}</ul>;
}
```

### 示例 2：Prisma 关系

**任务：** 在 Prisma 中查询相关数据

**获取：** `https://www.prisma.io/docs/orm/prisma-client/queries/relation-queries`

**发现：**
- 使用 `include` 进行热加载关联
- 使用 `select` 在同一查询中减少字段
- 嵌套的 `select` 用于关联的特定字段

**基于文档的实现：**
```typescript
// Source: https://www.prisma.io/docs/orm/prisma-client/queries/relation-queries
const tasks = await db.tasks.findMany({
  where: { userId },
  include: {
    labels: {
      select: { id: true, name: true, color: true },
    },
    assignee: {
      select: { id: true, name: true, email: true },
    },
  },
});
```

## 处理文档缺口

当文档不完整时：

```
1. Check the official repository examples/
   └── Real, tested code often reveals actual API signatures

2. Check the repository's test files
   └── Tests are written by maintainers and reflect intended behavior

3. Check the changelog/migration guides
   └── Often documents subtle behavior changes

4. Check GitHub Issues for the library
   └── Common problems and official responses

5. If still unclear: implement conservatively and add a TODO
   // TODO: Verify this with Prisma team — behavior unclear in v6 docs
   // Reference: https://github.com/prisma/prisma/issues/XXXXX
```

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "我了解这个框架，不需要查文档" | 框架版本变化。你了解它在 6 个月前的行为。获取当前版本的文档。 |
| "获取文档太耗时了" | 在错误的 API 上调试花费的时间是查阅文档的 10 倍。 |
| "训练数据有足够的知识" | 训练数据可能已经 1-2 年了。对快速发展的框架（Next.js、React）来说，这是一个长时间。 |
| "示例代码看起来是对的" | 示例可能是针对旧版本写的。引用与版本匹配的官方文档。 |

## 危险信号

- 调用文档中不存在的 API 方法
- 使用框架已废弃的配置格式
- 假设行为与实际不符（无文档验证）
- 通过调试工作来推断框架行为，而不是查阅文档
- 从旧博客文章复制代码，没有检查版本兼容性

## 验证

基于获取的文档实现之后：

- [ ] 已验证文档与已安装版本匹配
- [ ] 所有使用的 API 调用都存在于官方文档中
- [ ] 关键配置选项已根据文档验证
- [ ] 代码引用了文档来源（注释中的 URL）
- [ ] 框架在实现中的行为符合文档描述
