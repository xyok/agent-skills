---
name: security-and-hardening
description: 识别并修复安全漏洞。当实现涉及认证、用户数据、外部输入或敏感操作的功能时，或在部署之前进行安全审查时使用。
---

# 安全加固

## 概述

通过在边界处防御、最小化暴露面，以及预设代码可能受到攻击者控制的输入来保护应用程序。安全性不是事后添加的——它是贯穿整个系统设计和实现的一系列决策。

**核心原则：** 信任边界。知道哪些数据是可信的（由你的系统生成），哪些是不可信的（来自用户、第三方 API 或任何外部来源）。在不可信数据进入你的系统时验证和净化它。

## 适用场景

- 实现认证或授权
- 处理用户输入
- 存储敏感数据
- 与外部 API 或服务集成
- 在部署前的安全审查

## 三层边界系统

```
┌─────────────────────────────────────────────────┐
│  Tier 1: NETWORK BOUNDARY                        │
│  - TLS/HTTPS everywhere                          │
│  - CORS configured to allowed origins only       │
│  - Rate limiting on API endpoints                │
├─────────────────────────────────────────────────┤
│  Tier 2: APPLICATION BOUNDARY                    │
│  - Authentication (who are you?)                 │
│  - Authorization (what can you do?)              │
│  - Input validation and sanitization             │
├─────────────────────────────────────────────────┤
│  Tier 3: DATA BOUNDARY                           │
│  - Principle of least privilege                  │
│  - Sensitive data encrypted at rest              │
│  - Secrets management (no secrets in code)       │
└─────────────────────────────────────────────────┘
```

## OWASP Top 10 核心缺陷

### A01: 访问控制失效

最常见的严重安全问题。确保每个受保护的资源都验证请求者被授权访问该特定资源。

```typescript
// ✗ Missing authorization check
app.delete('/api/tasks/:id', authenticate, async (req, res) => {
  await db.tasks.delete({ where: { id: req.params.id } });
  res.sendStatus(204);
});

// ✓ Verify ownership before allowing mutation
app.delete('/api/tasks/:id', authenticate, async (req, res) => {
  const task = await db.tasks.findUnique({
    where: { id: req.params.id },
    select: { userId: true },
  });

  if (!task) return res.sendStatus(404);
  if (task.userId !== req.user.id) return res.sendStatus(403);

  await db.tasks.delete({ where: { id: req.params.id } });
  res.sendStatus(204);
});
```

**模式：** 对于每个修改操作（更新、删除），验证请求者拥有或有权访问该资源，而不仅仅是资源存在。

### A02: 加密失效

- 所有 HTTP 流量使用 TLS（HTTPS）
- 使用 bcrypt/scrypt/argon2 哈希密码（不是 MD5/SHA1）
- 对静态敏感数据（SSN、信用卡）使用 AES-256 加密
- 绝不将密码、密钥或个人数据记录到日志

```typescript
// ✓ Password hashing with bcrypt
import bcrypt from 'bcrypt';

async function hashPassword(plaintext: string): Promise<string> {
  return bcrypt.hash(plaintext, 12);  // 12 rounds = good balance
}

async function verifyPassword(plaintext: string, hash: string): Promise<boolean> {
  return bcrypt.compare(plaintext, hash);
}
```

### A03: 注入

注入攻击（SQL、命令、LDAP）通过将不可信数据发送到解析器来工作。始终使用参数化查询，而不是字符串拼接。

```typescript
// ✗ SQL injection vulnerability
const result = await db.query(
  `SELECT * FROM users WHERE email = '${userInput}'`
);

// ✓ Parameterized query
const result = await db.query(
  'SELECT * FROM users WHERE email = $1',
  [userInput]
);

// ✓ Using an ORM (Prisma handles this automatically)
const result = await db.users.findUnique({
  where: { email: userInput },
});
```

```typescript
// ✗ Command injection
import { exec } from 'child_process';
exec(`convert ${userInput} output.pdf`);  // User can inject shell commands

// ✓ Use library directly, never shell out with user input
import sharp from 'sharp';
await sharp(inputPath).toFormat('pdf').toFile(outputPath);
```

### A04: 不安全的设计

安全设计原则：
- **最小权限：** 每个组件只有它需要的权限
- **默认拒绝：** 默认拒绝访问，明确授予权限
- **纵深防御：** 多层安全控制，而不是单一边界
- **失效安全：** 当组件失败时，它以安全状态失败

### A05: 安全配置错误

```typescript
// ✓ Helmet.js: security headers in Express
import helmet from 'helmet';
app.use(helmet());
// Sets: X-Content-Type-Options, X-Frame-Options, HSTS, CSP, etc.

// ✓ CORS: only allow legitimate origins
import cors from 'cors';
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') ?? ['https://yourdomain.com'],
  methods: ['GET', 'POST', 'PATCH', 'DELETE'],
  credentials: true,
}));

// ✓ Rate limiting
import rateLimit from 'express-rate-limit';
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,  // limit each IP to 100 requests per windowMs
});
app.use('/api/', limiter);
```

### A06: 易受攻击和过时的组件

```bash
# Check for known vulnerabilities in dependencies
npm audit

# Upgrade packages with vulnerabilities
npm audit fix

# For major version bumps (read changelog first)
npm audit fix --force
```

设置 Dependabot 或 Renovate 以自动在 PR 中标记漏洞（参见 `ci-cd-and-automation`）。

### A07: 认证和会话管理失效

```typescript
// ✓ Secure session configuration
import session from 'express-session';
app.use(session({
  secret: process.env.SESSION_SECRET!,  // From environment variable
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: true,     // HTTPS only
    httpOnly: true,   // Not accessible to JavaScript
    sameSite: 'lax',  // CSRF protection
    maxAge: 24 * 60 * 60 * 1000,  // 24 hours
  },
}));
```

```typescript
// ✓ JWT: validate properly
import jwt from 'jsonwebtoken';

function verifyToken(token: string): JWTPayload {
  return jwt.verify(token, process.env.JWT_SECRET!) as JWTPayload;
  // jwt.verify throws if token is invalid, expired, or tampered
}
```

**绝不：** 在前端存储的 JWT 中包含敏感数据（它们只是 base64 编码，非加密）。

### A08: 软件和数据完整性失效

- 验证 webhook 签名（Stripe、GitHub 等）
- 不使用不可信来源的 eval() 或动态代码执行
- 锁定依赖版本（`package-lock.json`）

```typescript
// ✓ Verify webhook signatures
function verifyStripeWebhook(payload: string, signature: string): Stripe.Event {
  return stripe.webhooks.constructEvent(
    payload,
    signature,
    process.env.STRIPE_WEBHOOK_SECRET!  // Validates source authenticity
  );
}
```

### A09: 安全日志记录和监控失效

```typescript
// ✓ Log security-relevant events
logger.warn('Failed login attempt', {
  email: sanitizeEmail(email),  // Sanitize before logging
  ip: req.ip,
  timestamp: new Date().toISOString(),
  // Never log: passwords, tokens, full credit card numbers
});

logger.info('Permission denied', {
  userId: req.user.id,
  resource: req.path,
  method: req.method,
});
```

### A10: 服务端请求伪造（SSRF）

```typescript
// ✗ SSRF vulnerability: user controls the URL
async function fetchExternalData(url: string) {
  return fetch(url);  // Can be http://169.254.169.254/ (AWS metadata)
}

// ✓ Allowlist external URLs
const ALLOWED_HOSTS = ['api.example.com', 'cdn.example.com'];

async function fetchExternalData(url: string) {
  const parsed = new URL(url);
  if (!ALLOWED_HOSTS.includes(parsed.hostname)) {
    throw new Error(`Host ${parsed.hostname} is not allowed`);
  }
  return fetch(url);
}
```

## 密钥管理

```
✗ Never:
  - Hardcode secrets in source code
  - Commit .env files with real values
  - Log secrets to stdout/stderr
  - Pass secrets as URL query parameters
  - Include secrets in client-side code

✓ Always:
  - Use environment variables
  - Use a secrets manager (AWS Secrets Manager, Vault, Doppler)
  - Provide .env.example with placeholder values
  - Rotate secrets regularly
  - Use different secrets for each environment
```

```bash
# .env.example (committed — shows required variables)
DATABASE_URL=postgresql://user:password@host:5432/db
NEXTAUTH_SECRET=your-secret-here
STRIPE_SECRET_KEY=sk_test_...

# .env (NOT committed — real values)
DATABASE_URL=postgresql://prod_user:real_password@host:5432/prod_db
NEXTAUTH_SECRET=actual-random-secret-32-chars-min
STRIPE_SECRET_KEY=sk_live_actual_key
```

## npm 审计分类

```
Critical → Fix immediately, don't deploy
High     → Fix before next deployment
Moderate → Fix within 30 days
Low      → Fix as part of regular maintenance
```

运行 `npm audit` 时：
1. 阅读每个漏洞及其影响
2. 检查你的代码是否实际使用了有漏洞的路径
3. 先使用 `npm audit fix`，如果不工作再使用 `--force`
4. 测试修复没有破坏任何东西

## 常见的自我安慰

| 自我安慰 | 现实 |
|---|---|
| "这只是一个内部工具，不需要安全性" | 内部工具也持有真实数据，被真实用户使用，并且可能连接到外部系统。 |
| "我们以后再加安全措施" | 事后添加安全性的代价是十倍。从一开始就建立安全设计。 |
| "我信任这个用户" | 认证≠授权。即使是真实的已认证用户也不能访问其他人的数据。 |
| "密码是哈希的，所以我们是安全的" | 哈希密码只是安全的一小部分。SQL 注入、CSRF、XSS 等同样重要。 |
| "npm 审计中只是低漏洞" | 低漏洞会被利用。定期清理，不要让它们积累。 |

## 危险信号

- 未使用参数化查询的数据库操作
- 未检查所有权的删除/更新端点
- 存储在代码中的密钥（不是环境变量）
- 没有速率限制的 API 端点
- 从用户输入构建的 URL，没有验证
- 密码使用弱哈希（MD5、SHA1）
- npm 审计中的关键或高漏洞
- 前端代码中有密钥（JavaScript 会被用户看到）

## 验证

实现安全相关功能之后：

- [ ] 所有数据库查询使用参数化查询或 ORM
- [ ] 所有修改端点检查资源所有权
- [ ] 没有密钥硬编码在源代码中
- [ ] 已运行 `npm audit`，已解决关键/高漏洞
- [ ] 认证在变更操作中被强制执行
- [ ] 用户输入在边界处验证和净化
- [ ] 没有敏感数据被记录到日志
- [ ] 速率限制适用于认证端点
