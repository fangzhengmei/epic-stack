# Epic Stack 邮件发送链路分析

## 1. 邮件发送路径

### 1.1 使用的服务

Epic Stack 使用 **Resend** 作为邮件发送服务，**不存在 SMTP 路径**。

### 1.2 SMTP 路径不存在的代码证据

#### 证据 1：package.json 中无 SMTP 相关依赖

`package.json` 的 `dependencies` 和 `devDependencies` 中：
- 没有 `nodemailer`
- 没有 `smtp-server`
- 没有任何 SMTP 相关的包

唯一的邮件相关依赖是 `@react-email/components`，用于渲染邮件模板，而非发送。

#### 证据 2：代码中无 SMTP 配置

搜索结果显示，所有 `transport` / `transports` 关键词都与 WebAuthn/Passkey 相关，与邮件无关：
- `passkey.test.ts:12`: `transport: 'usb'`（Passkey 认证器传输方式）
- `schema.prisma:183`: `transports String?`（存储 Passkey 认证器元数据）
- `webauthn/registration.ts:119`: `transports: credential.transports?.join(',')`（Passkey 相关）

#### 证据 3：决策文档明确排除 SMTP

`docs/decisions/002-email-service.md:9-18`：

```markdown
Packages like `nodemailer` make it quite easy to send your
own emails through your own mailserver or a third party's SMTP server as well.

Unfortunately,
[deliverability will suffer if you're not using a service](...).
The TL;DR is you either dedicate your company's complete resources to "play the
game" of email deliverability, or you use a service that does. Otherwise, your
emails won't reliably make it through spam filters...

## Decision

We will use a service for sending email.
```

**决策结论**：明确选择使用邮件服务（Mailgun，后来迁移到 Resend），而非 SMTP。

### 1.3 Resend 与 SMTP 的关系

| 特性 | Resend | SMTP |
|------|--------|------|
| 代码中存在 | ✅ 是 | ❌ 否 |
| 调用方式 | REST API (`fetch`) | 不存在 |
| 依赖 | 无额外依赖 | `nodemailer` 等（不存在） |
| 开发模式 | MSW 拦截 HTTP 请求 | 不存在 |
| 可扩展性 | 可替换为其他邮件服务 API | 不存在 |

**关系总结**：
- Resend 是**唯一的**邮件发送机制
- 没有 SMTP 回退或备选路径
- 代码通过 `fetch` 直接调用 Resend REST API (`https://api.resend.com/emails`)
- 开发模式下使用 MSW (Mock Service Worker) **拦截 HTTP 请求**，而非模拟 SMTP

### 1.4 核心实现

邮件发送核心逻辑位于 `app/utils/email.server.ts`：

```typescript
// 使用 fetch 直接调用 Resend REST API
const response = await fetch('https://api.resend.com/emails', {
    method: 'POST',
    body: JSON.stringify(email),
    headers: {
        Authorization: `Bearer ${process.env.RESEND_API_KEY}`,
        'Content-Type': 'application/json',
    },
})
```
[app/utils/email.server.ts:55-62](app/utils/email.server.ts#L55-L62)

---

## 2. 发送成功时返回的数据结构（重要更正）

### 2.1 两个成功返回路径

`sendEmail` 函数有 **两个不同的成功返回路径**，返回的数据结构**不一致**：

#### 路径 A：未配置 Resend 时的"伪装成功"

```typescript
// email.server.ts:43-53
if (!process.env.RESEND_API_KEY && !process.env.MOCKS) {
    console.error(`RESEND_API_KEY not set and we're not in mocks mode.`)
    return {
        status: 'success',
        data: { id: 'mocked' },  // 直接包含 id
    } as const
}
```

**返回结构：**
```typescript
{
    status: 'success',
    data: { id: 'mocked' }  // response.data.id 可用
}
```

#### 路径 B：实际发送成功

```typescript
// email.server.ts:64-70
const data = await response.json()
const parsedData = resendSuccessSchema.safeParse(data)  // Zod safeParse

if (response.ok && parsedData.success) {
    return {
        status: 'success',
        data: parsedData,  // ⚠️ 这里返回的是 safeParse 的结果！
    } as const
}
```

**返回结构：**

`safeParse` 的返回类型是 `SafeParseReturnType`，成功时结构为：
```typescript
{
    status: 'success',
    data: {
        success: true,
        data: { id: 'actual-resend-id' }  // 注意：id 在 response.data.data.id
    }
}
```

### 2.2 结构不一致的影响

| 访问方式 | 路径 A（伪装成功） | 路径 B（实际成功） |
|---------|-------------------|-------------------|
| `response.status` | `'success'` | `'success'` |
| `response.data.id` | `'mocked'` ✓ | `undefined` ✗ |
| `response.data.data.id` | `undefined` ✗ | `'actual-resend-id'` ✓ |

### 2.3 当前调用方的处理

幸运的是，**所有调用方都只检查 `response.status`，不使用 `response.data`**：

```typescript
// signup.tsx:79-90 和 forgot-password.tsx:79-86
if (response.status === 'success') {
    return redirect(redirectTo.toString())  // 只检查 status，不使用 data
} else {
    return data({...}, { status: 500 })
}
```

因此，这种结构不一致**不会导致运行时错误**，但仍是潜在的维护陷阱。

---

## 3. 注册验证邮件流程

### 3.1 入口文件

注册流程位于 `app/routes/_auth/signup.tsx`。

### 3.2 完整流程

```
用户输入邮箱 → 检查邮箱是否已注册 → 创建验证记录 → 发送验证邮件 → 重定向到验证页面
                                    ↓
                              prepareVerification()
                                    ↓
                              生成 OTP 验证码
                                    ↓
                              存储到数据库 verification 表
                                    ↓
                              构造验证链接 (verifyUrl)
                                    ↓
                              sendEmail()
                                    ↓
                              检查发送结果
```

### 3.3 关键代码

```typescript
// 1. 准备验证（生成 OTP、存储到数据库）
const { verifyUrl, redirectTo, otp } = await prepareVerification({
    period: 10 * 60,  // 10 分钟有效期
    request,
    type: 'onboarding',
    target: email,
})

// 2. 发送邮件
const response = await sendEmail({
    to: email,
    subject: `Welcome to Epic Notes!`,
    react: <SignupEmail onboardingUrl={verifyUrl.toString()} otp={otp} />,
})

// 3. 处理发送结果
if (response.status === 'success') {
    return redirect(redirectTo.toString())
} else {
    return data(
        {
            result: submission.reply({ formErrors: [response.error.message] }),
        },
        { status: 500 },
    )
}
```
[app/routes/_auth/signup.tsx:66-90](app/routes/_auth/signup.tsx#L66-L90)

---

## 4. 密码重置邮件流程与账号存在性泄露（重要更正）

### 4.1 流程架构

密码重置涉及两个阶段：
1. **发起阶段** (`forgot-password.tsx`)：用户输入邮箱/用户名，请求发送重置邮件
2. **验证阶段** (`reset-password.server.ts`)：用户输入收到的验证码，完成验证

### 4.2 发起阶段 - 账号存在性泄露（确认）

**`forgot-password.tsx:30-47` 的代码：**

```typescript
const submission = await parseWithZod(formData, {
    schema: ForgotPasswordSchema.superRefine(async (data, ctx) => {
        const user = await prisma.user.findFirst({
            where: {
                OR: [
                    { email: data.usernameOrEmail },
                    { username: data.usernameOrEmail },
                ],
            },
            select: { id: true },
        })
        if (!user) {
            ctx.addIssue({
                path: ['usernameOrEmail'],
                code: z.ZodIssueCode.custom,
                message: 'No user exists with this username or email',  // ⚠️ 直接泄露！
            })
            return
        }
    }),
    async: true,
})
```

**问题确认：**
- 当用户输入不存在的邮箱/用户名时，返回明确的错误消息：`"No user exists with this username or email"`
- 攻击者可以通过枚举输入来判断哪些账号已注册

### 4.3 验证阶段 - 有隐私保护

**`reset-password.server.ts:18-25` 的代码：**

```typescript
// we don't want to say the user is not found if the email is not found
// because that would allow an attacker to check if an email is registered
if (!user) {
    return data(
        { result: submission.reply({ fieldErrors: { code: ['Invalid code'] } }) },
        { status: 400 },
    )
}
```

**对比分析：**

| 阶段 | 文件 | 用户不存在时的处理 | 保护措施 |
|------|------|-------------------|----------|
| **发起阶段** | `forgot-password.tsx` | 返回 `"No user exists with this username or email"` | ❌ 无保护，直接泄露 |
| **验证阶段** | `reset-password.server.ts` | 返回 `"Invalid code"` | ✅ 有保护，注释明确说明 |

### 4.4 安全风险

这是一个**不一致的安全设计**：
- 验证阶段考虑了隐私保护
- 但发起阶段直接泄露账号存在性

**攻击场景：**
1. 攻击者访问 `/forgot-password` 页面
2. 输入 `admin@company.com`
3. 如果返回 `"No user exists..."`，说明该邮箱未注册
4. 如果返回成功（重定向到 `/verify`），说明该邮箱已注册
5. 攻击者可以继续枚举常见用户名/邮箱

---

## 5. 未启用 Resend 时的失败路径分析（重要更正）

### 5.1 条件逻辑详解

`email.server.ts:43` 的条件：

```typescript
if (!process.env.RESEND_API_KEY && !process.env.MOCKS)
```

**关键点：这是 `AND` 关系，不是 `OR`！**

意味着：**只有当既没有 API key，也没有启用 MOCKS 时**，才会进入这个分支。

### 5.2 不同场景分析

#### 场景 1：生产环境（配置正确）
- `RESEND_API_KEY=xxx`（已设置）
- `MOCKS` 未设置
- 条件：`!true && !false` = `false && true` = `false`
- **结果**：正常调用 Resend API，实际发送邮件

#### 场景 2：开发模式（`npm run dev`）
- `package.json:14`: `"dev": "cross-env NODE_ENV=development MOCKS=true node index.ts"`
- `MOCKS=true`
- 条件：`!xxx && !true` = `xxx && false` = `false`
- **结果**：不会进入这个分支！MSW mock 服务器拦截请求

#### 场景 3：开发模式下的 Mock 实现

当 `MOCKS=true` 时：
1. `index.ts:19-21` 导入 `tests/mocks/index.ts`
2. MSW (Mock Service Worker) 启动，拦截 HTTP 请求
3. `tests/mocks/resend.ts:8-21` 定义了拦截处理：

```typescript
http.post(`https://api.resend.com/emails`, async ({ request }) => {
    requireHeader(request.headers, 'Authorization')
    const body = await request.json()
    console.info('🔶 mocked email contents:', body)
    const email = await writeEmail(body)  // 写入文件供检查
    
    return json({
        id: faker.string.uuid(),
        from: email.from,
        to: email.to,
        created_at: new Date().toISOString(),
    })
})
```

**开发模式下：**
- 邮件不会实际发送到 Resend
- 邮件内容会打印到控制台 (`🔶 mocked email contents:`)
- 同时写入文件（通过 `writeEmail`）供测试检查
- 返回伪造的成功响应

#### 场景 4：生产环境（忘记配置 API key）⚠️
- `RESEND_API_KEY` 未设置
- `MOCKS` 未设置
- 条件：`!false && !false` = `true && true` = `true`
- **结果**：进入"伪装成功"分支！

### 5.3 "伪装成功"的实际行为

```typescript
// email.server.ts:43-53
if (!process.env.RESEND_API_KEY && !process.env.MOCKS) {
    console.error(`RESEND_API_KEY not set and we're not in mocks mode.`)
    console.error(`To send emails, set the RESEND_API_KEY environment variable.`)
    console.error(`Would have sent the following email:`, JSON.stringify(email))
    return {
        status: 'success',      // ⚠️ 假装成功！
        data: { id: 'mocked' },
    } as const
}
```

**对用户注册的影响：**

```
1. 用户输入邮箱，点击注册
2. prepareVerification() 创建验证记录（OTP 有效期 10 分钟）
3. sendEmail() 返回 status: 'success'（但邮件未发送）
4. 用户被重定向到 /verify 页面
5. 用户等待邮件... 永远等不到
6. 10 分钟后验证记录过期
7. 用户无法完成注册流程
```

### 5.4 这是有意的设计选择

代码中的注释：
```typescript
// feel free to remove this condition once you've set up resend
```

**设计意图：**
- 让开发者在未配置 Resend 的情况下也能"体验"完整流程
- 避免因为缺少邮件配置而阻塞开发

**风险：**
- 生产环境如果忘记配置 `RESEND_API_KEY`，会静默失败
- 错误只打印到控制台，用户看不到任何提示
- 用户流程看似正常，但实际上无法完成注册

### 5.5 场景总结表

| 场景 | RESEND_API_KEY | MOCKS | 条件结果 | 实际发送邮件？ | 调用方感知 |
|------|----------------|-------|----------|---------------|-----------|
| 生产正确配置 | 有 | 无 | `false` | ✅ 是 | 正常 |
| 开发模式 | 任意 | `true` | `false` | ❌ 否（MSW 拦截） | 正常（但邮件内容在控制台） |
| **生产未配置** | **无** | **无** | **`true`** | ❌ 否 | **误以为成功** |

---

## 6. 发送失败场景对照矩阵（重要新增）

### 6.1 Resend 错误类型与代码处理

#### Resend 可能的错误类型（基于 API 规范推测）

| 错误类型 | 场景示例 | HTTP 状态码 |
|---------|---------|-------------|
| **Suppression** | 邮箱在抑制列表中（bounce、complaint、spam） | 400 / 403 |
| **Invalid Parameter** | 邮箱格式错误、缺少必要字段 | 400 |
| **Forbidden** | API key 无效或权限不足 | 403 |
| **Not Found** | 资源不存在 | 404 |
| **Too Many Requests** | 超过速率限制 | 429 |
| **Internal Server Error** | Resend 服务内部错误 | 500 |

#### 代码中的错误处理

`email.server.ts:5-17` 定义的错误 Schema：

```typescript
const resendErrorSchema = z.union([
    z.object({
        name: z.string(),        // 如 "invalid_parameter"
        message: z.string(),     // 详细错误信息
        statusCode: z.number(),  // HTTP 状态码
    }),
    z.object({
        name: z.literal('UnknownError'),
        message: z.literal('Unknown Error'),
        statusCode: z.literal(500),
        cause: z.any(),
    }),
])
```

**处理逻辑**：
1. 如果 Resend 返回的 JSON 能被 `resendErrorSchema` 解析，返回 `{ status: 'error', error: parseResult.data }`
2. 否则，返回 `{ status: 'error', error: { name: 'UnknownError', message: 'Unknown Error', statusCode: 500 } }`

**关键问题**：代码**不区分** suppression、bounce、complaint 等错误类型，只是把 `response.error.message` 直接显示给用户。

### 6.2 各页面在失败场景下的详细对照

#### 执行顺序分析

所有发送邮件的页面都遵循相同的执行顺序：

```
1. 表单验证（检查必填字段、格式等）
2. 业务检查（用户是否存在、邮箱是否已注册等）
3. prepareVerification() → 创建/更新 verification 记录（UPSERT）
4. sendEmail() → 发送邮件
5. 检查 response.status
   - success: 重定向
   - error: 返回 formErrors，HTTP 500
```

**关键点**：`prepareVerification` 在 `sendEmail` **之前**执行，且使用 `upsert`（不存在则创建，存在则更新）。

#### 对照矩阵

| 场景 | 注册页 (signup.tsx) | 找回密码页 (forgot-password.tsx) | 验证页 (verify.server.ts) |
|------|---------------------|----------------------------------|---------------------------|
| **是否发送邮件** | ✅ 是（`sendEmail` 在 action 中） | ✅ 是（`sendEmail` 在 action 中） | ❌ 否（只验证，不发送） |
| **prepareVerification 时机** | sendEmail 之前 | sendEmail 之前 | 不适用 |
| **邮件发送成功** | 重定向 `/verify` | 重定向 `/verify` | 不适用 |
| **邮件发送失败（Resend 返回错误）** | HTTP 500，显示 `response.error.message` | HTTP 500，显示 `response.error.message` | 不适用 |
| **邮件发送失败时的数据库变化** | verification 记录**已创建**（prepareVerification 在先） | verification 记录**已创建**（prepareVerification 在先） | 不适用 |
| **用户不存在时的处理** | 提前返回 `"A user already exists with this email"`（邮箱已存在时） | 提前返回 `"No user exists with this username or email"`（用户不存在时） | 返回 `"Invalid code"`（不泄露存在性） |
| **用户不存在时的 HTTP 状态码** | 400 | 400 | 400 |
| **验证成功后的数据库变化** | 不适用（验证页处理） | 不适用（验证页处理） | **删除** verification 记录 |
| **验证失败后的数据库变化** | 不适用 | 不适用 | **保留** verification 记录 |

### 6.3 发送失败场景的详细分析

#### 场景 1：Suppression / Bounce / Complaint（邮箱在抑制列表中）

**Resend 行为**：返回错误响应，HTTP 状态码通常为 400 或 403。

**代码处理流程**：

```
用户输入邮箱 → 表单验证通过 → 业务检查通过 → prepareVerification() 创建记录
                                                                 ↓
                                                          sendEmail()
                                                                 ↓
                                                    Resend 返回错误响应
                                                                 ↓
                                              sendEmail 返回 status: 'error'
                                                                 ↓
                              signup.tsx / forgot-password.tsx 检查 response.status
                                                                 ↓
                                        返回 formErrors: [response.error.message]
                                                                 ↓
                                                    HTTP 状态码：500
```

**结果**：
- ✅ 用户看到错误消息（`response.error.message` 的具体内容取决于 Resend）
- ⚠️ verification 记录**已创建**在数据库中（10 分钟后过期）
- ❌ 用户无法完成注册/重置流程

#### 场景 2：伪装成功（未配置 RESEND_API_KEY 且非 MOCKS 模式）

**行为**：`sendEmail` 返回 `status: 'success'`，但邮件未实际发送。

**处理流程**：

```
用户输入邮箱 → 表单验证通过 → 业务检查通过 → prepareVerification() 创建记录
                                                                 ↓
                                                          sendEmail()
                                                                 ↓
                                                    返回 status: 'success'（伪装）
                                                                 ↓
                              signup.tsx / forgot-password.tsx 检查 response.status
                                                                 ↓
                                                   重定向到 /verify 页面
```

**结果**：
- ❌ 用户**看不到任何错误**
- ❌ verification 记录**已创建**
- ❌ 用户被重定向到验证页面，但永远收不到邮件
- ⚠️ 错误只打印到**服务器控制台**，用户无感知

#### 场景 3：验证失败（用户输入错误的验证码）

**注意**：验证页**不发送邮件**，只处理用户输入的验证码。

**处理流程**：

```
用户输入验证码 → validateRequest()
                         ↓
                  isCodeValid() 检查
                         ↓
              验证码无效或 verification 记录不存在
                         ↓
              返回 fieldErrors: { code: ['Invalid code'] }
                         ↓
                    HTTP 状态码：400
```

**结果**：
- ✅ 用户看到 `"Invalid code"` 错误
- ⚠️ verification 记录**保留**（用户可以重试）
- ⚠️ **不泄露**用户是否存在（统一返回 "Invalid code"）

### 6.4 错误消息显示机制

所有页面都使用 Conform 表单系统处理错误：

**注册页和找回密码页**：
```typescript
// 发送失败时
return data(
    { result: submission.reply({ formErrors: [response.error.message] }) },
    { status: 500 },
)
```

**验证页**：
```typescript
// 验证失败时
ctx.addIssue({
    path: ['code'],
    code: z.ZodIssueCode.custom,
    message: `Invalid code`,
})
```

**前端展示**：
```typescript
// 表单级错误
<ErrorList errors={form.errors} id={form.errorId} />

// 字段级错误
errors={fields.usernameOrEmail.errors}  // 或 fields.code.errors
```

### 6.5 数据库验证记录的生命周期

| 操作 | 时间点 | 数据库操作 | 记录状态 |
|------|--------|-----------|---------|
| 用户提交注册/找回密码 | sendEmail 之前 | `prisma.verification.upsert()` | ✅ 已创建 |
| 邮件发送成功 | 重定向后 | 无操作 | ⏳ 等待用户验证 |
| 邮件发送失败 | 立即 | 无操作 | ⏳ 保留（10 分钟后过期） |
| 用户验证成功 | 验证页 | `prisma.verification.delete()` | ❌ 已删除 |
| 用户验证失败 | 验证页 | 无操作 | ⏳ 保留（可重试） |
| 超过有效期 | 10 分钟后 | 无操作（由应用逻辑判断） | ⏳ 过期（查询时忽略） |

**关键发现**：
- 没有数据库事务，`prepareVerification` 和 `sendEmail` 是两个独立操作
- 邮件发送失败时，**不会回滚**已创建的 verification 记录
- 验证记录的过期由应用逻辑控制（`expiresAt` 字段），而非数据库 TTL

---

## 7. 收件箱抑制（Suppression）对用户注册的影响

### 7.1 什么是收件箱抑制

邮件服务提供商（如 Resend）会维护一个**抑制列表（Suppression List）**，包含以下类型的邮箱：
- **Bounces（退信）**：邮箱不存在、邮箱已满、被拒收等
- **Complaints（投诉）**：用户点击了"这是垃圾邮件"
- **Spam（垃圾邮件）**：邮件被标记为垃圾邮件
- **Unsubscribes（退订）**：用户主动退订

当发送邮件到这些邮箱时，Resend 会直接返回错误，不会实际发送。

### 7.2 当前代码的处理情况

**当前代码库中没有针对收件箱抑制的特殊处理逻辑**。

搜索结果显示，代码中没有处理以下关键词的逻辑：
- `suppress` / `suppression`
- `bounce`
- `complaint`

### 7.3 对用户注册的实际影响

当用户尝试使用一个**被 Resend 抑制的邮箱**注册时：

```
用户输入邮箱 → 通过唯一性检查 → 创建验证记录 → 尝试发送邮件
                                                        ↓
                                                  Resend 返回错误
                                                        ↓
                                            错误消息显示在表单上
                                                        ↓
                                            用户无法完成注册流程
```

#### 具体影响：

1. **验证记录已创建**：`prepareVerification` 在 `sendEmail` 之前执行，即使邮件发送失败，验证记录也已写入数据库。
   - 代码位置：[signup.tsx:66-77](app/routes/_auth/signup.tsx#L66-L77)

2. **用户看到错误**：错误消息显示为 `formErrors`，用户知道发送失败。
   - 但错误消息可能不够友好（如 "Unknown Error" 或 Resend 的技术性错误信息）

3. **用户无法完成注册**：因为用户没有收到验证邮件中的 OTP 或链接。

4. **注册流程中的邮箱唯一性检查**：注册流程中如果邮箱已存在会直接返回 "A user already exists with this email"。
   - 代码位置：[signup.tsx:48-53](app/routes/_auth/signup.tsx#L48-L53)

---

## 8. 关键发现总结

### 8.1 SMTP 路径的结论

| 问题 | 答案 | 证据 |
|------|------|------|
| SMTP 路径是否存在？ | ❌ 不存在 | package.json 无 nodemailer，代码无 SMTP 配置 |
| Resend 与 SMTP 的关系？ | Resend 是唯一选择 | 决策文档明确排除 SMTP，选择邮件服务 |
| 可切换到 SMTP 吗？ | 需要自行修改 | 代码未预留 SMTP 抽象层 |

### 8.2 发送失败场景的核心问题

| 问题 | 位置 | 严重程度 | 描述 |
|------|------|----------|------|
| **数据结构不一致** | `email.server.ts` | 中 | 两个成功路径返回不同的 `response.data` 结构 |
| **账号存在性泄露** | `forgot-password.tsx` | **高** | 发起阶段返回明确的错误消息，可被攻击者枚举 |
| **伪装成功风险** | `email.server.ts` | 中 | 生产环境忘记配置 API key 时会静默失败 |
| **无事务保护** | `signup.tsx` / `forgot-password.tsx` | 中 | 邮件发送失败时，verification 记录已创建且不回滚 |
| **错误消息不友好** | 所有发送邮件的页面 | 低 | 直接显示 Resend 的技术错误消息 |

### 8.3 各页面行为对照总表

| 维度 | 注册页 (signup) | 找回密码页 (forgot-password) | 验证页 (verify) |
|------|-----------------|------------------------------|-----------------|
| **发送邮件？** | ✅ 是 | ✅ 是 | ❌ 否 |
| **发送成功后** | 重定向 `/verify` | 重定向 `/verify` | 不适用 |
| **发送失败后** | HTTP 500，formErrors | HTTP 500，formErrors | 不适用 |
| **失败时 DB 变化** | verification 已创建 | verification 已创建 | 不适用 |
| **用户不存在时** | 不适用（检查已存在） | 返回明确错误（泄露） | 返回 "Invalid code" |
| **验证成功后** | 不适用 | 不适用 | 删除 verification |
| **验证失败后** | 不适用 | 不适用 | 保留 verification |

### 8.4 容易误解的点

1. **MOCKS 环境变量的作用**：
   - ❌ 错误理解：`MOCKS=true` 会让 `sendEmail` 返回伪装成功
   - ✅ 正确理解：`MOCKS=true` 会启动 MSW mock 服务器，**拦截 HTTP 请求**，返回成功响应
   - 只有当 **既没有 API key，也没有 MOCKS** 时，才会进入 `sendEmail` 内部的伪装成功分支

2. **密码重置的两个阶段**：
   - 发起阶段 (`forgot-password.tsx`)：泄露账号存在性
   - 验证阶段 (`reset-password.server.ts`)：有隐私保护
   - 这是**不一致的设计**

3. **数据库验证记录**：
   - ❌ 错误理解：邮件发送成功后才创建验证记录
   - ✅ 正确理解：`prepareVerification` 在 `sendEmail` **之前**执行，使用 `upsert`
   - 邮件发送失败时，验证记录**已存在**于数据库中

---

## 附录：相关文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `app/utils/email.server.ts` | 邮件发送核心实现（含伪装成功分支、错误处理） |
| `app/routes/_auth/signup.tsx` | 用户注册流程 |
| `app/routes/_auth/forgot-password.tsx` | 密码重置发起阶段（泄露账号存在性） |
| `app/routes/_auth/reset-password.server.ts` | 密码重置验证阶段（有隐私保护） |
| `app/routes/_auth/verify.server.ts` | 统一验证处理逻辑 |
| `tests/mocks/resend.ts` | MSW mock 实现（开发模式下拦截 Resend 请求） |
| `tests/mocks/index.ts` | Mock 服务器启动入口 |
| `package.json` | 脚本定义（`MOCKS=true` 在 dev 脚本中）、依赖列表 |
| `docs/decisions/017-resend-email.md` | 选择 Resend 作为邮件服务的决策记录 |
| `docs/decisions/002-email-service.md` | 早期决策（已被 017 取代，明确排除 SMTP） |
