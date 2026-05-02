# Epic Stack 会话与权限链路分析（核对稿）

## 1. 关键口径核对清单

### 1.1 近期复验时长（已核对）

**代码位置**：`app/routes/_auth/login.server.ts:149-157`

```typescript
// if it's over two hours since they last verified, we should request 2FA again
const userHasTwoFA = await prisma.verification.findUnique({
	select: { id: true },
	where: { target_type: { target: userId, type: twoFAVerificationType } },
})
if (!userHasTwoFA) return false
const verifiedTime = authSession.get(verifiedTimeKey) ?? new Date(0)
const twoHours = 1000 * 60 * 2
return Date.now() - verifiedTime > twoHours
```

**准确结论**：
- 复验触发条件：**当前时间 - 上次验证时间 > 2 小时**
- 即：**超过 2 小时**才需要重新验证
- 不是"2 小时内需要验证"，而是"超过 2 小时需要重新验证"

### 1.2 登录流程分支条件（已核对）

**代码位置**：`app/routes/_auth/login.server.ts:31-80`

| 条件 | 行为 | Cookie 设置 |
|------|------|--------------|
| 用户**没有**启用 2FA | 直接建立正式会话 | 设置 `en_session` Cookie |
| 用户**有**启用 2FA | 进入 2FA 验证流程 | **不设置** `en_session`，只设置 `en_verification` |

**关键代码**：
```typescript
// 有 2FA 时（第 39-60 行）
if (userHasTwoFactor) {
	const verifySession = await verifySessionStorage.getSession()  // 注意：不传 cookie 参数！
	verifySession.set(unverifiedSessionIdKey, session.id)
	verifySession.set(rememberKey, remember)
	// ...
	return redirect(
		// ...
		{
			headers: {
				'set-cookie':
					await verifySessionStorage.commitSession(verifySession),  // 只设置 en_verification
			},
		},
	)
}

// 无 2FA 时（第 61-80 行）
else {
	const authSession = await authSessionStorage.getSession(
		request.headers.get('cookie'),
	)
	authSession.set(sessionKey, session.id)
	// ...
	return redirect(
		// ...
		{
			headers: {
				'set-cookie': await authSessionStorage.commitSession(authSession, {
					expires: remember ? session.expirationDate : undefined,
				}),  // 设置 en_session
			},
		},
	)
}
```

### 1.3 二次验证流程状态转换（已核对）

#### 阶段 1：密码验证后（临时状态）

**代码位置**：`app/routes/_auth/login.server.ts:39-60`

| 项目 | 状态 |
|------|------|
| 数据库会话 | 已创建（`prisma.session.create()`） |
| `en_session` Cookie | **未设置** |
| `en_verification` Cookie | 已设置 |
| `verifySession` 内容 | `unverified-session-id`, `remember` |
| 用户登录状态 | **未登录**（没有 `en_session`） |

**关键点**：`verifySessionStorage.getSession()` **没有传 cookie 参数**，创建的是一个**新的空会话**。

#### 阶段 2：2FA 代码验证通过后（正式登录）

**代码位置**：`app/routes/_auth/login.server.ts:83-137`

```typescript
export async function handleVerification({
	request,
	submission,
}: VerifyFunctionArgs) {
	// ...
	const authSession = await authSessionStorage.getSession(
		request.headers.get('cookie'),
	)
	const verifySession = await verifySessionStorage.getSession(
		request.headers.get('cookie'),  // 从请求读取 cookie
	)

	authSession.set(verifiedTimeKey, Date.now())  // 记录验证时间

	const unverifiedSessionId = verifySession.get(unverifiedSessionIdKey)
	if (unverifiedSessionId) {
		// 登录场景：有 unverifiedSessionId
		const session = await prisma.session.findUnique({
			select: { expirationDate: true },
			where: { id: unverifiedSessionId },
		})
		// ...
		authSession.set(sessionKey, unverifiedSessionId)  // 关键：转移会话 ID

		headers.append(
			'set-cookie',
			await authSessionStorage.commitSession(authSession, {
				expires: remember ? session.expirationDate : undefined,
			}),  // 设置 en_session Cookie
		)
	} else {
		// 已登录用户的敏感操作场景：没有 unverifiedSessionId
		headers.append(
			'set-cookie',
			await authSessionStorage.commitSession(authSession),  // 只更新，不设置 sessionKey
		)
	}

	headers.append(
		'set-cookie',
		await verifySessionStorage.destroySession(verifySession),  // 清除 en_verification
	)

	return redirect(safeRedirect(redirectTo), { headers })
}
```

| 项目 | 状态 |
|------|------|
| `en_session` Cookie | **已设置**（包含 `sessionId`） |
| `en_verification` Cookie | **已销毁** |
| `authSession` 内容 | `sessionId`, `verified-time` |
| 用户登录状态 | **已登录** |

### 1.4 角色校验与权限校验层级（已核对）

#### 调用链结构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              第一层：基础会话验证                               │
│  getUserId(request)                                                           │
│    ├─ 无 sessionId → 返回 null                                                 │
│    └─ 有 sessionId → 验证数据库会话                                           │
│         ├─ 会话有效 → 返回 userId                                              │
│         └─ 会话无效 → 抛出重定向到首页（清除 cookie）                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              第二层：登录状态校验                               │
│  requireUserId(request)                                                       │
│    └─ 调用 getUserId()                                                         │
│         ├─ 已登录 → 返回 userId                                                │
│         └─ 未登录 → 抛出重定向到登录页                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
┌───────────────────────┐ ┌───────────────────────┐ ┌───────────────────────┐
│   第三层：角色校验     │ │  第三层：权限校验      │ │ 第三层：近期验证校验   │
│                       │ │                       │ │                       │
│ requireUserWithRole() │ │ requireUserWithPermi- │ │ requireRecentVerifi-  │
│                       │ │ ssion()               │ │ cation()              │
│   └─ requireUserId()  │ │   └─ requireUserId()  │ │   └─ requireUserId()  │
│        └─ 检查角色    │ │        └─ 检查权限    │ │        └─ should-     │
│              ├─ 有    │ │              ├─ 有    │ │           RequestTwoFA│
│              │   → OK │ │              │   → OK │ │              ├─ 需   │
│              └─ 无    │ │              └─ 无    │ │              │   要   │
│                  →403 │ │                  →403 │ │              │   重   │
│                       │ │                       │ │              │   新   │
│                       │ │                       │ │              └─ 不需   │
│                       │ │                       │ │                  → OK   │
└───────────────────────┘ └───────────────────────┘ └───────────────────────┘
```

#### 各函数代码核对

**`requireUserWithRole`** (`app/utils/permissions.server.ts:43-60`)：
```typescript
export async function requireUserWithRole(request: Request, name: string) {
	const userId = await requireUserId(request)  // 先调用 requireUserId
	const user = await prisma.user.findFirst({
		select: { id: true },
		where: { id: userId, roles: { some: { name } } },
	})
	if (!user) {
		throw data(
			{
				error: 'Unauthorized',
				requiredRole: name,
				message: `Unauthorized: required role: ${name}`,
			},
			{ status: 403 },
		)
	}
	return user.id
}
```

**`requireUserWithPermission`** (`app/utils/permissions.server.ts:6-41`)：
```typescript
export async function requireUserWithPermission(
	request: Request,
	permission: PermissionString,
) {
	const userId = await requireUserId(request)  // 先调用 requireUserId
	const permissionData = parsePermissionString(permission)
	const user = await prisma.user.findFirst({
		// ... 权限查询逻辑
	})
	if (!user) {
		throw data(
			{
				error: 'Unauthorized',
				requiredPermission: permissionData,
				message: `Unauthorized: required permissions: ${permission}`,
			},
			{ status: 403 },
		)
	}
	return user.id
}
```

**`requireRecentVerification`** (`app/routes/_auth/verify.server.ts:58-74`)：
```typescript
export async function requireRecentVerification(request: Request) {
	const userId = await requireUserId(request)  // 先调用 requireUserId
	const shouldReverify = await shouldRequestTwoFA(request)
	if (shouldReverify) {
		// 重定向到 2FA 验证页
	}
}
```

---

## 2. 实际受保护路由示例分析

### 2.1 Admin 路由（角色校验）

**文件路径**：`app/routes/admin/cache/index.tsx`

#### Loader 校验（第 34-57 行）

```typescript
export async function loader({ request }: Route.LoaderArgs) {
	await requireUserWithRole(request, 'admin')  // 角色校验
	const searchParams = new URL(request.url).searchParams
	// ... 加载缓存管理页面数据
}
```

#### Action 校验（第 59-86 行）

```typescript
export async function action({ request }: Route.ActionArgs) {
	await requireUserWithRole(request, 'admin')  // 角色校验
	const formData = await request.formData()
	// ... 执行缓存操作
}
```

**校验特点**：
- 校验层级：`requireUserWithRole` → `requireUserId` → `getUserId`
- 挂载点：`loader` 和 `action` **都调用**
- 目的：确保只有 admin 角色的用户才能访问和操作

### 2.2 Notes 路由（权限校验）

**文件路径**：`app/routes/users/$username/notes/$noteId.tsx`

#### Loader（第 24-49 行）

```typescript
export async function loader({ params }: Route.LoaderArgs) {
	// 注意：没有调用任何身份验证函数！
	const note = await prisma.note.findUnique({
		where: { id: params.noteId },
		select: {
			id: true,
			title: true,
			content: true,
			ownerId: true,
			// ...
		},
	})
	// ...
}
```

#### Action（第 56-90 行）

```typescript
export async function action({ request }: Route.ActionArgs) {
	const userId = await requireUserId(request)  // 第一层：要求登录
	const formData = await request.formData()
	// ...
	
	const note = await prisma.note.findFirst({
		select: { id: true, ownerId: true, owner: { select: { username: true } } },
		where: { id: noteId },
	})
	// ...
	
	const isOwner = note.ownerId === userId
	await requireUserWithPermission(
		request,
		isOwner ? `delete:note:own` : `delete:note:any`,  // 第二层：权限校验
	)
	
	await prisma.note.delete({ where: { id: note.id } })
	// ...
}
```

**校验特点**：
- **Loader**：**不调用**任何身份验证函数
  - 笔记详情页面是**公开访问**的
  - 任何人都可以查看笔记内容
- **Action**：
  - 第一层：`requireUserId` → 要求登录
  - 第二层：`requireUserWithPermission` → 根据是否是所有者检查不同权限
    - 所有者：需要 `delete:note:own` 权限
    - 非所有者：需要 `delete:note:any` 权限（通常只有管理员有）

### 2.3 Change Email 路由（近期验证校验）

**文件路径**：
- `app/routes/settings/profile/change-email.tsx`
- `app/routes/settings/profile/change-email.server.tsx`

#### Loader 校验（第 34-46 行）

```typescript
export async function loader({ request }: Route.LoaderArgs) {
	await requireRecentVerification(request)  // 第一层：近期验证校验
	const userId = await requireUserId(request)  // 第二层：再次要求登录（虽然前面已经检查过）
	const user = await prisma.user.findUnique({
		where: { id: userId },
		select: { email: true },
	})
	// ...
}
```

#### Action（第 48-100 行）

```typescript
export async function action({ request }: Route.ActionArgs) {
	const userId = await requireUserId(request)  // 只要求登录
	const formData = await request.formData()
	// ... 发送验证邮件到新邮箱
}
```

#### Handle Verification（第 14-69 行）

```typescript
export async function handleVerification({
	request,
	submission,
}: VerifyFunctionArgs) {
	await requireRecentVerification(request)  // 近期验证校验
	// ... 执行邮箱变更
}
```

**校验特点**：
- **Loader**：
  - 第一层：`requireRecentVerification` → 检查是否需要重新验证
    - 如果启用了 2FA 且超过 2 小时未验证 → 重定向到 2FA 验证页
  - 第二层：`requireUserId` → 再次要求登录（冗余但安全）
- **Action**：
  - 只调用 `requireUserId`
  - 因为 action 是处理"请求修改邮箱"，提交新邮箱后会发送验证邮件
  - 真正的邮箱变更在 `handleVerification` 中执行
- **Handle Verification**：
  - 调用 `requireRecentVerification`
  - 确保在执行高敏感操作（修改邮箱）前用户已近期验证过

---

## 3. 完整调用链路（按真实顺序）

### 3.1 普通登录流程（无 2FA）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. 用户访问 /login                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  2. root.tsx loader()  [全局执行]                                             │
│      → getUserId(request)                                                      │
│          → 从 Cookie 读取 en_session                                           │
│          → 无 sessionId → 返回 null                                            │
│      → userId 为 null，不获取用户信息                                          │
│      → 返回数据（user: null）                                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  3. login.tsx loader()                                                         │
│      → requireAnonymous(request)                                               │
│          → getUserId(request) → 返回 null                                      │
│          → 用户未登录，继续加载页面                                             │
│      → 返回空数据                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ 用户输入用户名密码，提交表单
┌─────────────────────────────────────────────────────────────────────────────┐
│  4. login.tsx action()                                                         │
│      → requireAnonymous(request)  [确保未登录]                                 │
│      → checkHoneypot(formData)    [反机器人检查]                               │
│      → login({ username, password })                                           │
│          → verifyUserPassword()  [验证密码哈希]                                 │
│          → prisma.session.create()  [创建数据库会话]                           │
│              → sessionId = 123                                                 │
│              → expirationDate = 30天后                                         │
│      → handleNewSession({ session, remember })                                 │
│          → 检查用户是否启用 2FA → 未启用                                       │
│          → authSession.set(sessionKey, sessionId)                              │
│          → authSessionStorage.commitSession()                                   │
│              → 设置 en_session Cookie                                           │
│      → redirect(redirectTo)                                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  5. 用户已登录，后续请求                                                       │
│      → 每次请求都会执行 root.tsx loader()                                      │
│          → getUserId(request)                                                  │
│              → 从 Cookie 读取 en_session                                       │
│              → sessionId = 123                                                 │
│              → prisma.session.findUnique({                                     │
│                    where: { id: 123, expirationDate: { gt: now } }            │
│                  })                                                             │
│              → 会话有效 → 返回 userId                                           │
│          → 获取用户详情（包含角色和权限）                                       │
│      → 子路由根据需要调用 requireUserId/requireUserWithRole 等                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 两步验证登录流程（有 2FA）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  阶段 A：密码验证后（临时状态）                                                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ 密码验证成功
┌─────────────────────────────────────────────────────────────────────────────┐
│  login.tsx action()                                                           │
│      → login() 成功，创建数据库会话 sessionId=123                             │
│      → handleNewSession()                                                      │
│          → 检查用户是否启用 2FA → 已启用                                       │
│          → verifySessionStorage.getSession()  [创建新的空会话]                 │
│          → verifySession.set('unverified-session-id', 123)                   │
│          → verifySession.set('remember', true)                                 │
│          → verifySessionStorage.commitSession()                                 │
│              → 设置 en_verification Cookie（10分钟过期）                        │
│          → 重定向到 /verify?type=2fa&target=userId                            │
│                                                                                 │
│  【关键状态】                                                                    │
│  - 数据库会话：已创建                                                           │
│  - en_session Cookie：未设置 ❌                                                │
│  - en_verification Cookie：已设置 ✅                                           │
│  - 用户登录状态：未登录 ❌                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ 用户被重定向到 /verify
┌─────────────────────────────────────────────────────────────────────────────┐
│  verify.tsx loader()                                                          │
│      → 显示 2FA 代码输入页面                                                   │
│      → 注意：没有调用身份验证函数！                                             │
│      → 因为用户此时还未登录，但需要输入 2FA 代码                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ 用户输入 2FA 代码，提交
┌─────────────────────────────────────────────────────────────────────────────┐
│  阶段 B：2FA 验证通过后（正式登录）                                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  verify.tsx action()                                                          │
│      → validateRequest()                                                       │
│          → isCodeValid()  [验证 TOTP 代码]                                    │
│          → switch (type)                                                       │
│              case '2fa':                                                        │
│                  → handleLoginTwoFactorVerification()  [即 handleVerification] │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  handleVerification() (login.server.ts:83-137)                               │
│      → authSessionStorage.getSession(request.headers.get('cookie'))           │
│          → 从请求读取 en_session（此时为空）                                   │
│      → verifySessionStorage.getSession(request.headers.get('cookie'))          │
│          → 从请求读取 en_verification                                           │
│      → remember = verifySession.get('remember')                                │
│      → authSession.set('verified-time', Date.now())  [记录验证时间]            │
│      → unverifiedSessionId = verifySession.get('unverified-session-id')        │
│          → 123                                                                  │
│      → if (unverifiedSessionId)  [登录场景]                                    │
│          → prisma.session.findUnique({ id: 123 })  [验证数据库会话]            │
│          → authSession.set('sessionId', 123)  [关键：转移会话 ID]              │
│          → authSessionStorage.commitSession(authSession, {                     │
│                expires: remember ? session.expirationDate : undefined          │
│            })                                                                   │
│              → 设置 en_session Cookie ✅                                        │
│      → verifySessionStorage.destroySession(verifySession)                      │
│          → 清除 en_verification Cookie ❌                                       │
│      → redirect(redirectTo)                                                     │
│                                                                                 │
│  【关键状态】                                                                    │
│  - en_session Cookie：已设置 ✅（包含 sessionId=123）                          │
│  - en_verification Cookie：已清除 ❌                                           │
│  - 用户登录状态：已登录 ✅                                                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 已登录用户的高敏感操作（如修改邮箱）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. 用户访问 /settings/profile/change-email                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  2. root.tsx loader()                                                          │
│      → getUserId(request)                                                      │
│          → 从 Cookie 读取 en_session                                           │
│          → sessionId = 123                                                     │
│          → 验证数据库会话 → 返回 userId                                         │
│      → 获取用户详情（包含角色和权限）                                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  3. change-email.tsx loader()                                                 │
│      → requireRecentVerification(request)                                      │
│          → requireUserId(request) → 已登录，返回 userId                        │
│          → shouldRequestTwoFA(request)                                         │
│              → 检查是否有 unverified-session-id → 没有                         │
│              → getUserId(request) → 已登录                                      │
│              → 检查用户是否启用 2FA                                             │
│                  → 未启用 → 返回 false（不需要重新验证）                        │
│                  → 已启用 → 检查 verified-time                                  │
│                      → 2小时内 → 返回 false（不需要重新验证）                   │
│                      → 超过2小时 → 返回 true（需要重新验证）                    │
│          → 如果需要重新验证 → 重定向到 /verify?type=2fa                        │
│      → requireUserId(request)  [再次检查，冗余但安全]                           │
│      → 加载修改邮箱页面                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ 超过 2 小时需要重新验证
┌─────────────────────────────────────────────────────────────────────────────┐
│  4. 用户被重定向到 /verify?type=2fa                                            │
│      → 输入 2FA 代码，提交                                                      │
│      → handleVerification() 被调用                                             │
│          → 注意：此时没有 unverifiedSessionId                                   │
│          → authSession.set('verified-time', Date.now())  [只更新验证时间]      │
│          → authSessionStorage.commitSession(authSession)  [不设置 sessionId]   │
│          → 重定向回 /settings/profile/change-email                             │
│      → 用户继续操作                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 关键概念澄清

### 4.1 根路由 loader 不是"拦截器"

**代码位置**：`app/root.tsx:71-133`

```typescript
export async function loader({ request }: Route.LoaderArgs) {
	const timings = makeTimings('root loader')
	const userId = await time(() => getUserId(request), {
		timings,
		type: 'getUserId',
		desc: 'getUserId in root',
	})

	const user = userId
		? await time(
				() =>
					prisma.user.findUnique({
						select: {
							id: true,
							name: true,
							username: true,
							image: { select: { objectKey: true } },
							roles: {
								select: {
									name: true,
									permissions: {
										select: { entity: true, action: true, access: true },
									},
								},
							},
						},
						where: { id: userId },
					}),
				// ...
			)
		: null
	// ...
}
```

**澄清**：
- 根路由 loader **只调用 `getUserId`**，不调用 `requireUserId`
- `getUserId` 的行为：
  - 没有 `sessionId` → 返回 `null`（**不会阻止访问**）
  - 有 `sessionId` 但会话无效 → 抛出重定向到首页（清除 cookie）
  - 有有效会话 → 返回 `userId`
- 根路由 loader 的**主要目的**：
  - 获取用户信息供前端使用（显示头像、用户名、导航菜单等）
  - 不是"拦截器"，不会阻止未登录用户访问页面
- **真正的拦截**：
  - 在子路由的 loader/action 中通过调用 `requireUserId` 等函数实现
  - 例如：`/admin/cache` 的 loader 调用 `requireUserWithRole(request, 'admin')`

### 4.2 两个独立的 Cookie 存储

| Cookie 名称 | 用途 | 存储内容 | 生命周期 |
|-------------|------|----------|----------|
| `en_session` | 正式登录会话 | `sessionId`, `verified-time`, `expires` | 可配置（默认 30 天或会话） |
| `en_verification` | 临时验证会话 | `unverified-session-id`, `remember`, `new-email-address` 等 | **10 分钟** |

**代码位置**：
- `en_session`：`app/utils/session.server.ts:3-12`
- `en_verification`：`app/utils/verification.server.ts:1-13`

```typescript
// en_verification 的 maxAge 是 10 分钟
export const verifySessionStorage = createCookieSessionStorage({
	cookie: {
		name: 'en_verification',
		// ...
		maxAge: 60 * 10, // 10 minutes
		// ...
	},
})
```

### 4.3 handleVerification 的两种使用场景

**场景 1：登录时的 2FA 验证**

| 项目 | 状态 |
|------|------|
| 触发 | 用户登录，密码验证成功，启用了 2FA |
| `unverifiedSessionId` | 有（从 `en_verification` Cookie 读取） |
| 操作 | 设置 `sessionKey` → 用户正式登录 |
| Cookie 变化 | `en_session` 被设置，`en_verification` 被清除 |

**场景 2：已登录用户的敏感操作**

| 项目 | 状态 |
|------|------|
| 触发 | 用户执行高敏感操作（如修改邮箱），超过 2 小时未验证 |
| `unverifiedSessionId` | 没有 |
| 操作 | 只更新 `verified-time` |
| Cookie 变化 | `en_session` 被更新（`verified-time`），`en_verification` 被清除 |

**代码位置**：`app/routes/_auth/login.server.ts:103-129`

```typescript
const unverifiedSessionId = verifySession.get(unverifiedSessionIdKey)
if (unverifiedSessionId) {
	// 场景 1：登录时的 2FA 验证
	const session = await prisma.session.findUnique({
		select: { expirationDate: true },
		where: { id: unverifiedSessionId },
	})
	// ...
	authSession.set(sessionKey, unverifiedSessionId)  // 设置 sessionKey

	headers.append(
		'set-cookie',
		await authSessionStorage.commitSession(authSession, {
			expires: remember ? session.expirationDate : undefined,
		}),
	)
} else {
	// 场景 2：已登录用户的敏感操作
	headers.append(
		'set-cookie',
		await authSessionStorage.commitSession(authSession),  // 不设置 sessionKey
	)
}
```

---

## 5. 代码索引（核对版）

### 5.1 核心函数位置

| 函数名 | 文件路径 | 行号 | 功能说明 |
|--------|----------|------|----------|
| `getUserId` | `app/utils/auth.server.ts` | 29-47 | 基础会话验证，返回 userId 或 null |
| `requireUserId` | `app/utils/auth.server.ts` | 49-67 | 要求已登录，未登录则重定向 |
| `requireAnonymous` | `app/utils/auth.server.ts` | 69-74 | 要求未登录，已登录则重定向 |
| `requireUserWithRole` | `app/utils/permissions.server.ts` | 43-60 | 要求特定角色，无角色则 403 |
| `requireUserWithPermission` | `app/utils/permissions.server.ts` | 6-41 | 要求特定权限，无权限则 403 |
| `requireRecentVerification` | `app/routes/_auth/verify.server.ts` | 58-74 | 要求近期验证过，否则重定向到 2FA |
| `shouldRequestTwoFA` | `app/routes/_auth/login.server.ts` | 139-158 | 检查是否需要重新验证 2FA |
| `handleNewSession` | `app/routes/_auth/login.server.ts` | 17-81 | 处理新会话（密码验证后） |
| `handleVerification` | `app/routes/_auth/login.server.ts` | 83-137 | 处理验证完成（2FA 验证后） |

### 5.2 实际路由示例

| 路由 | 校验函数 | 挂载点 | 说明 |
|------|----------|--------|------|
| `/login` | `requireAnonymous` | loader, action | 只允许未登录用户访问 |
| `/admin/cache` | `requireUserWithRole('admin')` | loader, action | 只允许 admin 角色 |
| `/users/:username/notes/:noteId` (loader) | **无** | - | 公开访问，任何人可查看 |
| `/users/:username/notes/:noteId` (action) | `requireUserId` + `requireUserWithPermission` | action | 删除操作需要权限 |
| `/settings/profile/change-email` (loader) | `requireRecentVerification` + `requireUserId` | loader | 高敏感操作，需要近期验证 |
| `/settings/profile/change-email` (action) | `requireUserId` | action | 只要求登录 |

### 5.3 关键常量

| 常量名 | 值 | 说明 | 位置 |
|--------|-----|------|------|
| `SESSION_EXPIRATION_TIME` | `1000 * 60 * 60 * 24 * 30` | 30 天 | `app/utils/auth.server.ts:14` |
| `twoHours` | `1000 * 60 * 2` | 2 小时（复验阈值） | `app/routes/_auth/login.server.ts:156` |
| `maxAge` (verifySession) | `60 * 10` | 10 分钟 | `app/utils/verification.server.ts:9` |
| `sessionKey` | `'sessionId'` | 会话 ID 键名 | `app/utils/auth.server.ts:18` |
| `verifiedTimeKey` | `'verified-time'` | 验证时间键名 | `app/routes/_auth/login.server.ts:13` |
| `unverifiedSessionIdKey` | `'unverified-session-id'` | 未验证会话 ID 键名 | `app/routes/_auth/login.server.ts:14` |
| `rememberKey` | `'remember'` | 记住我键名 | `app/routes/_auth/login.server.ts:15` |

---

## 6. 核对结论总结

### 6.1 已修正的关键口径

| 口径 | 之前可能的误解 | 核对后的准确结论 |
|------|----------------|------------------|
| 近期复验时长 | 2 小时内需要验证 | **超过 2 小时**才需要重新验证 |
| 2FA 时的 Cookie | 可能认为设置了 `en_session` | **不设置** `en_session`，只设置 `en_verification` |
| 根路由 loader | 可能认为是拦截器 | **不是拦截器**，只获取用户信息供前端使用 |
| Notes 路由 | 可能认为需要登录才能查看 | **loader 不校验**，公开访问；**action 才校验** |
| `verifySessionStorage.getSession()` | 可能认为都从 cookie 读取 | `handleNewSession` 中**不传参数**，创建新会话 |

### 6.2 核心流程确认

**登录流程（无 2FA）**：
1. 密码验证成功 → 创建数据库会话
2. 立即设置 `en_session` Cookie → 用户正式登录

**登录流程（有 2FA）**：
1. 密码验证成功 → 创建数据库会话
2. **不设置** `en_session`，设置 `en_verification`（包含 `unverified-session-id`）
3. 用户输入 2FA 代码 → 验证通过
4. 从 `en_verification` 读取 `unverified-session-id`
5. 设置 `en_session`（包含 `sessionId`）→ 用户正式登录
6. 清除 `en_verification`

**高敏感操作（已登录用户）**：
1. 检查 `verified-time`
2. 超过 2 小时 → 重定向到 2FA 验证
3. 验证通过 → 只更新 `verified-time`，不重新设置 `sessionId`

### 6.3 校验层级确认

```
getUserId (基础会话验证)
    │
    └──→ requireUserId (要求登录)
            │
            ├──→ requireUserWithRole (角色校验)
            ├──→ requireUserWithPermission (权限校验)
            └──→ requireRecentVerification (近期验证校验)
```

所有校验函数都遵循**显式调用**原则，没有中间件自动拦截。真正的"拦截"发生在子路由的 loader/action 中，通过显式调用相应的校验函数实现。
