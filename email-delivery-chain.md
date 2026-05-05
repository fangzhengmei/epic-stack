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

### 1.3 Mock 模式

当 `RESEND_API_KEY` 未设置且非 mock 模式时，邮件不会实际发送，而是输出到控制台：

```typescript
if (!process.env.RESEND_API_KEY && !process.env.MOCKS) {
    console.error(`RESEND_API_KEY not set and we're not in mocks mode.`)
    console.error(`Would have sent the following email:`, JSON.stringify(email))
    return {
        status: 'success',
        data: { id: 'mocked' },
    } as const
}
```
[app/utils/email.server.ts:43-53](app/utils/email.server.ts#L43-L53)

---

## 2. 注册验证邮件流程

### 2.1 入口文件

注册流程位于 `app/routes/_auth/signup.tsx`。

### 2.2 完整流程

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

### 2.3 关键代码

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

## 3. 密码重置邮件流程

### 3.1 入口文件

密码重置流程位于 `app/routes/_auth/forgot-password.tsx`。

### 3.2 完整流程

```
用户输入用户名/邮箱 → 检查用户是否存在 → 创建验证记录 → 发送重置邮件 → 重定向到验证页面
```

### 3.3 关键代码

```typescript
// 1. 准备验证
const { verifyUrl, redirectTo, otp } = await prepareVerification({
    period: 10 * 60,
    request,
    type: 'reset-password',
    target: usernameOrEmail,
})

// 2. 发送邮件
const response = await sendEmail({
    to: user.email,
    subject: `Epic Notes Password Reset`,
    react: (
        <ForgotPasswordEmail onboardingUrl={verifyUrl.toString()} otp={otp} />
    ),
})

// 3. 处理发送结果
if (response.status === 'success') {
    return redirect(redirectTo.toString())
} else {
    return data(
        { result: submission.reply({ formErrors: [response.error.message] }) },
        { status: 500 },
    )
}
```
[app/routes/_auth/forgot-password.tsx:64-86](app/routes/_auth/forgot-password.tsx#L64-L86)

---

## 4. 发送失败后的状态回传机制

### 4.1 sendEmail 函数的返回类型

`sendEmail` 函数返回一个带标签的联合类型：

```typescript
// 成功返回
{
    status: 'success',
    data: { id: string }  // Resend 返回的邮件 ID
}

// 失败返回
{
    status: 'error',
    error: {
        name: string,        // 错误名称，如 'UnknownError'
        message: string,     // 错误消息
        statusCode: number,  // HTTP 状态码
        cause?: any          // 可选的原始错误数据
    }
}
```
[app/utils/email.server.ts:66-88](app/utils/email.server.ts#L66-L88)

### 4.2 调用方的处理方式

**注册流程 (`signup.tsx`) 和密码重置流程 (`forgot-password.tsx`) 的处理逻辑完全一致**：

| 发送结果 | 处理方式 | 代码位置 |
|---------|---------|---------|
| 成功 | 重定向到验证页面 (`/verify`) | [signup.tsx:79-80](app/routes/_auth/signup.tsx#L79-L80) |
| 失败 | 将错误消息放入 `formErrors`，返回 HTTP 500 | [signup.tsx:81-90](app/routes/_auth/signup.tsx#L81-L90) |

### 4.3 前端展示

错误消息通过 Conform 表单系统的 `formErrors` 展示：

```typescript
// 在 signup.tsx 中
<ErrorList errors={form.errors} id={form.errorId} />
```

这意味着用户会在表单顶部看到类似以下的错误消息：
- "Unknown Error"
- Resend API 返回的具体错误信息

---

## 5. 收件箱抑制（Suppression）对用户注册的影响

### 5.1 什么是收件箱抑制

邮件服务提供商（如 Resend）会维护一个**抑制列表（Suppression List）**，包含以下类型的邮箱：
- **Bounces（退信）**：邮箱不存在、邮箱已满、被拒收等
- **Complaints（投诉）**：用户点击了"这是垃圾邮件"
- **Spam（垃圾邮件）**：邮件被标记为垃圾邮件
- **Unsubscribes（退订）**：用户主动退订

当发送邮件到这些邮箱时，Resend 会直接返回错误，不会实际发送。

### 5.2 当前代码的处理情况

**当前代码库中没有针对收件箱抑制的特殊处理逻辑**。

搜索结果显示，代码中没有处理以下关键词的逻辑：
- `suppress` / `suppression`
- `bounce`
- `spam`（仅有 honeypot 防垃圾机器人，非邮件发送层面）
- `complaint`

### 5.3 对用户注册的实际影响

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

4. **隐私泄露风险**：由于邮箱唯一性检查在发送邮件之前执行，攻击者可以通过错误消息判断哪些邮箱已注册。
   - 注意：代码中已经考虑了这一点，密码重置流程中即使用户不存在也不会泄露信息。
   - 但注册流程中，如果邮箱已存在会直接返回 "A user already exists with this email"。

### 5.4 可能的改进方向

1. **错误消息优化**：
   - 区分不同类型的 Resend 错误（网络错误、配置错误、收件人问题等）
   - 对用户显示更友好的提示（如 "请检查您的邮箱是否正确" 或 "请使用其他邮箱"）

2. **抑制列表检查**：
   - 在发送邮件前主动查询 Resend 的抑制列表 API
   - 如果邮箱在抑制列表中，提前提示用户使用其他邮箱

3. **事务处理**：
   - 考虑使用数据库事务，确保邮件发送成功后才写入验证记录
   - 或在邮件发送失败后清理已创建的验证记录

4. **重试机制**：
   - 对于临时性错误（如网络问题），实现指数退避重试

---

## 6. 验证流程架构总结

### 6.1 验证类型

`verify.server.ts` 支持多种验证类型：

```typescript
// 验证类型枚举（从代码逻辑推断）
type VerificationTypes = 
    | 'onboarding'      // 新用户注册验证
    | 'reset-password'  // 密码重置验证
    | 'change-email'    // 邮箱变更验证
    | '2fa'             // 两步验证
```

### 6.2 统一验证入口

所有验证类型共用同一个验证处理函数 `validateRequest`：

```typescript
switch (submissionValue[typeQueryParam]) {
    case 'reset-password': {
        await deleteVerification()
        return handleResetPasswordVerification({ request, body, submission })
    }
    case 'onboarding': {
        await deleteVerification()
        return handleOnboardingVerification({ request, body, submission })
    }
    case 'change-email': {
        await deleteVerification()
        return handleChangeEmailVerification({ request, body, submission })
    }
    case '2fa': {
        return handleLoginTwoFactorVerification({ request, body, submission })
    }
}
```
[app/routes/_auth/verify.server.ts:183-199](app/routes/_auth/verify.server.ts#L183-L199)

### 6.3 数据存储

验证记录存储在 `verification` 表中，使用复合唯一键 `(target, type)`：

```typescript
// 存储验证数据
await prisma.verification.upsert({
    where: { target_type: { target, type } },
    create: verificationData,
    update: verificationData,
})
```
[app/utils/verification.server.ts:102-106](app/utils/verification.server.ts#L102-L106)

---

## 附录：相关文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `app/utils/email.server.ts` | 邮件发送核心实现（Resend API 调用） |
| `app/routes/_auth/signup.tsx` | 用户注册流程（含验证邮件发送） |
| `app/routes/_auth/forgot-password.tsx` | 密码重置流程（含重置邮件发送） |
| `app/routes/_auth/verify.server.ts` | 统一验证处理逻辑 |
| `app/utils/verification.server.ts` | 验证会话存储和验证准备工具 |
| `docs/decisions/017-resend-email.md` | 选择 Resend 作为邮件服务的决策记录 |
| `docs/decisions/002-email-service.md` | 早期选择邮件服务的决策记录（已被 017 取代） |
