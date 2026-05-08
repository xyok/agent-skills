# 安全检查清单

Web 应用安全快速参考。与 `security-and-hardening` 技能配合使用。

## 目录

- [提交前检查](#提交前检查)
- [认证](#认证)
- [授权](#授权)
- [输入验证](#输入验证)
- [安全响应头](#安全响应头)
- [CORS 配置](#cors-配置)
- [数据保护](#数据保护)
- [依赖安全](#依赖安全)
- [错误处理](#错误处理)
- [OWASP Top 10 快速参考](#owasp-top-10-快速参考)

## 提交前检查

- [ ] 代码中无密钥（`git diff --cached | grep -i "password\|secret\|api_key\|token"`）
- [ ] `.gitignore` 已覆盖：`.env`、`.env.local`、`*.pem`、`*.key`
- [ ] `.env.example` 使用占位值（而非真实密钥）

## 认证

- [ ] 密码使用 bcrypt（≥12 轮）、scrypt 或 argon2 进行哈希
- [ ] Session cookie：`httpOnly`、`secure`、`sameSite: 'lax'`
- [ ] 已配置 Session 过期（合理的 max-age）
- [ ] 登录接口启用速率限制（每 15 分钟 ≤10 次尝试）
- [ ] 密码重置令牌：有时间限制（≤1 小时），且只能使用一次
- [ ] 多次失败后账号锁定（可选，附带通知）
- [ ] 敏感操作支持 MFA（可选但推荐）

## 授权

- [ ] 每个受保护的接口都检查了认证
- [ ] 每次资源访问都检查了所有权/角色（防止 IDOR）
- [ ] 管理员接口需要管理员角色验证
- [ ] API 密钥的权限范围限定为最小必要权限
- [ ] JWT 令牌经过验证（签名、过期时间、颁发者）

## 输入验证

- [ ] 所有用户输入在系统边界处（API 路由、表单处理器）经过验证
- [ ] 验证使用允许列表（而非拒绝列表）
- [ ] 字符串长度受到约束（最小/最大）
- [ ] 数值范围经过验证
- [ ] 使用适当的库验证邮件、URL 和日期格式
- [ ] 文件上传：类型受限、大小受限、内容经过验证
- [ ] SQL 查询参数化（禁止字符串拼接）
- [ ] HTML 输出经过编码（使用框架自动转义）
- [ ] 重定向前验证 URL（防止开放重定向）

## 安全响应头

```
Content-Security-Policy: default-src 'self'; script-src 'self'
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 0  (disabled, rely on CSP)
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

## CORS 配置

```typescript
// Restrictive (recommended)
cors({
  origin: ['https://yourdomain.com', 'https://app.yourdomain.com'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
})

// NEVER use in production:
cors({ origin: '*' })  // Allows any origin
```

## 数据保护

- [ ] API 响应中已排除敏感字段（`passwordHash`、`resetToken` 等）
- [ ] 敏感数据未被记录到日志中（密码、令牌、完整信用卡号）
- [ ] PII 静态加密（如法规要求）
- [ ] 所有外部通信使用 HTTPS
- [ ] 数据库备份已加密

## 依赖安全

```bash
# Audit dependencies
npm audit

# Fix automatically where possible
npm audit fix

# Check for critical vulnerabilities
npm audit --audit-level=critical

# Keep dependencies updated
npx npm-check-updates
```

## 错误处理

```typescript
// Production: generic error, no internals
res.status(500).json({
  error: { code: 'INTERNAL_ERROR', message: 'Something went wrong' }
});

// NEVER in production:
res.status(500).json({
  error: err.message,
  stack: err.stack,         // Exposes internals
  query: err.sql,           // Exposes database details
});
```

## OWASP Top 10 快速参考

| # | 漏洞 | 防御措施 |
|---|------|----------|
| 1 | 访问控制失效 | 在每个接口检查授权，验证所有权 |
| 2 | 加密失败 | HTTPS、强哈希算法、密钥不入代码 |
| 3 | 注入 | 参数化查询、输入验证 |
| 4 | 不安全设计 | 威胁建模、规格驱动开发 |
| 5 | 安全配置错误 | 安全响应头、最小权限、审计依赖 |
| 6 | 易受攻击的组件 | `npm audit`、保持依赖更新、最少依赖 |
| 7 | 认证失败 | 强密码、速率限制、session 管理 |
| 8 | 数据完整性失败 | 验证更新/依赖、签名制品 |
| 9 | 日志记录失败 | 记录安全事件，不记录密钥 |
| 10 | SSRF | 验证/允许列表 URL，限制出站请求 |
