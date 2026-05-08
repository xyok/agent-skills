# 性能检查清单

Web 应用性能快速参考检查清单。与 `performance-optimization` 技能配合使用。

## 目录

- [Core Web Vitals 目标值](#core-web-vitals-目标值)
- [TTFB 诊断](#ttfb-诊断)
- [前端检查清单](#前端检查清单)
- [后端检查清单](#后端检查清单)
- [测量命令](#测量命令)
- [常见反模式](#常见反模式)

## Core Web Vitals 目标值

| 指标 | 良好 | 需要改进 | 较差 |
|------|------|----------|------|
| LCP（最大内容绘制） | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| INP（下次绘制的交互时间） | ≤ 200ms | ≤ 500ms | > 500ms |
| CLS（累积布局偏移） | ≤ 0.1 | ≤ 0.25 | > 0.25 |

## TTFB 诊断

当 TTFB 较慢（> 800ms）时，在 DevTools Network 瀑布图中逐项检查：

- [ ] **DNS 解析**慢 → 为已知来源添加 `<link rel="dns-prefetch">` 或 `<link rel="preconnect">`
- [ ] **TCP/TLS 握手**慢 → 启用 HTTP/2，考虑边缘部署，验证 keep-alive
- [ ] **服务器处理**慢 → 分析后端性能，检查慢查询，添加缓存

## 前端检查清单

### 图片
- [ ] 图片使用现代格式（WebP、AVIF）
- [ ] 图片使用响应式尺寸（`srcset` 和 `sizes`）
- [ ] 图片和 `<source>` 元素有明确的 `width` 和 `height`（防止艺术指导中的 CLS）
- [ ] 首屏以下的图片使用 `loading="lazy"` 和 `decoding="async"`
- [ ] 英雄图/LCP 图片使用 `fetchpriority="high"` 且不延迟加载

### JavaScript
- [ ] 打包体积在 200KB gzip 以内（初始加载）
- [ ] 对路由和重功能使用动态 `import()` 进行代码分割
- [ ] 启用 tree shaking（确认依赖项提供 ESM 并标记 `sideEffects: false`）
- [ ] `<head>` 中无阻塞性 JavaScript（使用 `defer` 或 `async`）
- [ ] 将繁重计算交由 Web Workers 处理（如适用）
- [ ] 对使用相同 props 频繁重渲染的昂贵组件使用 `React.memo()`
- [ ] 仅在性能分析显示有收益时使用 `useMemo()` / `useCallback()`
- [ ] 拆分长任务（> 50ms）以保持主线程可用——这是改善 INP 的主要手段
- [ ] 在长时间运行的循环中使用 `yieldToMain` 模式，让输入事件在各块之间执行
- [ ] 在条件允许时使用现代调度 API：`scheduler.yield()`（首选）、带优先级的 `scheduler.postTask()`、`isInputPending()` 按需让出
- [ ] 使用 `requestIdleCallback` 处理可推迟的非紧急工作（分析上报、预取、预热）
- [ ] 将非关键工作移出事件处理器（如分析、日志），避免延迟交互响应
- [ ] 第三方脚本使用 `async` / `defer` 加载，审计其体积，对重型脚本（聊天组件、嵌入内容）使用外观模式

### CSS
- [ ] 关键 CSS 内联或预加载
- [ ] 非关键样式无渲染阻塞
- [ ] 生产环境无 CSS-in-JS 运行时开销（使用提取方式）

### 字体
- [ ] 限制在 2–3 个字体族，每个字体族 2–3 个字重（每增加一个字重就多一个请求）
- [ ] 仅使用 WOFF2 格式（最小、通用支持——跳过 WOFF/TTF/EOT）
- [ ] 尽可能自托管（第三方字体 CDN 会增加 DNS + TCP + TLS 往返）
- [ ] 预加载 LCP 关键字体：`<link rel="preload" as="font" type="font/woff2" crossorigin>`
- [ ] 使用 `font-display: swap`（非关键字体用 `optional`）以避免 FOIT 阻塞渲染
- [ ] 通过 `unicode-range` 子集化，仅向每个页面发送其所需的字形
- [ ] 当需要多种字重/样式时考虑可变字体（一个文件替代多个）
- [ ] 使用 `size-adjust`、`ascent-override`、`descent-override` 调整备用字体指标，减少字体切换时的 CLS
- [ ] 在引入任何自定义字体之前先考虑系统字体栈

### 网络
- [ ] 静态资源使用长 `max-age` + 内容哈希进行缓存
- [ ] API 响应在适当时进行缓存（`Cache-Control`）
- [ ] 启用 HTTP/2 或 HTTP/3
- [ ] 对已知来源使用 `<link rel="preconnect">` 预连接
- [ ] 对关键非图片资源使用 `fetchpriority`（如关键 `<link rel="preload">`、首屏 `<script>`）——不仅限于 `<img>`
- [ ] 无不必要的重定向

### 渲染
- [ ] 无布局抖动（强制同步布局）
- [ ] 动画使用 `transform` 和 `opacity`（GPU 加速）
- [ ] 长列表使用虚拟化（如 `react-window`）
- [ ] 无不必要的全页面重渲染
- [ ] 屏幕外区域使用 `content-visibility: auto` 配合 `contain-intrinsic-size`，跳过不可见区域的布局/绘制
- [ ] 无 `unload` 事件处理器，HTML 响应不使用 `Cache-Control: no-store`——保持 back/forward cache（bfcache）资格

## 后端检查清单

### 数据库
- [ ] 无 N+1 查询模式（使用预加载/联表查询）
- [ ] 查询有适当的索引
- [ ] 列表接口分页（绝不使用 `SELECT * FROM table`）
- [ ] 已配置连接池
- [ ] 已启用慢查询日志

### API
- [ ] 响应时间 < 200ms（p95）
- [ ] 请求处理器中无同步的繁重计算
- [ ] 使用批量操作代替循环调用单个接口
- [ ] 响应压缩（gzip/brotli）
- [ ] 合理的缓存策略（内存缓存、Redis、CDN）

### 基础设施
- [ ] 静态资源使用 CDN
- [ ] 服务器靠近用户（或边缘部署）
- [ ] 已配置水平扩展（如需要）
- [ ] 负载均衡器有健康检查接口

## 测量命令

### INP 现场数据与 DevTools 工作流

1. **先看现场数据** — 在优化之前，先从 [CrUX Vis](https://developer.chrome.com/docs/crux/vis) 或你的 RUM 工具获取真实用户的 INP 数据
2. **识别慢交互** — 打开 DevTools → Performance 面板 → 在交互过程中录制；寻找由点击/按键触发的长任务
3. **在中端 Android 设备上测试** — INP 问题通常只在较慢的设备上出现；使用真实设备或 DevTools CPU 降速（4×–6× 减速）

```bash
# Lighthouse CLI
npx lighthouse https://localhost:3000 --output json --output-path ./report.json

# Bundle analysis
npx webpack-bundle-analyzer stats.json
# or for Vite:
npx vite-bundle-visualizer

# Check bundle size
npx bundlesize

# Web Vitals in code
import { onLCP, onINP, onCLS } from 'web-vitals';
onLCP(console.log);
onINP(console.log);
onCLS(console.log);

# INP with interaction-level detail (attribution build)
import { onINP } from 'web-vitals/attribution';
onINP(({ value, attribution }) => {
  const { interactionTarget, inputDelay, processingDuration, presentationDelay } = attribution;
  console.log({ value, interactionTarget, inputDelay, processingDuration, presentationDelay });
});
```

## 常见反模式

| 反模式 | 影响 | 修复方案 |
|--------|------|----------|
| N+1 查询 | 数据库负载线性增长 | 使用联表查询、includes 或批量加载 |
| 无限制查询 | 内存耗尽、超时 | 始终分页，添加 LIMIT |
| 缺少索引 | 随数据增长读取变慢 | 为过滤/排序列添加索引 |
| 布局抖动 | 卡顿、掉帧 | 批量 DOM 读取，再批量写入 |
| 未优化图片 | LCP 慢，浪费带宽 | 使用 WebP、响应式尺寸、懒加载 |
| 大型打包文件 | 可交互时间慢 | 代码分割、tree shaking、审计依赖 |
| 阻塞主线程 | INP 差，界面无响应 | 使用 `scheduler.yield()` / `yieldToMain` 拆分长任务，交由 Web Workers 处理 |
| 内存泄漏 | 内存持续增长，最终崩溃 | 清理监听器、定时器、引用 |
