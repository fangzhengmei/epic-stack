# GitHub OAuth 登录流程分析

## 1. 入口文件

**文件**: `app/routes/_auth/auth.$provider/callback.ts`

这是 GitHub OAuth 授权后的回调处理入口，所有的流程分叉判断都在此文件中进行。

---

## 2. 流程分叉总览（修正版）

```
                    ┌─────────────────────────────────┐
                    │  GitHub OAuth 回调到达 callback.ts │
                    └─────────────────┬───────────────┘
                                      │
                        ┌─────────────▼─────────────┐
                        │   1. 认证成功，获取 profile  │
                        └─────────────┬─────────────┘
                                      │
              ┌───────────────────────▼───────────────────────┐
              │         查询 existingConnection (Connection 表) │
              │              同时获取当前登录用户 userId          │
              └───────────────────────┬───────────────────────┘
                                      │
         ┌────────────────────────────┼────────────────────────────┐
         │                            │                            │
   ┌─────▼─────┐              ┌──────▼──────┐              ┌──────▼──────┐
   │ 已登录且   │              │  Connection  │              │  都不满足    │
   │ Connection │              │   已存在     │              │(未登录,无连接)│
   │  已存在    │              │  (用户未登录) │              │             │
   └─────┬─────┘              └──────┬──────┘              └──────┬──────┘
         │                            │                            │
    ┌────▼────┐                  ┌────▼────┐         ┌────────────▼────────────┐
    │判断 user │                  │makeSession│        │查询是否有邮箱匹配的用户    │
    │ID 相同?  │                  │          │         │(通过 profile.email)      │
    └────┬────┘                  └────┬────┘         └────────────┬────────────┘
         │                            │                            │
   ┌─────▼─────┐               ┌─────▼─────┐              ┌───────▼───────┐
   │ 相同       │   不同        │handleNew- │              │ 邮箱匹配?      │
   └─────┬─────┴───────┐       │ Session  │              └───────┬───────┘
         │             │       └─────┬─────┘                      │
    ┌────▼────┐   ┌────▼────┐       │                    ┌────────▼────────┐
    │ 已连接   │   │ 已连接   │  ┌────▼────┐    ┌────────│      是         │──────────┐
    │ 提示    │   │ 到其他   │  │检查是否  │    │        └─────────────────┘          │
    └────┬────┘   │ 账号    │  │有2FA?    │    │                                   │
         │        └────┬────┘  └────┬────┘    │                              ┌───────▼───────┐
    ┌────▼────┐        │         │          ┌───▼───┐                         │       否       │
    │ 跳转    │   ┌────▼────┐   ┌───▼───┐   │ make  │                         └───────┬───────┘
    │/settings│   │ 跳转    │   │ 有2FA │   │Session│                                 │
    │         │   │/settings│   └───┬───┘   └───┬───┘                         ┌─────────▼─────────┐
    └────┬────┘   └────┬────┘       │           │                             │  新用户，进入     │
         │             │      ┌──────▼──────┐    │                             │    Onboarding     │
    ┌────▼────┐   ┌────▼────┐│  重定向到   │    │                             └─────────┬─────────┘
    │ 结束    │   │ 结束    ││  /verify?   │    │                                       │
    └────┬────┘   └────┬────┘│  type=2fa   │    │                                ┌─────▼─────┐
         │             │     └──────┬──────┘    │                                │  跳转     │
    ┌────▼────┐   ┌────▼────┐      │           │                                │/onboarding│
    │         │   │         │      │           │                                │/github    │
    └─────────┘   └─────────┘      │           │                                └─────┬─────┘
                                     │           │                                      │
                              ┌──────▼──────┐    │                               ┌──────▼──────┐
                              │  用户输入   │    │                               │  Onboarding │
                              │  2FA 验证码 │    │                               │   流程      │
                              └──────┬──────┘    │                               └──────┬──────┘
                                     │           │                                      │
                              ┌──────▼──────┐    │                               ┌──────▼──────┐
                              │ 验证通过后   │    │                               │signupWith-  │
                              │ 设置        │    │                               │ Connection  │
                              │ authSession │    │                               │ 创建用户    │
                              └──────┬──────┘    │                               └──────┬──────┘
                                     │           │                                      │
                              ┌──────▼──────┐    │                               ┌──────▼──────┐
                              │  销毁       │    │                               │ 检查是否    │
                              │verifySession│    │                               │ 有2FA?      │
                              └──────┬──────┘    │                               └──────┬──────┘
                                     │           │                                      │
                              ┌──────▼──────┐    │                         ┌────────────┴────────────┐
                              │  登录成功    │    │                         │                         │
                              │  跳转首页    │◄───┘              ┌──────────▼──────────┐    ┌────▼────┐
                              └─────────────┘                    │      有 2FA          │    │ 无 2FA  │
                                                                 └──────────┬──────────┘    └────┬────┘
                                                                            │                    │
                                                                   ┌────────▼────────┐    ┌──────▼──────┐
                                                                   │ 重定向到        │    │  登录成功    │
                                                                   │ /verify?type=2fa│    │  跳转首页    │
                                                                   └────────┬────────┘    └─────────────┘
                                                                            │
                                                                   ┌────────▼────────┐
                                                                   │  用户输入 2FA   │
                                                                   │  验证码         │
                                                                   └────────┬────────┘
                                                                            │
                                                                   ┌────────▼────────┐
                                                                   │ 验证通过后设置   │
                                                                   │ authSession     │
                                                                   └────────┬────────┘
                                                                            │
                                                                   ┌────────▼────────┐
                                                                   │  登录成功       │
                                                                   │  跳转首页       │
                                                                   └─────────────────┘
```

---

## 3. 详细流程分叉判断

### 3.1 前置准备（callback.ts 第 40-78 行）

```typescript
// 1. 执行 OAuth 认证，获取用户 profile
const authResult = await authenticator.authenticate(providerName, request)

// 2. 查询是否已存在 Connection 记录
const existingConnection = await prisma.connection.findUnique({
    select: { userId: true },
    where: {
        providerName_providerId: {
            providerName,
            providerId: String(profile.id),
        },
    },
})

// 3. 获取当前登录用户 ID（如果用户已登录）
const userId = await getUserId(request)
```

---

### 3.2 分叉 1：已登录用户 + Connection 已存在（第 82-102 行）

**判断条件**: `existingConnection && userId`

```typescript
if (existingConnection && userId) {
    if (existingConnection.userId === userId) {
        // 子分叉 1a: Connection 属于当前登录用户
        // 行为：提示"Already Connected"，跳转到 /settings/profile/connections
        return redirectWithToast(
            '/settings/profile/connections',
            {
                title: 'Already Connected',
                description: `Your "${profile.username}" ${label} account is already connected.`,
            },
            { headers: destroyRedirectTo },
        )
    } else {
        // 子分叉 1b: Connection 属于其他用户
        // 行为：提示该 GitHub 账号已连接到其他账号
        return redirectWithToast(
            '/settings/profile/connections',
            {
                title: 'Already Connected',
                description: `The "${profile.username}" ${label} account is already connected to another account.`,
            },
            { headers: destroyRedirectTo },
        )
    }
}
```

**场景**: 用户已经登录了系统，同时这个 GitHub 账号之前已经关联过某个账号。

---

### 3.3 分叉 2：已登录用户 + Connection 不存在（第 104-122 行）

**判断条件**: `userId` 存在（用户已登录），但 `existingConnection` 不存在

```typescript
if (userId) {
    // 创建新的 Connection 记录，关联到当前登录用户
    await prisma.connection.create({
        data: {
            providerName,
            providerId: String(profile.id),
            userId,
        },
    })
    // 行为：跳转到连接设置页面，提示连接成功
    return redirectWithToast(
        '/settings/profile/connections',
        {
            title: 'Connected',
            type: 'success',
            description: `Your "${profile.username}" ${label} account has been connected.`,
        },
        { headers: destroyRedirectTo },
    )
}
```

**场景**: 用户已登录系统，现在想要绑定 GitHub 账号到当前账号。

**结果**: 直接创建 Connection 记录，不进入 Onboarding，**也不经过 2FA 检查**（因为用户已经登录）。

---

### 3.4 分叉 3：Connection 已存在（用户未登录）（第 124-127 行）

**判断条件**: `existingConnection` 存在，但 `userId` 不存在（用户未登录）

```typescript
if (existingConnection) {
    // 行为：调用 makeSession，内部会检查 2FA
    return makeSession({ request, userId: existingConnection.userId })
}
```

**场景**: 用户之前已经用这个 GitHub 账号注册过，现在只是重新登录。

**⚠️ 重要修正**: 这里**不是直接登录**，而是调用 `makeSession`，`makeSession` 内部会调用 `handleNewSession` 来检查 2FA。

---

### 3.5 分叉 4：邮箱匹配已有用户（第 131-152 行）

**判断条件**: 
- `existingConnection` 不存在
- 但 `profile.email` 在 User 表中存在

```typescript
const user = await prisma.user.findUnique({
    select: { id: true },
    where: { email: profile.email.toLowerCase() },
})
if (user) {
    // 创建 Connection 关联到该已有用户
    await prisma.connection.create({
        data: {
            providerName,
            providerId: String(profile.id),
            userId: user.id,
        },
    })
    // 行为：调用 makeSession，内部会检查 2FA
    return makeSession(
        { request, userId: user.id },
        {
            headers: await createToastHeaders({
                title: 'Connected',
                description: `Your "${profile.username}" ${label} account has been connected.`,
            }),
        },
    )
}
```

**场景**: 
- 用户之前用邮箱注册过账号（比如传统的用户名+密码注册）
- 现在用同一个邮箱的 GitHub 账号登录
- 系统检测到邮箱匹配，自动将 GitHub 账号连接到已有账号

**⚠️ 重要修正**: 这里**不是直接登录**，而是调用 `makeSession`，`makeSession` 内部会调用 `handleNewSession` 来检查 2FA。

---

### 3.6 分叉 5：全新用户 - 进入 Onboarding（第 154-177 行）

**判断条件**: 以上所有条件都不满足

```typescript
// 这是一个新用户，进入 Onboarding 流程
const verifySession = await verifySessionStorage.getSession()

// 1. 保存邮箱到 session
verifySession.set(onboardingEmailSessionKey, profile.email)

// 2. 保存预填充的 profile 信息（用于表单自动填充）
verifySession.set(prefilledProfileKey, {
    ...profile,
    email: normalizeEmail(profile.email),
    username:
        typeof profile.username === 'string'
            ? normalizeUsername(profile.username)
            : undefined,
})

// 3. 保存 providerId
verifySession.set(providerIdKey, profile.id)

// 4. 构建重定向 URL，跳转到 /onboarding/github
const onboardingRedirect = [
    `/onboarding/${providerName}`,
    redirectTo ? new URLSearchParams({ redirectTo }) : null,
]
    .filter(Boolean)
    .join('?')

// 行为：重定向到 Onboarding 页面
return redirect(onboardingRedirect, {
    headers: combineHeaders(
        { 'set-cookie': await verifySessionStorage.commitSession(verifySession) },
        destroyRedirectTo,
    ),
})
```

**场景**: 
- 完全新的用户
- GitHub 账号之前没关联过
- 邮箱也没在系统中注册过

**结果**: 进入 Onboarding 流程。

---

## 4. 关键中间层：makeSession 与 handleNewSession

### 4.1 makeSession 函数（callback.ts 第 180-200 行）

**调用时机**: 分叉 3 和分叉 4 都会调用 `makeSession`

```typescript
async function makeSession(
    {
        request,
        userId,
        redirectTo,
    }: { request: Request; userId: string; redirectTo?: string | null },
    responseInit?: ResponseInit,
) {
    redirectTo ??= '/'
    // 1. 在数据库中创建 Session 记录
    const session = await prisma.session.create({
        select: { id: true, expirationDate: true, userId: true },
        data: {
            expirationDate: getSessionExpirationDate(),
            userId,
        },
    })
    // 2. 调用 handleNewSession 处理登录逻辑（包括 2FA 检查）
    return handleNewSession(
        { request, session, redirectTo, remember: true },
        { headers: combineHeaders(responseInit?.headers, destroyRedirectTo) },
    )
}
```

**关键点**: 
- `makeSession` 只是创建数据库中的 Session 记录
- 实际的登录逻辑（包括 2FA 检查）在 `handleNewSession` 中

---

### 4.2 handleNewSession 函数（login.server.ts 第 17-81 行）

**这是 2FA 分支的核心判断位置**

```typescript
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
    // 🔴 关键：检查用户是否设置了 2FA
    // 查询 Verification 表中是否存在 type='2fa' 的记录
    const verification = await prisma.verification.findUnique({
        select: { id: true },
        where: {
            target_type: { target: session.userId, type: twoFAVerificationType },
        },
    })
    const userHasTwoFactor = Boolean(verification)

    if (userHasTwoFactor) {
        // ┌─────────────────────────────────────────────────────────┐
        // │ 分支 A: 用户有 2FA 设置 - 进入 2FA 验证流程              │
        // └─────────────────────────────────────────────────────────┘
        
        // 1. 使用 verifySession 存储未验证的 sessionId
        const verifySession = await verifySessionStorage.getSession()
        verifySession.set(unverifiedSessionIdKey, session.id)  // 存储 'unverified-session-id'
        verifySession.set(rememberKey, remember)                // 存储 'remember'
        
        // 2. 构建 2FA 验证页面的 URL
        const redirectUrl = getRedirectToUrl({
            request,
            type: twoFAVerificationType,  // '2fa'
            target: session.userId,
            redirectTo,
        })
        
        // 3. 重定向到 /verify 页面，参数: type=2fa, target=userId, redirectTo=...
        return redirect(
            `${redirectUrl.pathname}?${redirectUrl.searchParams}`,
            combineResponseInits(
                {
                    headers: {
                        'set-cookie':
                            await verifySessionStorage.commitSession(verifySession),
                    },
                },
                responseInit,
            ),
        )
    } else {
        // ┌─────────────────────────────────────────────────────────┐
        // │ 分支 B: 用户无 2FA 设置 - 直接完成登录                    │
        // └─────────────────────────────────────────────────────────┘
        
        // 1. 直接设置 authSession
        const authSession = await authSessionStorage.getSession(
            request.headers.get('cookie'),
        )
        authSession.set(sessionKey, session.id)  // 直接将 sessionId 存入 authSession

        // 2. 重定向到首页或目标页面
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

---

## 5. 2FA 验证完整流程

### 5.1 2FA 触发条件

**2FA 检查会在以下路径触发**:

| 路径 | 触发位置 | 2FA 检查 |
|------|----------|----------|
| 分叉 3: Connection 已存在 | `callback.ts:126` → `makeSession` → `handleNewSession` | ✅ 会检查 |
| 分叉 4: 邮箱匹配已有用户 | `callback.ts:143` → `makeSession` → `handleNewSession` | ✅ 会检查 |
| Onboarding 完成后 | `$provider.tsx:action` → `signupWithConnection` → 返回 session → 手动设置 authSession | ⚠️ 不会检查 |

**⚠️ 重要发现**: Onboarding 完成后的登录**不会经过 `handleNewSession`**，而是直接在 `$provider.tsx` 的 action 中设置 `authSession`。这意味着：
- 新用户通过 OAuth 注册时，即使未来设置了 2FA，这次注册流程不会触发 2FA
- 但新用户注册完成后，**可以立即去设置 2FA**

### 5.2 2FA 验证页面（verify.tsx）

**路由**: `/verify?type=2fa&target=userId&redirectTo=...`

```typescript
// verify.tsx 第 62-69 行
const headings: Record<VerificationTypes, React.ReactNode> = {
    // ... 其他类型
    '2fa': (
        <>
            <h1 className="text-h1">Check your 2FA app</h1>
            <p className="text-body-md text-muted-foreground mt-3">
                Please enter your 2FA code to verify your identity.
            </p>
        </>
    ),
}
```

### 5.3 2FA 验证处理（login.server.ts 第 83-137 行）

```typescript
export async function handleVerification({
    request,
    submission,
}: VerifyFunctionArgs) {
    invariant(
        submission.status === 'success',
        'Submission should be successful by now',
    )
    
    // 1. 获取两个 session
    const authSession = await authSessionStorage.getSession(
        request.headers.get('cookie'),
    )
    const verifySession = await verifySessionStorage.getSession(
        request.headers.get('cookie'),
    )

    // 2. 从 verifySession 读取之前存储的数据
    const remember = verifySession.get(rememberKey)           // 'remember'
    const { redirectTo } = submission.value
    const headers = new Headers()
    
    // 记录验证时间（用于后续的 2FA 重新验证判断）
    authSession.set(verifiedTimeKey, Date.now())              // 'verified-time'

    // 3. 获取未验证的 sessionId
    const unverifiedSessionId = verifySession.get(unverifiedSessionIdKey)  // 'unverified-session-id'
    
    if (unverifiedSessionId) {
        // 验证 session 仍然有效
        const session = await prisma.session.findUnique({
            select: { expirationDate: true },
            where: { id: unverifiedSessionId },
        })
        if (!session) {
            throw await redirectWithToast('/login', {
                type: 'error',
                title: 'Invalid session',
                description: 'Could not find session to verify. Please try again.',
            })
        }
        
        // 🔴 关键：将 unverifiedSessionId 设置到 authSession
        // 这才真正完成登录！
        authSession.set(sessionKey, unverifiedSessionId)

        headers.append(
            'set-cookie',
            await authSessionStorage.commitSession(authSession, {
                expires: remember ? session.expirationDate : undefined,
            }),
        )
    } else {
        headers.append(
            'set-cookie',
            await authSessionStorage.commitSession(authSession),
        )
    }

    // 4. 清理 verifySession
    headers.append(
        'set-cookie',
        await verifySessionStorage.destroySession(verifySession),
    )

    // 5. 跳转到目标页面
    return redirect(safeRedirect(redirectTo), { headers })
}
```

---

### 5.4 2FA 流程的 Session 流转

```
┌────────────────────────────────────────────────────────────────────────────┐
│                     2FA 验证流程的 Session 状态变化                          │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  阶段 1: handleNewSession (检测到 2FA)                                     │
│  ───────────────────────────────────────────────────────────────────────  │
│                                                                            │
│    authSession:                                                            │
│      ┌────────────────────────────────────────────────────────────────┐  │
│      │  (空 - 没有设置 sessionKey)                                      │  │
│      └────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│    verifySession (新建):                                                   │
│      ┌────────────────────────────────────────────────────────────────┐  │
│      │  unverifiedSessionIdKey: 'session_xxx'  ← 存储数据库中的 sessionId │  │
│      │  rememberKey: true                                              │  │
│      └────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│    操作: 重定向到 /verify?type=2fa&target=userId                          │
│          携带 verifySession 的 cookie                                      │
│                                                                            │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  阶段 2: 用户在 /verify 页面输入 2FA 验证码                                 │
│  ───────────────────────────────────────────────────────────────────────  │
│                                                                            │
│    用户操作: 输入 6 位验证码，提交表单                                       │
│                                                                            │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  阶段 3: handleVerification (验证通过后)                                    │
│  ───────────────────────────────────────────────────────────────────────  │
│                                                                            │
│    读取 verifySession:                                                      │
│      ┌────────────────────────────────────────────────────────────────┐  │
│      │  unverifiedSessionIdKey: 'session_xxx'                          │  │
│      │  rememberKey: true                                              │  │
│      └────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│    更新 authSession:                                                        │
│      ┌────────────────────────────────────────────────────────────────┐  │
│      │  sessionKey: 'session_xxx'  ← 🔴 关键：现在真正登录了！          │  │
│      │  verifiedTimeKey: 1714876800000  ← 记录验证时间                 │  │
│      └────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│    销毁 verifySession:                                                      │
│      ┌────────────────────────────────────────────────────────────────┐  │
│      │  (已清除)                                                         │  │
│      └────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│    操作: 重定向到 redirectTo (通常是首页)                                   │
│          携带更新后的 authSession cookie                                   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

### 5.5 2FA 触发条件总结

**哪些路径会触发 2FA 检查？**

| 场景 | 代码路径 | 是否触发 2FA |
|------|----------|-------------|
| 已有账号通过 GitHub 重新登录 | `callback.ts:126` → `makeSession` → `handleNewSession` | ✅ 会 |
| 邮箱匹配已有账号（自动连接） | `callback.ts:143` → `makeSession` → `handleNewSession` | ✅ 会 |
| 传统用户名密码登录 | `login.tsx:77` → `handleNewSession` | ✅ 会 |
| WebAuthn 登录 | `webauthn/authentication.ts:85` → `handleNewSession` | ✅ 会 |
| **全新用户 OAuth 注册（Onboarding）** | `$provider.tsx:action` → 直接设置 `authSession` | ❌ **不会** |

**为什么 Onboarding 后不会触发 2FA？**

查看 `onboarding/$provider.tsx` 的 action 函数（第 137-159 行）：

```typescript
const { session, remember, redirectTo } = submission.value

// 🔴 直接设置 authSession，绕过了 handleNewSession
const authSession = await authSessionStorage.getSession(
    request.headers.get('cookie'),
)
authSession.set(sessionKey, session.id)
const headers = new Headers()
headers.append(
    'set-cookie',
    await authSessionStorage.commitSession(authSession, {
        expires: remember ? session.expirationDate : undefined,
    }),
)
headers.append(
    'set-cookie',
    await verifySessionStorage.destroySession(verifySession),
)

return redirectWithToast(...)
```

**设计意图**: 
- 全新用户不可能已经设置了 2FA（2FA 需要登录后才能设置）
- 所以 Onboarding 流程不需要 2FA 检查
- 用户注册完成后，可以去设置页面开启 2FA

---

## 6. Onboarding 流程分析

### 6.1 两种 Onboarding 路由

| 路由 | 文件 | 用途 | 2FA 检查 |
|------|------|------|----------|
| `/onboarding` | `onboarding/index.tsx` | 传统注册（邮箱验证后），需要设置密码 | ❌ 不会（新用户） |
| `/onboarding/:provider` | `onboarding/$provider.tsx` | OAuth 注册，不需要设置密码 | ❌ 不会（新用户） |

### 6.2 OAuth Onboarding 流程（$provider.tsx）

**入口路由**: `/onboarding/github`

#### 6.2.1 Loader - 数据准备（第 77-92 行）

```typescript
export async function loader({ request, params }: Route.LoaderArgs) {
    // 1. 验证用户必须是匿名的（未登录）
    // 2. 从 verifySession 中获取 email、providerId、providerName
    const { email } = await requireData({ request, params })

    // 3. 获取预填充的 profile 信息
    const verifySession = await verifySessionStorage.getSession(
        request.headers.get('cookie'),
    )
    const prefilledProfile = verifySession.get(prefilledProfileKey)

    return {
        email,
        status: 'idle',
        submission: {
            initialValue: prefilledProfile ?? {},
        } as SubmissionResult,
    }
}
```

#### 6.2.2 表单字段（第 38-47 行）

```typescript
const SignupFormSchema = z.object({
    imageUrl: z.string().optional(),        // 头像 URL（从 GitHub 获取）
    username: UsernameSchema,                // 用户名（预填充 GitHub username）
    name: NameSchema,                        // 姓名（预填充 GitHub name）
    agreeToTermsOfServiceAndPrivacyPolicy: z.boolean(),  // 必须同意
    remember: z.boolean().optional(),        // 记住登录
    redirectTo: z.string().optional(),       // 登录后跳转
})
```

**注意**: OAuth Onboarding **没有密码字段**！

#### 6.2.3 Action - 提交处理（第 94-160 行）

```typescript
export async function action({ request, params }: Route.ActionArgs) {
    // 1. 验证数据
    const { email, providerId, providerName } = await requireData({
        request,
        params,
    })

    // 2. 解析表单数据，验证用户名唯一性
    const submission = await parseWithZod(formData, {
        schema: SignupFormSchema.superRefine(async (data, ctx) => {
            // 检查用户名是否已存在
            const existingUser = await prisma.user.findUnique({
                where: { username: data.username },
                select: { id: true },
            })
            if (existingUser) {
                ctx.addIssue({
                    path: ['username'],
                    code: z.ZodIssueCode.custom,
                    message: 'A user already exists with this username',
                })
                return
            }
        }).transform(async (data) => {
            // 3. 创建用户账号和 Connection
            const session = await signupWithConnection({
                ...data,
                email,
                providerId: String(providerId),
                providerName,
            })
            return { ...data, session }
        }),
        async: true,
    })

    // 4. 处理登录跳转
    if (submission.status !== 'success') {
        // 返回错误信息
        return data({ result: submission.reply() }, ...)
    }

    const { session, remember, redirectTo } = submission.value

    // 5. 🔴 直接设置 auth session（绕过 handleNewSession，不检查 2FA）
    const authSession = await authSessionStorage.getSession(...)
    authSession.set(sessionKey, session.id)

    // 6. 清理 verifySession，跳转成功
    return redirectWithToast(
        safeRedirect(redirectTo),
        { title: 'Welcome', description: 'Thanks for signing up!' },
        { headers },
    )
}
```

### 6.3 核心注册函数：signupWithConnection

**文件**: `app/utils/auth.server.ts` 第 151-201 行

```typescript
export async function signupWithConnection({
    email,
    username,
    name,
    providerId,
    providerName,
    imageUrl,
}: {
    email: User['email']
    username: User['username']
    name: User['name']
    providerId: Connection['providerId']
    providerName: Connection['providerName']
    imageUrl?: string
}) {
    // 1. 创建 User 记录，同时创建 Connection 记录
    const user = await prisma.user.create({
        data: {
            email: email.toLowerCase(),
            username: username.toLowerCase(),
            name,
            roles: { connect: { name: 'user' } },
            connections: { create: { providerId, providerName } },  // 关键：同时创建 Connection
        },
        select: { id: true },
    })

    // 2. 如果有头像 URL，下载并保存头像
    if (imageUrl) {
        const imageFile = await downloadFile(imageUrl)
        await prisma.user.update({
            where: { id: user.id },
            data: {
                image: {
                    create: {
                        objectKey: await uploadProfileImage(user.id, imageFile),
                    },
                },
            },
        })
    }

    // 3. 创建 Session 记录
    const session = await prisma.session.create({
        data: {
            expirationDate: getSessionExpirationDate(),
            userId: user.id,
        },
        select: { id: true, expirationDate: true },
    })

    return session
}
```

**与传统 signup 的区别**:

| 特性 | `signupWithConnection` (OAuth) | `signup` (传统) |
|------|--------------------------------|-----------------|
| 密码 | 不需要 | 需要 |
| Connection | 同时创建 | 不创建 |
| 头像处理 | 自动下载 GitHub 头像 | 无 |

---

## 7. 关键 Session 管理

### 7.1 Session 类型

| Session 存储 | Key | 用途 |
|--------------|-----|------|
| `authSessionStorage` | `sessionKey` = 'sessionId' | 已登录用户的 Session ID |
| `authSessionStorage` | `verifiedTimeKey` = 'verified-time' | 上次 2FA 验证时间 |
| `verifySessionStorage` | `onboardingEmailSessionKey` | Onboarding 流程中的邮箱 |
| `verifySessionStorage` | `prefilledProfileKey` | OAuth 预填充的用户信息 |
| `verifySessionStorage` | `providerIdKey` | OAuth provider ID |
| `verifySessionStorage` | `unverifiedSessionIdKey` | 2FA 流程中未验证的 sessionId |
| `verifySessionStorage` | `rememberKey` | 2FA 流程中的 remember 标志 |

### 7.2 2FA 流程中的 Session Key 定义

**文件**: `login.server.ts` 第 13-15 行

```typescript
const verifiedTimeKey = 'verified-time'
const unverifiedSessionIdKey = 'unverified-session-id'
const rememberKey = 'remember'
```

### 7.3 2FA Verification Type 定义

**文件**: `settings/profile/two-factor/_layout.tsx` 第 12 行

```typescript
export const twoFAVerificationType = '2fa' satisfies VerificationTypes
```

---

## 8. 修正后的完整流程图（时序版）

```
用户操作                    GitHub                    callback.ts            handleNewSession         /verify
   │                          │                           │                        │                      │
   │  1. 点击 GitHub 登录      │                           │                        │                      │
   ├─────────────────────────►│                           │                        │                      │
   │                          │                           │                        │                      │
   │  2. 授权完成后回调        │                           │                        │                      │
   │◄─────────────────────────┤                           │                        │                      │
   │                          │                           │                        │                      │
   │  3. 携带 code 回调        │                           │                        │                      │
   ├────────────────────────────────────────────────────►│                        │                      │
   │                          │                           │  4. 交换 token         │                      │
   │                          │                           │     获取 profile        │                      │
   │                          │                           │                        │                      │
   │                          │                           │  5. 分支判断            │                      │
   │                          │                           │                        │                      │
   │                          │                           │  ┌──────────────────┐  │                      │
   │                          │                           │  │ 分叉 3 或 4?      │  │                      │
   │                          │                           │  │ (已有账号登录)     │  │                      │
   │                          │                           │  └────────┬─────────┘  │                      │
   │                          │                           │           │             │                      │
   │                          │                           │     [是]  │  [否]       │                      │
   │                          │                           │           │             │                      │
   │                          │                           │           │      全新用户 │                      │
   │                          │                           │           │      进入     │                      │
   │                          │                           │           │      Onboarding│                      │
   │                          │                           │           │             │                      │
   │                          │                           │           ▼             │                      │
   │                          │                    ┌──────▼──────┐                 │                      │
   │                          │                    │ makeSession │                 │                      │
   │                          │                    │ 创建 Session│                 │                      │
   │                          │                    └──────┬──────┘                 │                      │
   │                          │                           │                        │                      │
   │                          │                           ├───────────────────────►│                      │
   │                          │                           │                        │                      │
   │                          │                           │                        │  6. 检查 2FA?        │
   │                          │                           │                        │                      │
   │                          │                           │                        │     ┌──────┴──────┐  │
   │                          │                           │                        │     ▼             ▼  │
   │                          │                           │                        │  [有 2FA]     [无 2FA]│
   │                          │                           │                        │     │             │  │
   │                          │                           │                        │     │             │  │
   │                          │                           │                        │     │      直接设置   │
   │                          │                           │                        │     │      authSession│
   │                          │                           │                        │     │             │  │
   │                          │                           │                        │     │      登录成功   │
   │                          │                           │                        │     │      跳转首页   │
   │                          │                           │                        │     │             │  │
   │                          │                           │                        │     │             │  │
   │                          │                           │                        │  7. 设置         │  │
   │                          │                           │                        │     verifySession │
   │                          │                           │                        │     (unverified-  │
   │                          │                           │                        │      session-id)  │
   │                          │                           │                        │     │             │  │
   │                          │                           │                        │     ▼             │  │
   │◄─────────────────────────────────────────────────────────────────────────────┤  8. 重定向到      │
   │                          │                           │                        │     /verify?      │
   │                          │                           │                        │     type=2fa      │
   │                          │                           │                        │     │             │  │
   │                          │                           │                        │     │             │  │
   │  9. 加载 2FA 验证页面     │                           │                        │     │             │
   ├────────────────────────────────────────────────────────────────────────────────────────────────────────►│
   │                          │                           │                        │                     │
   │                          │                           │                        │  10. 显示 2FA 输入框 │
   │◄────────────────────────────────────────────────────────────────────────────────────────────────────────┤
   │                          │                           │                        │                     │
   │  11. 输入 6 位验证码      │                           │                        │                     │
   ├────────────────────────────────────────────────────────────────────────────────────────────────────────►│
   │                          │                           │                        │                     │
   │                          │                           │                        │  12. 验证验证码     │
   │                          │                           │                        │      从 verifySession│
   │                          │                           │                        │      读取 sessionId  │
   │                          │                           │                        │                     │
   │                          │                           │                        │  13. 设置           │
   │                          │                           │                        │      authSession    │
   │                          │                           │                        │      销毁           │
   │                          │                           │                        │      verifySession  │
   │                          │                           │                        │                     │
   │◄────────────────────────────────────────────────────────────────────────────────────────────────────────┤
   │                          │                           │                        │                     │
   │  14. 登录成功，跳转首页   │                           │                        │                     │
   │                          │                           │                        │                     │
```

---

## 9. 关键文件索引

| 文件路径 | 职责 |
|---------|------|
| `app/routes/_auth/auth.$provider/callback.ts` | OAuth 回调入口，所有流程分叉判断 |
| `app/routes/_auth/login.server.ts` | `handleNewSession` 2FA 检查核心、`handleVerification` 2FA 验证完成 |
| `app/routes/_auth/verify.tsx` | 2FA 验证页面 UI |
| `app/routes/_auth/verify.server.ts` | 验证请求分发、`getRedirectToUrl` |
| `app/routes/_auth/onboarding/$provider.tsx` | OAuth Onboarding 页面（无密码） |
| `app/routes/_auth/onboarding/index.tsx` | 传统 Onboarding 页面（需密码） |
| `app/utils/auth.server.ts` | `signupWithConnection`、`signup` 等核心函数 |
| `app/routes/settings/profile/two-factor/_layout.tsx` | `twoFAVerificationType` 定义 |

---

## 10. 修正后的总结

### 10.1 流程分叉决策树（完整准确版）

```
GitHub OAuth 回调 (callback.ts)
    │
    ├──► 已登录用户? (userId 存在)
    │       │
    │       ├──► Connection 已存在?
    │       │       │
    │       │       ├──► 是同一用户? ──► 提示"已连接"，跳转 /settings
    │       │       │
    │       │       └──► 不同用户? ───► 提示"已连接到其他账号"
    │       │
    │       └──► Connection 不存在? ──► 创建 Connection，绑定到当前账号
    │                                     (不经过 2FA 检查，因为已登录)
    │
    └──► 未登录用户? (userId 不存在)
            │
            ├──► Connection 已存在? ──► makeSession → handleNewSession
            │                                   │
            │                                   ├──► 有 2FA? ──► 重定向 /verify?type=2fa
            │                                   │                   │
            │                                   │              用户输入验证码
            │                                   │                   │
            │                                   │              handleVerification
            │                                   │                   │
            │                                   │              设置 authSession，登录成功
            │                                   │
            │                                   └──► 无 2FA? ──► 直接设置 authSession，登录成功
            │
            ├──► 邮箱匹配已有用户? ──► 创建 Connection → makeSession → handleNewSession
            │                                   │
            │                                   ├──► 有 2FA? ──► 重定向 /verify?type=2fa
            │                                   │
            │                                   └──► 无 2FA? ──► 直接登录成功
            │
            └──► 全新用户 ──► 重定向 /onboarding/github
                                    │
                                    ▼
                              Onboarding 表单
                              (username, name, 无需密码)
                                    │
                                    ▼
                              signupWithConnection()
                                    │
                                    ├──► 创建 User + Connection
                                    ├──► 下载 GitHub 头像
                                    └──► 创建 Session
                                    │
                                    ▼
                              🔴 直接设置 authSession
                              (绕过 handleNewSession)
                                    │
                                    ▼
                              登录成功，跳转首页
                              (新用户无 2FA，合理)
```

### 10.2 2FA 触发条件的准确结论

| 场景 | 是否触发 2FA | 原因 |
|------|-------------|------|
| 已有 GitHub 连接的用户重新登录 | ✅ 会 | 经过 `handleNewSession` |
| 邮箱匹配已有账号（自动连接） | ✅ 会 | 经过 `handleNewSession` |
| 全新用户 OAuth 注册 | ❌ 不会 | 直接设置 `authSession`，绕过 `handleNewSession` |
| 已登录用户绑定新 GitHub | ❌ 不会 | 用户已登录，不需要重新认证 |

### 10.3 Onboarding 判断时机

**Onboarding 只在以下条件同时满足时才会触发**（在 `callback.ts` 中判断）：

1. 用户 **未登录** (`userId` 为 null)
2. GitHub 账号 **未关联过** (`existingConnection` 为 null)
3. GitHub 邮箱 **未注册过** (`user` 为 null)

**Onboarding 执行位置**：
- **判断**: `app/routes/_auth/auth.$provider/callback.ts` 第 154-177 行
- **执行**: `app/routes/_auth/onboarding/$provider.tsx` 的 `action` 函数
- **核心创建**: `app/utils/auth.server.ts` 的 `signupWithConnection` 函数
- **⚠️ 注意**: Onboarding 完成后**不会检查 2FA**（设计上是合理的，因为新用户不可能已经设置了 2FA）

### 10.4 之前文档的不准确之处修正

| 之前的错误描述 | 正确的描述 |
|---------------|-----------|
| "直接登录成功，跳转到首页" | "调用 `makeSession`，内部会通过 `handleNewSession` 检查 2FA" |
| 遗漏 2FA 分支 | 补充完整的 2FA 验证流程，包括 Session 流转和页面跳转 |
| 未区分不同登录路径的 2FA 行为 | 明确列出哪些路径会触发 2FA，哪些不会 |
