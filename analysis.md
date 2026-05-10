# Epic Stack 验证系统分析报告

## 一、验证入口共用机制

### 1.1 统一验证入口架构

Epic Stack 采用了**统一验证入口 + 分发处理器**的架构模式，所有验证流程都通过 `/verify` 路由处理。

**核心文件：**
- `app/routes/_auth/verify.tsx` - 验证页面和 Schema 定义
- `app/routes/_auth/verify.server.ts` - 验证服务器端处理逻辑

### 1.2 验证类型定义

验证系统支持四种验证类型：

```typescript
const types = ['onboarding', 'reset-password', 'change-email', '2fa'] as const
```

| 验证类型 | 用途 | 触发场景 |
|---------|------|---------|
| `onboarding` | 注册/邮箱验证 | 用户注册新账号时验证邮箱 |
| `reset-password` | 密码重置验证 | 用户忘记密码时验证身份 |
| `change-email` | 邮箱变更验证 | 用户修改邮箱时验证 |
| `2fa` | 二次验证 | 用户登录时的二次验证，或敏感操作前的重新验证 |

### 1.3 统一入口实现

**验证流程：**

1. **准备验证阶段** (`prepareVerification`)
   - 生成 TOTP 验证码
   - 创建/更新数据库中的 `Verification` 记录
   - 构建重定向 URL

   **文件位置：** `app/routes/_auth/verify.server.ts:76-112`

2. **验证页面阶段** (`/verify` 路由)
   - 统一的 OTP 输入界面
   - 根据验证类型显示不同的提示文字
   - 表单提交到统一的 `action`

3. **验证分发阶段** (`validateRequest`)
   - 统一的验证码校验
   - 根据 `type` 参数分发到对应处理器

   **分发逻辑：** `app/routes/_auth/verify.server.ts:183-199`

### 1.4 各流程的验证处理函数

每个验证类型都有独立的 `handleVerification` 函数：

| 验证类型 | 处理函数位置 | 处理逻辑 |
|---------|-------------|---------|
| `onboarding` | `app/routes/_auth/onboarding/index.server.ts:7-19` | 设置验证会话中的邮箱，重定向到 `/onboarding` |
| `reset-password` | `app/routes/_auth/reset-password.server.ts:8-34` | 查找用户，设置验证会话中的用户名，重定向到 `/reset-password` |
| `change-email` | `app/routes/settings/profile/change-email.server.tsx:14-69` | **要求近期验证**，从会话获取新邮箱，更新数据库，发送通知邮件 |
| `2fa` | `app/routes/_auth/login.server.ts:83-137` | 记录验证时间，完成登录流程 |

## 二、各流程验证机制详细分析

### 2.1 注册流程 (Onboarding)

**触发位置：** `app/routes/_auth/signup.tsx:66-71`

```typescript
const { verifyUrl, redirectTo, otp } = await prepareVerification({
    period: 10 * 60,        // 10 分钟有效期
    request,
    type: 'onboarding',
    target: email,          // target = 邮箱地址
})
```

**流程特点：**
- 用户未登录状态
- `target` 是用户提交的邮箱地址
- 验证成功后将邮箱存入 `verifySessionStorage`
- 不要求近期验证（用户还没登录）

### 2.2 重置密码流程 (Reset Password)

**触发位置：** `app/routes/_auth/forgot-password.tsx:64-69`

```typescript
const { verifyUrl, redirectTo, otp } = await prepareVerification({
    period: 10 * 60,        // 10 分钟有效期
    request,
    type: 'reset-password',
    target: usernameOrEmail, // target = 用户名或邮箱
})
```

**流程特点：**
- 用户未登录状态
- `target` 是用户名或邮箱
- 验证成功后查找用户，将用户名存入 `verifySessionStorage`
- 不要求近期验证

### 2.3 更改邮箱流程 (Change Email)

**触发位置：** `app/routes/settings/profile/change-email.tsx:73-78`

```typescript
const { otp, redirectTo, verifyUrl } = await prepareVerification({
    period: 10 * 60,        // 10 分钟有效期
    request,
    target: userId,         // target = 用户 ID
    type: 'change-email',
})
```

**流程特点：**
- 用户已登录状态
- `target` 是用户 ID（不是邮箱）
- **双重验证要求**：
  1. 访问页面时要求近期验证（`requireRecentVerification`）
  2. 验证处理时再次要求近期验证
- 新邮箱地址存储在 `verifySessionStorage` 中
- 验证成功后发送通知到旧邮箱

**特殊要求：** `app/routes/settings/profile/change-email.server.tsx:18`
```typescript
await requireRecentVerification(request)
```

## 三、验证码与二次验证的生命周期差异

### 3.1 验证码 (Email Code) 的生命周期

**数据库模型：** `prisma/schema.prisma:126-155`

```prisma
model Verification {
  id         String   @id @default(cuid())
  createdAt  DateTime @default(now())
  type       String
  target     String
  secret     String
  algorithm  String
  digits     Int
  period     Int
  charSet    String
  expiresAt  DateTime?
  @@unique([target, type])
}
```

**生命周期特征：**

| 特性 | 说明 |
|-----|------|
| 有效期 | 10 分钟 (`period: 10 * 60`) |
| 存储方式 | 数据库 `Verification` 表 |
| 过期处理 | `expiresAt` 字段标记过期时间 |
| 验证后行为 | **立即删除** |
| 唯一性约束 | `@@unique([target, type])` - 同一 target 和 type 只能有一个验证码 |

**验证后删除逻辑：** `app/routes/_auth/verify.server.ts:172-195`

```typescript
switch (submissionValue[typeQueryParam]) {
    case 'reset-password': {
        await deleteVerification()  // 验证后删除
        return handleResetPasswordVerification(...)
    }
    case 'onboarding': {
        await deleteVerification()  // 验证后删除
        return handleOnboardingVerification(...)
    }
    case 'change-email': {
        await deleteVerification()  // 验证后删除
        return handleChangeEmailVerification(...)
    }
    case '2fa': {
        // 2FA 验证后不删除！
        return handleLoginTwoFactorVerification(...)
    }
}
```

### 3.2 二次验证 (2FA) 的生命周期

**生命周期特征：**

| 特性 | 说明 |
|-----|------|
| 有效期 | **长期**（无 `expiresAt` 或 `expiresAt: null`） |
| 存储方式 | 数据库 `Verification` 表（同一模型） |
| 过期处理 | 不依赖数据库过期，而是通过会话中的 `verified-time` 控制 |
| 验证后行为 | **不删除**，保留在数据库中作为 2FA 配置 |
| 记录验证时间 | 在 `authSessionStorage` 中存储 `verified-time` |

**验证时间存储：** `app/routes/_auth/login.server.ts:101`

```typescript
authSession.set(verifiedTimeKey, Date.now())
```

**重新验证逻辑：** `app/routes/_auth/login.server.ts:139-157`

```typescript
export async function shouldRequestTwoFA(request: Request) {
    // ...
    // if it's over two hours since they last verified, we should request 2FA again
    const verifiedTime = authSession.get(verifiedTimeKey) ?? new Date(0)
    const twoHours = 1000 * 60 * 2
    return Date.now() - verifiedTime > twoHours  // 2 小时后需要重新验证
}
```

### 3.3 关键差异总结

| 对比维度 | 验证码 (onboarding/reset-password/change-email) | 二次验证 (2fa) |
|---------|---------------------------------------------|----------------|
| 数据库过期时间 | 10 分钟 (`expiresAt` 有值) | 长期有效 (`expiresAt: null` 或不存在) |
| 验证后行为 | **立即删除** 数据库记录 | **保留** 数据库记录（作为 2FA 配置） |
| 控制机制 | 依赖数据库 `expiresAt` 字段 | 依赖会话中的 `verified-time` 时间戳 |
| 重新验证频率 | 一次性使用 | 每 2 小时需要重新验证 |
| 用途 | 一次性验证（邮箱验证、重置密码等） | 持续的身份验证加强 |

## 四、敏感设置对近期验证的要求

### 4.1 什么是"近期验证"

**定义：** 用户在最近 2 小时内完成过 2FA 验证。

**检查函数：** `app/routes/_auth/login.server.ts:139-157`

```typescript
export async function shouldRequestTwoFA(request: Request) {
    // ...
    const verifiedTime = authSession.get(verifiedTimeKey) ?? new Date(0)
    const twoHours = 1000 * 60 * 2
    return Date.now() - verifiedTime > twoHours
}
```

### 4.2 要求近期验证的场景

**当前实现中：**

| 场景 | 文件位置 | 要求 |
|-----|---------|------|
| 更改邮箱 (loader) | `app/routes/settings/profile/change-email.tsx:35` | `await requireRecentVerification(request)` |
| 更改邮箱 (action) | `app/routes/settings/profile/change-email.server.tsx:18` | `await requireRecentVerification(request)` |

**`requireRecentVerification` 实现：** `app/routes/_auth/verify.server.ts:58-74`

```typescript
export async function requireRecentVerification(request: Request) {
    const userId = await requireUserId(request)
    const shouldReverify = await shouldRequestTwoFA(request)
    if (shouldReverify) {
        // 重定向到 2FA 验证页面
        const redirectUrl = getRedirectToUrl({
            request,
            target: userId,
            type: twoFAVerificationType,
            redirectTo: reqUrl.pathname + reqUrl.search,
        })
        throw await redirectWithToast(redirectUrl.toString(), {
            title: 'Please Reverify',
            description: 'Please reverify your account before proceeding',
        })
    }
}
```

### 4.3 设计逻辑分析

**为什么更改邮箱需要双重验证？**

1. **高风险操作**：更改邮箱是高敏感操作，可能导致账户被盗
2. **额外安全层**：即使攻击者获得了用户的会话，也需要 2FA 才能更改邮箱
3. **时间窗口控制**：2 小时的时间窗口平衡了安全性和用户体验

**不要求近期验证的操作：**

| 操作 | 文件位置 | 验证方式 |
|-----|---------|---------|
| 更改密码 | `app/routes/settings/profile/password.tsx` | 验证当前密码（不需要 2FA） |
| 更改头像 | 类似 | 无特殊验证要求 |
| 更改用户名 | 类似 | 无特殊验证要求 |

### 4.4 更改邮箱的完整安全流程

```
用户访问 /settings/profile/change-email
        ↓
[Loader] requireRecentVerification(request)
    ├─ 检查是否登录
    ├─ 检查是否启用 2FA
    └─ 检查最近 2 小时内是否验证过 2FA
        ├─ 已验证 → 显示更改邮箱表单
        └─ 未验证 → 重定向到 /verify?type=2fa&target=userId
                    验证成功后 redirect 回来
        ↓
用户提交新邮箱
        ↓
[Action] 发送验证邮件到新邮箱
        ↓
用户点击邮件链接或输入验证码
        ↓
[验证 Handler] requireRecentVerification(request) 再次检查
        ↓
更新数据库中的邮箱
        ↓
发送通知邮件到旧邮箱
```

## 五、架构设计总结

### 5.1 统一入口的优势

1. **代码复用**：验证 UI、验证码校验逻辑统一实现
2. **易于扩展**：新增验证类型只需添加新的 `handleVerification` 函数
3. **安全性**：统一的安全检查点，便于审计和加固
4. **用户体验**：一致的验证界面和流程

### 5.2 两种验证机制的设计哲学

| 机制 | 设计哲学 | 适用场景 |
|-----|---------|---------|
| 一次性验证码 | 短期、一次性的身份验证 | 注册、密码重置 |
| 持续 2FA | 长期、可重复使用的验证配置 | 登录保护、敏感操作保护 |

### 5.3 敏感操作的安全模型

```
┌─────────────────────────────────────────────────────┐
│                    安全层级模型                       │
├─────────────────────────────────────────────────────┤
│  层级 1: 基础认证                                    │
│  └─ 登录验证 (用户名/密码 或 OAuth)                   │
├─────────────────────────────────────────────────────┤
│  层级 2: 二次验证 (2FA)                              │
│  └─ TOTP 验证码，每 2 小时需要重新验证                 │
├─────────────────────────────────────────────────────┤
│  层级 3: 敏感操作验证 (双重保护)                      │
│  └─ 更改邮箱需要：                                   │
│     1. 已登录                                        │
│     2. 最近 2 小时内完成 2FA 验证                     │
│     3. 新邮箱的邮件验证码验证                         │
└─────────────────────────────────────────────────────┘
```
