# Epic Stack Passkey 登录实现深度分析

## 概述

本文档深入分析 Epic Stack 中 Passkey（WebAuthn）登录的完整实现，包括注册流程和登录流程的安全生命周期，以及两者之间的关键差异。

---

## 技术栈

| 组件 | 库/工具 |
|------|---------|
| 服务端 WebAuthn | `@simplewebauthn/server` |
| 浏览器端 WebAuthn | `@simplewebauthn/browser` |
| 数据验证 | `zod` |
| 数据库 | SQLite + Prisma |
| 会话管理 | React Router Cookie + Prisma Session |

---

## 一、数据库模型设计

### Passkey 表结构

```prisma
model Passkey {
  id             String   @id           // 凭据 ID (credential.id)
  aaguid         String                 // 认证器的 AAGUID
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt
  publicKey      Bytes                  // 公钥 (用于验证签名)
  userId         String                 // 关联的用户 ID
  webauthnUserId String                // WebAuthn 用户句柄
  counter        BigInt                 // 签名计数器 (防重放)
  deviceType     String                 // 'singleDevice' 或 'multiDevice'
  backedUp       Boolean                // 是否已备份
  transports     String?                // 传输方式 (逗号分隔)

  user           User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index(userId)
}
```

### 关键字段说明

| 字段 | 作用 | 安全意义 |
|------|------|----------|
| `publicKey` | 存储认证器生成的公钥 | 服务端使用此公钥验证客户端签名 |
| `counter` | 签名计数器 | 防止克隆攻击，每次验证后递增 |
| `aaguid` | 认证器全局唯一标识 | 识别认证器型号，可用于策略控制 |
| `deviceType` | 设备类型 | 区分平台认证器(如 Touch ID)和跨平台认证器(如 YubiKey) |
| `backedUp` | 备份状态 | 指示凭据是否已同步到云，影响安全策略 |

---

## 二、公共基础设施

### 2.1 Challenge Cookie 管理 (`utils.server.ts:9-16`)

```typescript
export const passkeyCookie = createCookie('webauthn-challenge', {
  path: '/',
  sameSite: 'lax',
  httpOnly: true,
  maxAge: 60 * 60 * 2,      // 2 小时有效期
  secure: process.env.NODE_ENV === 'production',
  secrets: [process.env.SESSION_SECRET],
})
```

**安全设计要点：**
- `httpOnly: true` - 防止 XSS 攻击读取 cookie
- `sameSite: 'lax'` - 防止 CSRF 攻击
- `secure` - 生产环境仅通过 HTTPS 传输
- 签名加密 - 使用 `SESSION_SECRET` 签名，防止篡改
- 有效期 2 小时 - 限制挑战的时间窗口

### 2.2 WebAuthn 配置 (`utils.server.ts:81-89`)

```typescript
export function getWebAuthnConfig(request: Request) {
  const url = new URL(getDomainUrl(request))
  return {
    rpName: url.hostname,
    rpID: url.hostname,
    origin: url.origin,
  } as const
}
```

**关键概念：**
- `rpID` (Relying Party ID) - 依赖方标识，通常是域名
- `origin` - 完整源地址，用于验证响应来源
- 动态从请求获取 - 支持多环境部署

---

## 三、注册流程详解

### 3.1 流程图

```
┌──────────────┐     GET /webauthn/registration      ┌──────────────┐
│  浏览器  │ ─────────────────────────────────> │   服务端    │
│ (用户已登录│                                    │  (已认证)   │
└──────────┘                                    └──────┬───────┘
                                                     │
       ┌─────────────────────────────────────────────┘
       │ 1. requireUserId() 验证用户登录状态
       │ 2. 查询用户已有 passkeys (用于排除)
       │ 3. generateRegistrationOptions() 生成选项
       │ 4. 将 challenge + userId 存入加密 cookie
       └─────────────────────────────────────────────┐
                                                     │
┌──────────┐     返回 RegistrationOptions          │
│  浏览器  │ <───────────────────────────────── ───┘
│          │
└────┬─────┘
     │
     │ 用户触发设备认证 (Touch ID / Face ID / YubiKey)
     │ 浏览器调用 navigator.credentials.create()
     │
┌────┴─────┐     POST /webauthn/registration      ┌──────────────┐
│  浏览器  │ ─────────────────────────────────> │   服务端    │
│          │    (携带 RegistrationResponse)      │              │
└──────────┘                                    └──────┬───────┘
                                                     │
       ┌─────────────────────────────────────────────┘
       │ 1. 从 cookie 读取 challenge 进行绑定验证
       │ 2. verifyRegistrationResponse() 验证响应
       │    - 验证签名
       │    - 验证 origin/rpID
       │    - 验证用户存在性
       │ 3. 检查凭据是否已注册
       │ 4. 将凭据信息写入 Passkey 表
       │ 5. 清除 challenge cookie
       └─────────────────────────────────────────────┐
                                                     │
┌──────────┐         返回成功状态                     │
│  浏览器  │ <──────────────────────────────────────┘
│          │
└──────────┘
```

### 3.2 服务端实现细节

#### 阶段 1: 生成注册选项 (`registration.ts:16-55`)

```typescript
export async function loader({ request }: Route.LoaderArgs) {
  const userId = await requireUserId(request)  // 关键：用户必须已登录
  
  // 获取用户已有 passkeys，防止重复注册同一设备
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
    attestationType: 'none',           // 不要求认证器证明
    excludeCredentials: passkeys,          // 排除已有凭据
    authenticatorSelection: {
      residentKey: 'preferred',         // 优先使用可发现凭据
      userVerification: 'preferred',     // 优先要求用户验证
    },
  })

  // 将 challenge 与 userId 绑定存入 cookie
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

**注册选项关键设计：**

| 参数 | 值 | 说明 |
|------|-----|------|
| `attestationType: 'none'` | 无认证 | 不验证认证器制造商身份，隐私优先 |
| `excludeCredentials` | 已有 passkeys | 防止用户重复注册同一设备 |
| `residentKey: 'preferred'` | 优先可发现 | 允许无用户名登录 (passkey 特性) |
| `userVerification: 'preferred'` | 优先验证 | 要求生物识别/PIN 验证 |

#### 阶段 2: 验证注册响应 (`registration.ts:57-136`)

```typescript
export async function action({ request }: Route.ActionArgs) {
  try {
    const userId = await requireUserId(request)
    const body = await request.json()
    const result = RegistrationResponseSchema.safeParse(body)
    if (!result.success) {
      throw new Error('Invalid registration response')
    }
    const data = result.data

    // 从 cookie 获取绑定的 challenge
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

    // 核心验证逻辑
    const verification = await verifyRegistrationResponse({
      response: data,
      expectedChallenge: challenge,      // 匹配 cookie 中的 challenge
      expectedOrigin: origin,          // 验证源
      expectedRPID: rpID,               // 验证依赖方
      requireUserVerification: true,    // 强制用户验证
    })

    const { verified, registrationInfo } = verification
    if (!verified || !registrationInfo) {
      throw new Error('Registration verification failed')
    }
    
    const { credential, credentialDeviceType, credentialBackedUp, aaguid } =
      registrationInfo

    // 防重放：检查凭据是否已注册
    const existingPasskey = await prisma.passkey.findUnique({
      where: { id: credential.id },
      select: { id: true },
    })
    if (existingPasskey) {
      throw new Error('This passkey has already been registered')
    }

    // 持久化存储凭据
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
    // ... 错误处理
  }
}
```

### 3.3 前端触发 (`passkeys.tsx:98-124`)

```typescript
async function handlePasskeyRegistration() {
  try {
    setError(null)
    // 1. 获取注册选项
    const resp = await fetch('/webauthn/registration')
    const jsonResult = await resp.json()
    const parsedResult = RegistrationOptionsSchema.parse(jsonResult)

    // 2. 触发浏览器认证器 API
    const regResult = await startRegistration({
      optionsJSON: parsedResult.options,
    })

    // 3. 发送响应到服务端验证
    const verificationResp = await fetch('/webauthn/registration', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(regResult),
    })

    if (!verificationResp.ok) {
      throw new Error('Failed to verify registration')
    }

    void revalidator.revalidate()
  } catch (err) {
    console.error('Failed to create passkey:', err)
    setError('Failed to create passkey. Please try again.')
  }
}
```

---

## 四、登录流程详解

### 4.1 流程图

```
┌──────────┐  GET /webauthn/authentication        ┌──────────────┐
│  浏览器  │ ─────────────────────────────────> │   服务端    │
│ (匿名用户)│                                    │              │
└──────────┘                                    └──────┬───────┘
                                                     │
       ┌─────────────────────────────────────────────┘
       │ 1. generateAuthenticationOptions() 生成选项
       │    - 无需用户信息 (可发现凭据)
       │ 2. 将 challenge 存入加密 cookie
       └─────────────────────────────────────────────┐
                                                     │
┌──────────┐     返回 AuthenticationOptions      │
│  浏览器  │ <─────────────────────────────────── ┘
│          │
└────┬─────┘
     │
     │ 用户选择 passkey (可发现凭据自动填充或用户选择)
     │ 浏览器调用 navigator.credentials.get()
     │ 设备验证用户身份
     │
┌────┴─────┐  POST /webauthn/authentication     ┌──────────────┐
│  浏览器  │ ─────────────────────────────────> │   服务端    │
│          │    (携带 AuthenticationResponse)     │              │
└──────────┘                                    └──────┬───────┘
                                                     │
       ┌─────────────────────────────────────────────┘
       │ 1. 从 cookie 读取 challenge
       │ 2. 根据 authResponse.id 查询 Passkey 记录
       │ 3. verifyAuthenticationResponse() 验证签名
       │    - 使用存储的 publicKey 验证签名
       │    - 验证 counter (防重放/克隆)
       │ 4. 更新 counter 到数据库
       │ 5. 创建新 Session
       │ 6. 清除 challenge cookie
       └─────────────────────────────────────────────┐
                                                     │
┌──────────┐     返回成功 + 重定向地址              │
│  浏览器  │ <──────────────────────────────────────┘
│          │
└──────────┘
```

### 4.2 服务端实现细节

#### 阶段 1: 生成认证选项 (`authentication.ts:15-27`)

```typescript
export async function loader({ request }: Route.LoaderArgs) {
  const config = getWebAuthnConfig(request)
  const options = await generateAuthenticationOptions({
    rpID: config.rpID,
    userVerification: 'preferred',
    // 注意：没有 allowCredentials - 使用可发现凭据
  })

  const cookieHeader = await passkeyCookie.serialize({
    challenge: options.challenge,
    // 注意：没有 userId - 登录时用户未知
  })

  return Response.json({ options }, { headers: { 'Set-Cookie': cookieHeader } })
}
```

**与注册的关键差异：

| 差异点 | 注册 | 登录 |
|--------|------|------|
| 用户状态 | 必须已登录 (`requireUserId`) | 匿名用户 |
| `allowCredentials` | 不适用 | 未指定 (可发现凭据) | 排除已有凭据 |
| Cookie 内容 | challenge + userId | 仅 challenge |
| 用户信息 | 需要 userName, userID, displayName | 无需提供 |

#### 阶段 2: 验证认证响应 (`authentication.ts:29-113`)

```typescript
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

    // 关键：根据凭据 ID 查找 passkey 记录
    const passkey = await prisma.passkey.findUnique({
      where: { id: authResponse.id },
      include: { user: true },
    })
    if (!passkey) {
      throw new Error('Passkey not found')
    }

    const config = getWebAuthnConfig(request)

    // 核心验证：需要存储的公钥和计数器
    const verification = await verifyAuthenticationResponse({
      response: authResponse,
      expectedChallenge: cookie.challenge,
      expectedOrigin: config.origin,
      expectedRPID: config.rpID,
      credential: {
        id: authResponse.id,
        publicKey: passkey.publicKey,    // 使用存储的公钥验证签名
        counter: Number(passkey.counter), // 用于防重放攻击
      },
    })

    if (!verification.verified) {
      throw new Error('Authentication verification failed')
    }

    // 更新计数器 (防止克隆攻击)
    await prisma.passkey.update({
      where: { id: passkey.id },
      data: { counter: BigInt(verification.authenticationInfo.newCounter) },
    })

    // 创建会话，登录成功
    const session = await prisma.session.create({
      select: { id: true, expirationDate: true, userId: true },
      data: {
        expirationDate: getSessionExpirationDate(),
        userId: passkey.userId,
      },
    })

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
    // ... 错误处理
  }
}
```

### 4.3 前端触发 (`login.tsx:238-277`)

```typescript
async function handlePasskeyLogin() {
  try {
    setPasskeyMessage('Generating Authentication Options')
    // 1. 获取认证选项
    const optionsResponse = await fetch('/webauthn/authentication')
    const json = await optionsResponse.json()
    const { options } = AuthenticationOptionsSchema.parse(json)

    setPasskeyMessage('Requesting your authorization')
    // 2. 触发浏览器认证器 API (用户选择 passkey)
    const authResponse = await startAuthentication({ optionsJSON: options })
    setPasskeyMessage('Verifying your passkey')

    // 3. 发送响应到服务端验证
    const verificationResponse = await fetch('/webauthn/authentication', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ authResponse, remember, redirectTo }),
    })

    const verificationJson = await verificationResponse.json().catch(() => ({
      status: 'error',
      error: 'Unknown error',
    }))

    const parsedResult =
      VerificationResponseSchema.safeParse(verificationJson)
    if (!parsedResult.success) {
      throw new Error(parsedResult.error.message)
    } else if (parsedResult.data.status === 'error') {
      throw new Error(parsedResult.data.error)
    }
    const { location } = parsedResult.data

    setPasskeyMessage("You're logged in! Navigating...")
    await navigate(location ?? '/')
  } catch (e) {
    const errorMessage = getErrorMessage(e)
    setError(`Failed to authenticate with passkey: ${errorMessage}`)
  }
}
```

---

## 五、注册流程 vs 登录流程：关键差异对比

### 5.1 差异总览表

| 维度 | 注册流程 (Registration) | 登录流程 (Authentication) |
|------|---------------------------|--------------------------|
| **用户状态 | 必须已登录 (认证态) | 匿名用户 (未认证) |
| **主要目标 | 创建并存储新凭据 | 验证现有凭据并建立会话 |
| **凭据来源** | 认证器新生成 | 从数据库查询 |
| **Challenge 绑定** | challenge + userId | 仅 challenge |
| **验证所需数据** | 无需预存数据 | 需要 publicKey + counter |
| **WebAuthn API** | `generateRegistrationOptions` | `generateAuthenticationOptions` |
| | `verifyRegistrationResponse` | `verifyAuthenticationResponse` |
| **浏览器 API** | `navigator.credentials.create()` | `navigator.credentials.get()` |
| **数据库操作** | INSERT Passkey 记录 | SELECT + UPDATE Passkey |
| **会话创建** | 否 (已有会话 | 是 (创建新 Session) |
| **用户识别方式** | 会话中的 userId | 凭据 ID 查询 |

### 5.2 核心安全机制差异详解

#### 1. 用户上下文的识别机制

**注册流程：**
```
用户已登录 → 会话中已有 userId
    ↓
Passkey 绑定到该 userId
    ↓
新凭据属于已知用户
```

**登录流程：**
```
用户匿名 → 无会话
    ↓
用户选择 passkey (可发现凭据)
    ↓
凭据 ID → 查询 Passkey 表
    ↓
获取关联的 userId
    ↓
识别用户身份
```

#### 2. Challenge 的生命周期管理

| 阶段 | 注册 | 登录 |
|------|------|------|
| 生成 | 绑定 `{ challenge, userId }` | `{ challenge }` |
| 存储 | 加密 Cookie (httpOnly + signed) | 加密 Cookie (httpOnly + signed) |
| 验证 | 匹配 challenge + 验证 userId 匹配 | 仅匹配 challenge |
| 清除 | 验证成功后清除 | 验证成功后清除 |
| 有效期 | 2 小时 | 2 小时 |

#### 3. 验证逻辑的核心差异

**注册验证 (`verifyRegistrationResponse`)：**
- 验证 attestationObject (包含新凭据证明)
- 验证 clientDataJSON
- 提取并返回新凭据的公钥
- **不需要**预先知道公钥 (正在创建凭据

**登录验证 (`verifyAuthenticationResponse`)：**
- 验证 authenticatorData + signature
- **需要**提供存储的公钥验证签名
- **需要**提供 counter 防重放
- 返回 newCounter 用于更新

```typescript
// 注册验证：无需预先知道凭据
const verification = await verifyRegistrationResponse({
  response: data,
  expectedChallenge: challenge,
  expectedOrigin: origin,
  expectedRPID: rpID,
  requireUserVerification: true,
  // 没有 credential 参数！
})

// 登录验证：必须提供存储的凭据信息
const verification = await verifyAuthenticationResponse({
  response: authResponse,
  expectedChallenge: cookie.challenge,
  expectedOrigin: config.origin,
  expectedRPID: config.rpID,
  credential: {
    id: authResponse.id,
    publicKey: passkey.publicKey,  // 必须！
    counter: Number(passkey.counter), // 必须！
  },
})
```

#### 4. Counter (签名计数器机制

这是登录流程独有的安全机制：

```
每次认证：
┌─────────────────────────────────────────────────────────┐
│  服务端存储 counter = 5                              │
│         ↓                                           │
│  认证器签名时使用 counter = 5                        │
│         ↓                                           │
│  认证器递增 counter → 6 (在 authenticatorData 中   │
│         ↓                                           │
│  服务端验证：newCounter (6) > storedCounter (5)    │
│         ↓                                           │
│  更新数据库 counter = 6                            │
└─────────────────────────────────────────────────────────┘
```

**安全目的：** 防止认证器被克隆。如果攻击者克隆了认证器，两次认证器的 counter 会落后于真实认证器，服务端会检测到并拒绝。

---

## 六、安全架构与威胁模型

### 6.1 安全防护机制总结

| 威胁 | 防护机制 | 实现位置 |
|------|---------|----------|
| **XSS 攻击 | httpOnly Cookie | `utils.server.ts:12` |
| **CSRF 攻击** | sameSite: 'lax' + signed cookie | `utils.server.ts:11,15` |
| **重放攻击** | Challenge 一次性使用 + 绑定验证 | registration.ts, authentication.ts |
| **中间人攻击** | origin + rpID 验证 | 服务端 verify*Response |
| **凭据克隆** | Counter 递增验证 | authentication.ts:72-75 |
| **网络窃听** | HTTPS 强制 (生产环境) | `secure: process.env.NODE_ENV === 'production'` |
| **凭据篡改** | Cookie 签名加密 | `secrets: [process.env.SESSION_SECRET]` |

### 6.2 关键安全决策分析

#### 1. Attestation Type: 'none'

```typescript
attestationType: 'none'
```

**含义：** 不要求认证器提供制造商证明 (Attestation Statement)

**权衡：**
- ✅ 隐私保护：不收集不到认证器型号信息
- ⚠️ 无法验证认证器的真实性
- ✅ 适合大多数消费级应用

**替代选项：**
- `'direct'` - 直接验证认证器制造商证明
- `'indirect'` - 间接验证 (通过隐私 CA)

#### 2. Resident Key: 'preferred'

```typescript
residentKey: 'preferred'
```

**含义：** 优先创建可发现凭据 (Discoverable Credentials)

**用户体验：**
- ✅ 无用户名登录
- ✅ 浏览器/系统级自动填充 passkey
- ⚠️ 占用认证器存储空间

#### 3. User Verification: 'preferred'

```typescript
userVerification: 'preferred'
```

**含义：** 优先要求用户验证 (生物识别/PIN)

**安全 vs 体验权衡：**
- `'required'` - 强制验证，更安全但可能兼容性差
- `'preferred'` - 推荐验证但不强制，平衡
- `'discouraged'` - 不推荐，仅用于低风险场景

---

## 七、代码位置速查

| 功能模块 | 文件路径 |
|-----------|---------|
| 注册流程服务端 | `app/routes/_auth/webauthn/registration.ts` |
| 登录流程服务端 | `app/routes/_auth/webauthn/authentication.ts` |
| WebAuthn 工具 | `app/routes/_auth/webauthn/utils.server.ts` |
| 注册前端触发 | `app/routes/settings/profile/passkeys.tsx` |
| 登录前端触发 | `app/routes/_auth/login.tsx` (PasskeyLogin 组件) |
| 数据库模型 | `prisma/schema.prisma` (Passkey model) |
| 通用认证工具 | `app/utils/auth.server.ts` |

---

## 八、关键 API 参考

### @simplewebauthn/server

| 函数 | 用途 |
|------|------|
| `generateRegistrationOptions` | 生成注册选项 |
| `verifyRegistrationResponse` | 验证注册响应 |
| `generateAuthenticationOptions` | 生成认证选项 |
| `verifyAuthenticationResponse` | 验证认证响应 |

### @simplewebauthn/browser

| 函数 | 对应浏览器 API | 用途 |
|------|---------------|------|
| `startRegistration` | `navigator.credentials.create()` | 触发注册流程 |
| `startAuthentication` | `navigator.credentials.get()` | 触发登录流程 |

---

## 九、总结

Epic Stack 的 Passkey 实现遵循 WebAuthn 标准，通过 `@simplewebauthn` 库提供的抽象层，实现了完整的无密码认证流程。

### 注册与登录的核心差异本质上是：

1. **注册 = 创建信任**：用户已认证，创建新的信任锚点 (凭据)
2. **登录 = 使用信任**：用户未认证，使用已有的信任锚点证明身份

这种设计完全符合 WebAuthn 规范，同时通过多层安全机制 (Cookie 安全、Counter 防重放、Origin 验证等) 确保了生产环境的安全性。
