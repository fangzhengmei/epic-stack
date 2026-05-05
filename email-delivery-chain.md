# Epic Stack 邮件发送链路分析

## 1. 邮件发送路径

### 1.1 使用的服务

Epic Stack 使用 **Resend** 作为邮件发送服务，而非 SMTP。

**决策依据**（参考 `docs/decisions/017-resend-email.md`）：
- Resend 有清晰且慷慨的免费层级（每月 3,000 封邮件）
- UI 简单易用
- 价格比 Mailgun 更便宜
- 使用 REST API 而非 SDK，降低与特定服务的耦合度

### 1.2 核心实现

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

## 6. 发送失败后的状态回传机制

### 6.1 失败返回类型

当 Resend API 返回错误或响应无法解析时：

```typescript
// email.server.ts:71-88
} else {
    const parseResult = resendErrorSchema.safeParse(data)
    if (parseResult.success) {
        return {
            status: 'error',
            error: parseResult.data,  // { name, message, statusCode }
        } as const
    } else {
        return {
            status: 'error',
            error: {
                name: 'UnknownError',
                message: 'Unknown Error',
                statusCode: 500,
                cause: data,
            } satisfies ResendError,
        } as const
    }
}
```

### 6.2 调用方的统一处理

**注册流程 (`signup.tsx`) 和密码重置流程 (`forgot-password.tsx`) 的处理逻辑完全一致**：

```typescript
if (response.status === 'success') {
    return redirect(redirectTo.toString())
} else {
    return data(
        { result: submission.reply({ formErrors: [response.error.message] }) },
        { status: 500 },
    )
}
```

| 发送结果 | 处理方式 |
|---------|---------|
| `status === 'success'` | 重定向到验证页面 (`/verify`) |
| `status === 'error'` | 将 `response.error.message` 放入 `formErrors`，返回 HTTP 500 |

### 6.3 注意事项

⚠️ **"伪装成功"场景不会触发错误处理**

当未配置 `RESEND_API_KEY` 且不在 mock 模式时，`sendEmail` 返回 `status: 'success'`，调用方会认为发送成功，不会进入错误处理分支。

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

### 8.1 已确认的问题

| 问题 | 位置 | 严重程度 | 描述 |
|------|------|----------|------|
| **数据结构不一致** | `email.server.ts` | 中 | 两个成功路径返回不同的 `response.data` 结构 |
| **账号存在性泄露** | `forgot-password.tsx` | **高** | 发起阶段返回明确的错误消息，可被攻击者枚举 |
| **伪装成功风险** | `email.server.ts` | 中 | 生产环境忘记配置 API key 时会静默失败 |

### 8.2 容易误解的点

1. **MOCKS 环境变量的作用**：
   - ❌ 错误理解：`MOCKS=true` 会让 `sendEmail` 返回伪装成功
   - ✅ 正确理解：`MOCKS=true` 会启动 MSW mock 服务器，**拦截 HTTP 请求**，返回成功响应
   - 只有当 **既没有 API key，也没有 MOCKS** 时，才会进入 `sendEmail` 内部的伪装成功分支

2. **密码重置的两个阶段**：
   - 发起阶段 (`forgot-password.tsx`)：泄露账号存在性
   - 验证阶段 (`reset-password.server.ts`)：有隐私保护
   - 这是**不一致的设计**

---

## 附录：相关文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `app/utils/email.server.ts` | 邮件发送核心实现（含伪装成功分支） |
| `app/routes/_auth/signup.tsx` | 用户注册流程 |
| `app/routes/_auth/forgot-password.tsx` | 密码重置发起阶段（泄露账号存在性） |
| `app/routes/_auth/reset-password.server.ts` | 密码重置验证阶段（有隐私保护） |
| `app/routes/_auth/verify.server.ts` | 统一验证处理逻辑 |
| `tests/mocks/resend.ts` | MSW mock 实现（开发模式下拦截 Resend 请求） |
| `tests/mocks/index.ts` | Mock 服务器启动入口 |
| `package.json` | 脚本定义（`MOCKS=true` 在 dev 脚本中） |
| `docs/decisions/017-resend-email.md` | 选择 Resend 作为邮件服务的决策记录 |
