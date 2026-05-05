# Epic Stack Passkey / WebAuthn 实现分析

## 概述

Epic Stack 实现了基于 WebAuthn 标准的 Passkey（密钥）认证系统，使用 `@simplewebauthn/server` 和 `@simplewebauthn/browser` 库来处理协议细节。Passkey 提供了更安全、抗钓鱼的无密码认证替代方案。

相关文件：
- 注册流程：`app/routes/_auth/webauthn/registration.ts`
- 认证流程：`app/routes/_auth/webauthn/authentication.ts`
- 工具函数：`app/routes/_auth/webauthn/utils.server.ts`
- 设置页面：`app/routes/settings/profile/passkeys.tsx`
- 数据库模型：`prisma/schema.prisma`

---

## 1. 注册流程 (Registration)

### 1.1 流程概览

注册流程分为两个阶段：
1. **获取注册选项 (GET `/webauthn/registration`)** - 生成挑战和配置
2. **验证注册响应 (POST `/webauthn/registration`)** - 验证浏览器返回的凭证

### 1.2 获取注册选项 (Loader)

```typescript
// app/routes/_auth/webauthn/registration.ts:16-55
export async function loader({ request }: Route.LoaderArgs) {
    const userId = await requireUserId(request)
    const passkeys = await prisma.passkey.findMany({
        where: { userId },
        select: { id: true },
    })
    const user = await prisma.user.findUniqueOrThrow({
        where: { id: userId },
        select: { email: true, name: true, username: true },
    })

    const config = getWebAuthnConfig(request)
    const options = await generateRegistrationOptions({
        rpName: config.rpName,
        rpID: config.rpID,
        userName: user.username,
        userID: new TextEncoder().encode(userId),
        userDisplayName: user.name ?? user.email,
        attestationType: 'none',
        excludeCredentials: passkeys, // 排除已注册的密钥
        authenticatorSelection: {
            residentKey: 'preferred',
            userVerification: 'preferred',
        },
    })

    return Response.json(
        { options },
        {
            headers: {
                'Set-Cookie': await passkeyCookie.serialize(
                    PasskeyCookieSchema.parse({
                        challenge: options.challenge,
                        userId: options.user.id,
                    }),
                ),
            },
        },
    )
}
```

**关键点：**
- `requireUserId(request)` - 用户必须已登录才能注册新密钥
- `excludeCredentials: passkeys` - 防止重复注册同一设备
- `residentKey: 'preferred'` - 优先使用可发现凭证（允许用户选择账户）
- `userVerification: 'preferred'` - 优先请求用户验证（生物识别/PIN）

### 1.3 验证注册响应 (Action)

```typescript
// app/routes/_auth/webauthn/registration.ts:57-136
export async function action({ request }: Route.ActionArgs) {
    try {
        const userId = await requireUserId(request)

        const body = await request.json()
        const result = RegistrationResponseSchema.safeParse(body)
        if (!result.success) {
            throw new Error('Invalid registration response')
        }

        // 从 Cookie 获取挑战
        const passkeyCookieData = await passkeyCookie.parse(
            request.headers.get('Cookie'),
        )
        const parsedPasskeyCookieData =
            PasskeyCookieSchema.safeParse(passkeyCookieData)
        if (!parsedPasskeyCookieData.success) {
            throw new Error('No challenge found')
        }
        const { challenge, userId: webauthnUserId } = parsedPasskeyCookieData.data

        const domain = new URL(getDomainUrl(request)).hostname
        const rpID = domain
        const origin = getDomainUrl(request)

        const verification = await verifyRegistrationResponse({
            response: data,
            expectedChallenge: challenge,
            expectedOrigin: origin,
            expectedRPID: rpID,
            requireUserVerification: true,
        })

        const { verified, registrationInfo } = verification
        if (!verified || !registrationInfo) {
            throw new Error('Registration verification failed')
        }
        const { credential, credentialDeviceType, credentialBackedUp, aaguid } =
            registrationInfo

        const existingPasskey = await prisma.passkey.findUnique({
            where: { id: credential.id },
            select: { id: true },
        })

        if (existingPasskey) {
            throw new Error('This passkey has already been registered')
        }

        // 在数据库中创建新密钥
        await prisma.passkey.create({
            data: {
                id: credential.id,
                aaguid,
                publicKey: Buffer.from(credential.publicKey),
                userId,
                webauthnUserId,
                counter: credential.counter,
                deviceType: credentialDeviceType,
                backedUp: credentialBackedUp,
                transports: credential.transports?.join(','),
            },
        })

        return Response.json({ status: 'success' } as const, {
            headers: {
                'Set-Cookie': await passkeyCookie.serialize('', { maxAge: 0 }),
            },
        })
    } catch (error) {
        // 错误处理...
    }
}
```

---

## 2. 认证流程 (Authentication)

### 2.1 流程概览

认证流程同样分为两个阶段：
1. **获取认证选项 (GET `/webauthn/authentication`)** - 生成挑战
2. **验证认证响应 (POST `/webauthn/authentication`)** - 验证签名并创建会话

### 2.2 获取认证选项 (Loader)

```typescript
// app/routes/_auth/webauthn/authentication.ts:15-27
export async function loader({ request }: Route.LoaderArgs) {
    const config = getWebAuthnConfig(request)
    const options = await generateAuthenticationOptions({
        rpID: config.rpID,
        userVerification: 'preferred',
    })

    const cookieHeader = await passkeyCookie.serialize({
        challenge: options.challenge,
    })

    return Response.json({ options }, { headers: { 'Set-Cookie': cookieHeader } })
}
```

**注意：** 认证阶段不需要用户已登录，也不需要 `excludeCredentials`，因为浏览器会自动提示用户选择已注册的密钥。

### 2.3 验证认证响应并接入 Session (Action)

```typescript
// app/routes/_auth/webauthn/authentication.ts:29-113
export async function action({ request }: Route.ActionArgs) {
    const cookieHeader = request.headers.get('Cookie')
    const cookie = await passkeyCookie.parse(cookieHeader)
    const deletePasskeyCookie = await passkeyCookie.serialize('', { maxAge: 0 })
    try {
        if (!cookie?.challenge) {
            throw new Error('Authentication challenge not found')
        }

        const body = await request.json()
        const result = PasskeyLoginBodySchema.safeParse(body)
        if (!result.success) {
            throw new Error('Invalid authentication response')
        }
        const { authResponse, remember, redirectTo } = result.data

        // 根据凭证 ID 查找密钥记录
        const passkey = await prisma.passkey.findUnique({
            where: { id: authResponse.id },
            include: { user: true },
        })
        if (!passkey) {
            throw new Error('Passkey not found')
        }

        const config = getWebAuthnConfig(request)

        // 验证签名
        const verification = await verifyAuthenticationResponse({
            response: authResponse,
            expectedChallenge: cookie.challenge,
            expectedOrigin: config.origin,
            expectedRPID: config.rpID,
            credential: {
                id: authResponse.id,
                publicKey: passkey.publicKey,
                counter: Number(passkey.counter),
            },
        })

        if (!verification.verified) {
            throw new Error('Authentication verification failed')
        }

        // 更新计数器（防止重放攻击）
        await prisma.passkey.update({
            where: { id: passkey.id },
            data: { counter: BigInt(verification.authenticationInfo.newCounter) },
        })

        // 创建新的 Session（与普通登录相同）
        const session = await prisma.session.create({
            select: { id: true, expirationDate: true, userId: true },
            data: {
                expirationDate: getSessionExpirationDate(),
                userId: passkey.userId,
            },
        })

        // 接入普通 Session 系统
        const response = await handleNewSession(
            {
                request,
                session,
                remember,
                redirectTo: redirectTo ?? undefined,
            },
            { headers: { 'Set-Cookie': deletePasskeyCookie } },
        )

        return Response.json(
            {
                status: 'success',
                location: response.headers.get('Location'),
            },
            { headers: response.headers },
        )
    } catch (error) {
        // 错误处理...
    }
}
```

---

## 3. 挑战生成与临时 Cookie 管理

### 3.1 Cookie 配置

```typescript
// app/routes/_auth/webauthn/utils.server.ts:9-16
export const passkeyCookie = createCookie('webauthn-challenge', {
    path: '/',
    sameSite: 'lax',
    httpOnly: true,
    maxAge: 60 * 60 * 2, // 2 小时有效期
    secure: process.env.NODE_ENV === 'production',
    secrets: [process.env.SESSION_SECRET],
})
```

**Cookie 安全属性：**
- `httpOnly: true` - 防止 XSS 攻击读取
- `sameSite: 'lax'` - 防止 CSRF 攻击
- `secure: true` (生产环境) - 仅通过 HTTPS 传输
- `maxAge: 2小时` - 短期有效，减少风险窗口

### 3.2 Cookie Schema

```typescript
// app/routes/_auth/webauthn/utils.server.ts:18-21
export const PasskeyCookieSchema = z.object({
    challenge: z.string(), // 随机生成的挑战字符串
    userId: z.string(),    // 仅注册时使用，用户 ID
})
```

### 3.3 挑战生成机制

挑战由 `@simplewebauthn/server` 库生成：

| 阶段 | 函数 | Cookie 内容 |
|------|------|-------------|
| 注册开始 | `generateRegistrationOptions()` | `{ challenge, userId }` |
| 认证开始 | `generateAuthenticationOptions()` | `{ challenge }` |

### 3.4 Cookie 写入与清理时机

**写入时机：**
- 注册：GET `/webauthn/registration` 返回响应时
- 认证：GET `/webauthn/authentication` 返回响应时

**清理时机：**
- ✅ 注册成功：`{ maxAge: 0 }` 立即删除
- ✅ 认证成功：`{ maxAge: 0 }` 立即删除
- ✅ 认证失败：`{ headers: { 'Set-Cookie': deletePasskeyCookie } }`

**安全设计：**
1. 挑战一次性使用，验证后立即失效
2. Cookie 与 session 分离，passkey 流程专用
3. 2 小时超时，防止长期悬挂的挑战

---

## 4. 浏览器验证后接入普通 Session

### 4.1 Session 集成点

Passkey 认证成功后，**复用现有的 Session 系统**，与密码登录/OAuth 登录完全一致。

```typescript
// app/routes/_auth/webauthn/authentication.ts:77-93
// 1. 创建 Session 记录（与普通登录相同）
const session = await prisma.session.create({
    select: { id: true, expirationDate: true, userId: true },
    data: {
        expirationDate: getSessionExpirationDate(),
        userId: passkey.userId,
    },
})

// 2. 调用统一的 Session 处理函数
const response = await handleNewSession(
    {
        request,
        session,
        remember,
        redirectTo: redirectTo ?? undefined,
    },
    { headers: { 'Set-Cookie': deletePasskeyCookie } },
)
```

### 4.2 handleNewSession 内部逻辑

```typescript
// app/routes/_auth/login.server.ts:17-81
export async function handleNewSession(
    {
        request,
        session,
        redirectTo,
        remember,
    }: {
        request: Request
        session: { userId: string; id: string; expirationDate: Date }
        redirectTo?: string
        remember: boolean
    },
    responseInit?: ResponseInit,
) {
    // 检查用户是否启用了 2FA
    const verification = await prisma.verification.findUnique({
        select: { id: true },
        where: {
            target_type: { target: session.userId, type: twoFAVerificationType },
        },
    })
    const userHasTwoFactor = Boolean(verification)

    if (userHasTwoFactor) {
        // 有 2FA：先跳转到 2FA 验证页面
        const verifySession = await verifySessionStorage.getSession()
        verifySession.set(unverifiedSessionIdKey, session.id)
        verifySession.set(rememberKey, remember)
        const redirectUrl = getRedirectToUrl({...})
        return redirect(
            `${redirectUrl.pathname}?${redirectUrl.searchParams}`,
            combineResponseInits(
                {
                    headers: {
                        'set-cookie':
                            await verifySessionStorage.commitSession(verifySession),
                    },
                },
                responseInit, // 包含删除 passkey cookie 的 header
            ),
        )
    } else {
        // 无 2FA：直接设置 auth session
        const authSession = await authSessionStorage.getSession(
            request.headers.get('cookie'),
        )
        authSession.set(sessionKey, session.id)

        return redirect(
            safeRedirect(redirectTo),
            combineResponseInits(
                {
                    headers: {
                        'set-cookie': await authSessionStorage.commitSession(authSession, {
                            expires: remember ? session.expirationDate : undefined,
                        }),
                    },
                },
                responseInit,
            ),
        )
    }
}
```

### 4.3 Session 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    Passkey 认证成功后                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. prisma.session.create()                                 │
│     ┌─────────────────────────────────────────┐            │
│     │  Session 表                              │            │
│     │  - id: "sess_abc123"                   │            │
│     │  - userId: "user_xyz"                   │            │
│     │  - expirationDate: 2 weeks later        │            │
│     └─────────────────────────────────────────┘            │
│                           ↓                                  │
│  2. handleNewSession()                                      │
│                           ↓                                  │
│  ┌──────────────────────────────────────────────────┐     │
│  │  有 2FA?                                          │     │
│  │    ├─ YES → verifySessionStorage (临时)          │     │
│  │    │              - unverifiedSessionId          │     │
│  │    │              - remember                      │     │
│  │    │         ↓                                    │     │
│  │    │         跳转到 /verify?type=2fa              │     │
│  │    │                                              │     │
│  │    └─ NO  → authSessionStorage (正式)            │     │
│  │                  - sessionKey = "sess_abc123"   │     │
│  │                  - expires: 2 weeks (if remember)│     │
│  │              ↓                                    │     │
│  │              跳转到 redirectTo                     │     │
│  └──────────────────────────────────────────────────┘     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.4 Cookie 组合

最终响应包含多个 Set-Cookie：

| Cookie | 操作 | 来源 |
|--------|------|------|
| `webauthn-challenge` | 删除 (`maxAge: 0`) | passkey 流程 |
| `auth_session` | 设置 (或 `verify_session`) | 普通 Session 系统 |

---

## 5. 设置页面多密钥管理

### 5.1 页面结构

文件：`app/routes/settings/profile/passkeys.tsx`

```tsx
export default function Passkeys({ loaderData }: Route.ComponentProps) {
    const revalidator = useRevalidator()
    const [error, setError] = useState<string | null>(null)

    async function handlePasskeyRegistration() {
        // 前端注册流程...
    }

    return (
        <div className="flex flex-col gap-6">
            {/* 标题 + 注册按钮 */}
            <div className="flex justify-between gap-4">
                <h1 className="text-h1">Passkeys</h1>
                <Button onClick={handlePasskeyRegistration}>
                    Register new passkey
                </Button>
            </div>

            {/* 密钥列表 */}
            {loaderData.passkeys.length ? (
                <ul>
                    {loaderData.passkeys.map((passkey) => (
                        <li key={passkey.id}>
                            {/* 显示：设备类型 + 注册时间 */}
                            <span>
                                {passkey.deviceType === 'platform'
                                    ? 'Device'
                                    : 'Security Key'}
                            </span>
                            <span>Registered {formatDistanceToNow(...)}</span>
                            
                            {/* 删除按钮 */}
                            <Form method="POST">
                                <input type="hidden" name="passkeyId" value={passkey.id} />
                                <Button type="submit" name="intent" value="delete">
                                    Delete
                                </Button>
                            </Form>
                        </li>
                    ))}
                </ul>
            ) : (
                <div>No passkeys registered yet</div>
            )}
        </div>
    )
}
```

### 5.2 前端注册流程

```typescript
// app/routes/settings/profile/passkeys.tsx:98-124
async function handlePasskeyRegistration() {
    try {
        setError(null)
        
        // 1. 获取注册选项（服务器生成挑战）
        const resp = await fetch('/webauthn/registration')
        const jsonResult = await resp.json()
        const parsedResult = RegistrationOptionsSchema.parse(jsonResult)

        // 2. 调用浏览器 WebAuthn API（弹出生物识别提示）
        const regResult = await startRegistration({
            optionsJSON: parsedResult.options,
        })

        // 3. 发送验证到服务器
        const verificationResp = await fetch('/webauthn/registration', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(regResult),
        })

        if (!verificationResp.ok) {
            throw new Error('Failed to verify registration')
        }

        // 4. 刷新页面数据
        void revalidator.revalidate()
    } catch (err) {
        console.error('Failed to create passkey:', err)
        setError('Failed to create passkey. Please try again.')
    }
}
```

### 5.3 后端删除逻辑

```typescript
// app/routes/settings/profile/passkeys.tsx:30-57
export async function action({ request }: Route.ActionArgs) {
    const userId = await requireUserId(request)
    const formData = await request.formData()
    const intent = formData.get('intent')

    if (intent === 'delete') {
        const passkeyId = formData.get('passkeyId')
        if (typeof passkeyId !== 'string') {
            return Response.json(
                { status: 'error', error: 'Invalid passkey ID' },
                { status: 400 },
            )
        }

        // 安全删除：同时验证 userId，防止越权
        await prisma.passkey.delete({
            where: {
                id: passkeyId,
                userId, // 关键：确保密钥属于当前用户
            },
        })
        return Response.json({ status: 'success' })
    }

    return Response.json(
        { status: 'error', error: 'Invalid intent' },
        { status: 400 },
    )
}
```

### 5.4 多密钥管理特性

| 特性 | 实现方式 |
|------|----------|
| 列出所有密钥 | `prisma.passkey.findMany({ where: { userId } })` |
| 显示设备类型 | `deviceType: 'platform' | 'cross-platform'` |
| 显示注册时间 | `formatDistanceToNow(createdAt)` |
| 防止重复注册 | `excludeCredentials` + 数据库唯一约束 |
| 安全删除 | `where: { id, userId }` 双重验证 |

---

## 6. 数据库模型

```prisma
// prisma/schema.prisma:171-186
model Passkey {
  id             String   @id           // 凭证 ID (credential.id)
  aaguid         String                 // 认证器型号标识
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt
  publicKey      Bytes                  // 公钥（用于验证签名）
  user           User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  userId         String
  webauthnUserId String                 // WebAuthn 用户句柄
  counter        BigInt                 // 签名计数器（防重放）
  deviceType     String                 // 'singleDevice' 或 'multiDevice'
  backedUp       Boolean                // 是否已备份（同步）
  transports     String?                // 传输方式，逗号分隔 (usb,nfc,ble,...)

  @@index(userId)
}
```

### 6.1 关键字段说明

| 字段 | 用途 |
|------|------|
| `id` | 凭证唯一标识，由认证器生成 |
| `publicKey` | 核心：用于验证后续认证的签名 |
| `counter` | 每次认证递增，防止重放攻击 |
| `aaguid` | 可用于识别认证器品牌/型号 |
| `deviceType` | 区分内置设备(platform)和安全密钥(cross-platform) |
| `backedUp` | 指示是否已同步到云（如 iCloud Keychain） |

---

## 7. 完整流程图

### 7.1 注册流程

```
┌──────────┐                    ┌──────────┐                    ┌──────────┐
│  Browser │                    │  Server  │                    │    DB    │
└────┬─────┘                    └────┬─────┘                    └────┬─────┘
     │                                │                                │
     │  1. GET /webauthn/registration │                                │
     │────────────────────────────────>│                                │
     │                                │                                │
     │                                │  2. 查询用户已有密钥            │
     │                                │───────────────────────────────>│
     │                                │                                │
     │                                │  3. generateRegistrationOptions │
     │                                │     (生成 challenge)           │
     │                                │                                │
     │                                │  4. Set-Cookie:                │
     │                                │     webauthn-challenge         │
     │                                │     { challenge, userId }      │
     │  5. 返回 options               │                                │
     │<────────────────────────────────│                                │
     │                                │                                │
     │  6. navigator.credentials.create() │                            │
     │     (弹出生物识别提示)          │                                │
     │                                │                                │
     │  7. POST /webauthn/registration │                                │
     │     { attestationResponse }    │                                │
     │────────────────────────────────>│                                │
     │                                │                                │
     │                                │  8. 从 Cookie 读取 challenge    │
     │                                │                                │
     │                                │  9. verifyRegistrationResponse  │
     │                                │                                │
     │                                │  10. 存储 Passkey 记录          │
     │                                │───────────────────────────────>│
     │                                │                                │
     │                                │  11. Set-Cookie:                │
     │                                │     webauthn-challenge         │
     │                                │     (maxAge: 0 → 删除)         │
     │  12. { status: 'success' }     │                                │
     │<────────────────────────────────│                                │
     │                                │                                │
```

### 7.2 认证流程

```
┌──────────┐                    ┌──────────┐                    ┌──────────┐
│  Browser │                    │  Server  │                    │    DB    │
└────┬─────┘                    └────┬─────┘                    └────┬─────┘
     │                                │                                │
     │  1. GET /webauthn/authentication │                              │
     │────────────────────────────────>│                                │
     │                                │                                │
     │                                │  2. generateAuthenticationOptions│
     │                                │     (生成 challenge)           │
     │                                │                                │
     │                                │  3. Set-Cookie:                │
     │                                │     webauthn-challenge         │
     │                                │     { challenge }               │
     │  4. 返回 options               │                                │
     │<────────────────────────────────│                                │
     │                                │                                │
     │  5. navigator.credentials.get()│                                │
     │     (弹出密钥选择/生物识别)      │                                │
     │                                │                                │
     │  6. POST /webauthn/authentication │                             │
     │     { authResponse, remember } │                                │
     │────────────────────────────────>│                                │
     │                                │                                │
     │                                │  7. 从 Cookie 读取 challenge    │
     │                                │                                │
     │                                │  8. 查询 Passkey + User        │
     │                                │───────────────────────────────>│
     │                                │                                │
     │                                │  9. verifyAuthenticationResponse│
     │                                │     (使用存储的 publicKey)     │
     │                                │                                │
     │                                │  10. 更新 counter               │
     │                                │───────────────────────────────>│
     │                                │                                │
     │                                │  11. 创建 Session 记录          │
     │                                │───────────────────────────────>│
     │                                │                                │
     │                                │  12. handleNewSession()        │
     │                                │      - 设置 auth_session Cookie│
     │                                │      - 检查 2FA                 │
     │                                │                                │
     │                                │  13. Set-Cookie:                │
     │                                │      - webauthn-challenge (删除)│
     │                                │      - auth_session (设置)      │
     │  14. { status: 'success',      │                                │
     │       location: '/home' }      │                                │
     │<────────────────────────────────│                                │
     │                                │                                │
```

---

## 8. 安全设计要点

### 8.1 挑战与 Cookie 安全

| 安全措施 | 实现 |
|----------|------|
| 防止重放攻击 | 一次性 challenge + 签名计数器 |
| 防止 XSS 读取 | `httpOnly: true` |
| 防止 CSRF | `sameSite: 'lax'` |
| 防止中间人 | `secure: true` (生产环境) |
| 短期有效 | `maxAge: 2 小时` |
| 使用后清理 | 验证成功/失败后立即删除 |

### 8.2 密钥存储安全

| 措施 | 说明 |
|------|------|
| 私钥永不离开设备 | 由浏览器/操作系统管理 |
| 服务器只存公钥 | 公钥只能用于验证，不能伪造签名 |
| 用户验证 | 支持 `userVerification` (生物识别/PIN) |
| 域名绑定 | 密钥绑定到特定 RPID，防止跨域使用 |

### 8.3 授权安全

| 场景 | 保护措施 |
|------|----------|
| 注册密钥 | 需要已登录 (`requireUserId`) |
| 删除密钥 | 验证 `userId` 匹配 |
| 越权访问 | 所有操作都有用户上下文检查 |

---

## 9. 依赖库

- **服务器端**: `@simplewebauthn/server`
  - `generateRegistrationOptions()`
  - `verifyRegistrationResponse()`
  - `generateAuthenticationOptions()`
  - `verifyAuthenticationResponse()`

- **客户端**: `@simplewebauthn/browser`
  - `startRegistration()` - 封装 `navigator.credentials.create()`
  - `startAuthentication()` - 封装 `navigator.credentials.get()`

- **Schema 验证**: `zod`
  - `PasskeyCookieSchema`
  - `RegistrationResponseSchema`
  - `AuthenticationResponseSchema`
  - `PasskeyLoginBodySchema`

- **Cookie 管理**: `react-router`
  - `createCookie()` - 类型安全的 Cookie 操作

---

## 10. 参考文档

- 决策文档: `docs/decisions/039-passkeys.md`
- 认证文档: `docs/authentication.md`
- WebAuthn 标准: https://www.w3.org/TR/webauthn-2/
- SimpleWebAuthn: https://simplewebauthn.dev/
