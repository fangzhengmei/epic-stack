# Epic Stack 会话与权限链路深度分析

## 1. 登录与两步验证完整流程

### 1.1 普通登录流程（无 2FA）

#### 1.1.1 登录表单提交

当用户在登录页面输入用户名和密码并提交时，流程如下：

**调用顺序：**
1. `app/routes/_auth/login.tsx` 的 `action` 函数被调用
2. 调用 `requireAnonymous(request)` 确保用户未登录
3. 解析表单数据，验证 honeypot
4. 调用 `login({ username, password })` 验证用户凭证

#### 1.1.2 登录验证逻辑

```typescript
// app/utils/auth.server.ts:76-93
export async function login({
	username,
	password,
}: {
	username: User['username']
	password: string
}) {
	const user = await verifyUserPassword({ username }, password)
	if (!user) return null
	const session = await prisma.session.create({
		select: { id: true, expirationDate: true, userId: true },
		data: {
			expirationDate: getSessionExpirationDate(),
			userId: user.id,
		},
	})
	return session
}
```

**关键步骤：**
1. **验证密码**：调用 `verifyUserPassword` 检查用户名和密码是否匹配
   - 从数据库获取用户的密码哈希
   - 使用 bcrypt 比对输入密码和存储的哈希
2. **创建数据库会话**：如果验证成功，在数据库中创建会话记录
   - `expirationDate`: 默认 30 天后过期
   - `userId`: 关联到用户 ID
3. **返回会话信息**：返回会话的 `id`、`expirationDate` 和 `userId`

#### 1.1.3 处理新会话

登录验证成功后，调用 `handleNewSession` 处理会话建立：

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
	const verification = await prisma.verification.findUnique({
		select: { id: true },
		where: {
			target_type: { target: session.userId, type: twoFAVerificationType },
		},
	})
	const userHasTwoFactor = Boolean(verification)

	if (userHasTwoFactor) {
		// 处理双因素认证流程（见下文）
	} else {
		// 直接建立正式会话
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

**无 2FA 时的会话建立：**
1. **检查 2FA 状态**：查询数据库，检查用户是否启用了双因素认证
2. **获取认证会话**：从请求的 Cookie 中获取或创建 `authSession`
3. **存储会话 ID**：将会话 ID 存储到认证会话中
   - `sessionKey` 定义为 `'sessionId'`
4. **提交会话到 Cookie**：
   - 调用 `authSessionStorage.commitSession`
   - 如果用户勾选了 "记住我"，设置 Cookie 过期时间为会话过期时间
   - 否则，Cookie 为会话 Cookie（浏览器关闭后失效）
5. **重定向**：重定向到用户请求的页面或首页

---

### 1.2 两步验证流程（有 2FA）

#### 1.2.1 第一步：密码验证后进入临时状态

当用户启用了双因素认证时，`handleNewSession` 会走不同的分支：

```typescript
// app/routes/_auth/login.server.ts:39-60
if (userHasTwoFactor) {
	const verifySession = await verifySessionStorage.getSession()
	verifySession.set(unverifiedSessionIdKey, session.id)
	verifySession.set(rememberKey, remember)
	const redirectUrl = getRedirectToUrl({
		request,
		type: twoFAVerificationType,
		target: session.userId,
		redirectTo,
	})
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
}
```

**关键变量定义：**
```typescript
// app/routes/_auth/login.server.ts:13-15
const verifiedTimeKey = 'verified-time'
const unverifiedSessionIdKey = 'unverified-session-id'
const rememberKey = 'remember'
```

**临时会话建立流程：**
1. **创建验证会话**：获取或创建 `verifySession`（存储在名为 `en_verification` 的 Cookie 中）
   
   ```typescript
   // app/utils/verification.server.ts:1-13
   export const verifySessionStorage = createCookieSessionStorage({
   	cookie: {
   		name: 'en_verification',
   		sameSite: 'lax',
   		path: '/',
   		httpOnly: true,
   		maxAge: 60 * 10, // 10 分钟过期
   		secrets: process.env.SESSION_SECRET.split(','),
   		secure: process.env.NODE_ENV === 'production',
   	},
   })
   ```

2. **存储临时数据**：
   - `unverifiedSessionIdKey`: 存储数据库中已创建的会话 ID
   - `rememberKey`: 存储用户是否勾选了 "记住我"

3. **构建重定向 URL**：
   - 路径：`/verify`
   - 查询参数：
     - `type`: `'2fa'`（验证类型）
     - `target`: 用户 ID
     - `redirectTo`: 登录成功后要跳转的页面

4. **提交验证会话 Cookie**：
   - 设置 `en_verification` Cookie
   - 注意：**此时并没有设置 `en_session` Cookie**，用户处于未登录状态

5. **重定向到验证页面**：用户被重定向到 `/verify` 页面输入 2FA 代码

#### 1.2.2 第二步：2FA 代码验证

用户在验证页面输入 2FA 代码并提交：

**调用顺序：**
1. `app/routes/_auth/verify.tsx` 的 `action` 函数被调用
2. 调用 `validateRequest(request, formData)` 处理验证

```typescript
// app/routes/_auth/verify.server.ts:140-199
export async function validateRequest(
	request: Request,
	body: URLSearchParams | FormData,
) {
	const submission = await parseWithZod(body, {
		schema: VerifySchema.superRefine(async (data, ctx) => {
			const codeIsValid = await isCodeValid({
				code: data[codeQueryParam],
				type: data[typeQueryParam],
				target: data[targetQueryParam],
			})
			if (!codeIsValid) {
				ctx.addIssue({
					path: ['code'],
					code: z.ZodIssueCode.custom,
					message: `Invalid code`,
				})
				return
			}
		}),
		async: true,
	})

	if (submission.status !== 'success') {
		return data(
			{ result: submission.reply() },
			{ status: submission.status === 'error' ? 400 : 200 },
		)
	}

	// ... 省略其他验证类型处理

	switch (submissionValue[typeQueryParam]) {
		// ... 省略其他 case
		case '2fa': {
			return handleLoginTwoFactorVerification({ request, body, submission })
		}
	}
}
```

**代码验证逻辑：**
1. **解析表单数据**：使用 Zod 验证输入
2. **验证 2FA 代码**：调用 `isCodeValid` 检查代码是否有效
   - 从数据库查找对应的验证记录
   - 使用 `verifyTOTP` 验证时间-based 一次性密码
3. **根据验证类型分发处理**：对于 `'2fa'` 类型，调用 `handleLoginTwoFactorVerification`

#### 1.2.3 第三步：临时会话转正（关键步骤）

这是整个流程最关键的部分，临时会话如何转换为正式登录态：

```typescript
// app/routes/_auth/login.server.ts:83-137
export async function handleVerification({
	request,
	submission,
}: VerifyFunctionArgs) {
	invariant(
		submission.status === 'success',
		'Submission should be successful by now',
	)
	const authSession = await authSessionStorage.getSession(
		request.headers.get('cookie'),
	)
	const verifySession = await verifySessionStorage.getSession(
		request.headers.get('cookie'),
	)

	const remember = verifySession.get(rememberKey)
	const { redirectTo } = submission.value
	const headers = new Headers()
	authSession.set(verifiedTimeKey, Date.now())

	const unverifiedSessionId = verifySession.get(unverifiedSessionIdKey)
	if (unverifiedSessionId) {
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

	headers.append(
		'set-cookie',
		await verifySessionStorage.destroySession(verifySession),
	)

	return redirect(safeRedirect(redirectTo), { headers })
}
```

**临时会话转正流程详解：**

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | 获取两个会话 | `authSession`（正式会话）和 `verifySession`（验证会话） |
| 2 | 提取临时数据 | 从 `verifySession` 中获取 `remember` 和 `unverifiedSessionId` |
| 3 | 记录验证时间 | 在 `authSession` 中设置 `verified-time` 为当前时间 |
| 4 | **关键：转移会话 ID** | 将 `unverifiedSessionId` 从验证会话移动到正式会话 |
| 5 | 验证数据库会话 | 确保该会话 ID 在数据库中仍然存在且有效 |
| 6 | **提交正式 Cookie** | 调用 `authSessionStorage.commitSession`，设置 `en_session` Cookie |
| 7 | **销毁验证 Cookie** | 调用 `verifySessionStorage.destroySession`，清除 `en_verification` Cookie |
| 8 | 重定向 | 重定向到用户请求的页面 |

**关键数据流向：**

```
密码验证成功后：
  ┌─────────────────┐     ┌─────────────────────┐
  │  数据库会话表   │────▶│  verifySession      │
  │  (已创建)       │     │  (en_verification)  │
  │  sessionId=123  │     │  unverified-session │
  └─────────────────┘     │  -id=123            │
                          │  remember=true       │
                          └─────────────────────┘
                          ↑ 此时 en_session 为空

2FA 验证通过后：
  ┌─────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
  │  数据库会话表   │────▶│  authSession        │────▶│  en_session Cookie  │
  │  sessionId=123  │     │  sessionId=123      │     │  已设置              │
  └─────────────────┘     │  verified-time=...  │     └─────────────────────┘
                          └─────────────────────┘
                                                  
                          ┌─────────────────────┐
                          │  verifySession      │────▶│  已销毁               │
                          │  (en_verification)  │     │  Cookie 被清除        │
                          └─────────────────────┘
```

---

## 2. 后台拦截机制分析

### 2.1 全局身份验证（根路由 Loader）

在 `app/root.tsx` 的 `loader` 中，会进行全局身份验证：

```typescript
// app/root.tsx:71-133
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
				{ timings, type: 'find user', desc: 'find user in root' },
			)
		: null
	if (userId && !user) {
		console.info('something weird happened')
		// something weird happened... The user is authenticated but we can't find
		// them in the database. Maybe they were deleted? Let's log them out.
		await logout({ request, redirectTo: '/' })
	}
	// ... 其他逻辑
}
```

**全局验证流程：**
1. **调用 `getUserId`**：这是每次请求都会执行的第一步验证
2. **获取用户详情**：如果有用户 ID，从数据库获取用户信息（包括角色和权限）
3. **处理异常情况**：如果有用户 ID 但找不到用户记录，注销用户

### 2.2 `getUserId` 函数详解

`getUserId` 是身份验证的核心函数：

```typescript
// app/utils/auth.server.ts:29-47
export async function getUserId(request: Request) {
	const authSession = await authSessionStorage.getSession(
		request.headers.get('cookie'),
	)
	const sessionId = authSession.get(sessionKey)
	if (!sessionId) return null
	const session = await prisma.session.findUnique({
		select: { userId: true },
		where: { id: sessionId, expirationDate: { gt: new Date() } },
	})
	if (!session?.userId) {
		throw redirect('/', {
			headers: {
				'set-cookie': await authSessionStorage.destroySession(authSession),
			},
		})
	}
	return session.userId
}
```

**验证步骤：**
1. **从 Cookie 获取会话**：解析 `en_session` Cookie，获取 `authSession`
2. **提取会话 ID**：从 `authSession` 中获取 `sessionId`
3. **无会话 ID**：直接返回 `null`，表示用户未登录
4. **数据库验证**：
   - 查询条件：`id = sessionId` 且 `expirationDate > now`
   - 只选择 `userId` 字段
5. **会话无效处理**：
   - 如果数据库中找不到有效的会话记录
   - 抛出重定向到首页
   - 同时销毁客户端的会话 Cookie
6. **返回用户 ID**：验证通过，返回用户 ID

### 2.3 路由级别的拦截机制

Epic Stack 采用**显式调用**的拦截机制，而非中间件模式。不同的路由根据安全需求调用不同的验证函数。

#### 2.3.1 验证函数层级

| 层级 | 函数 | 用途 | 调用链 |
|------|------|------|--------|
| 1 | `getUserId` | 基础身份验证 | 直接查询会话 |
| 2 | `requireUserId` | 要求已登录 | `getUserId` → 未登录则重定向 |
| 2 | `requireAnonymous` | 要求未登录 | `getUserId` → 已登录则重定向 |
| 3 | `requireUserWithRole` | 要求特定角色 | `requireUserId` → 检查角色 |
| 3 | `requireUserWithPermission` | 要求特定权限 | `requireUserId` → 检查权限 |
| 3 | `requireRecentVerification` | 要求近期验证过 | `requireUserId` → 检查 2FA 验证时间 |

#### 2.3.2 各验证函数实现

**`requireUserId`**：
```typescript
// app/utils/auth.server.ts:49-67
export async function requireUserId(
	request: Request,
	{ redirectTo }: { redirectTo?: string | null } = {},
) {
	const userId = await getUserId(request)
	if (!userId) {
		const requestUrl = new URL(request.url)
		redirectTo =
			redirectTo === null
				? null
				: (redirectTo ?? `${requestUrl.pathname}${requestUrl.search}`)
		const loginParams = redirectTo ? new URLSearchParams({ redirectTo }) : null
		const loginRedirect = ['/login', loginParams?.toString()]
			.filter(Boolean)
			.join('?')
		throw redirect(loginRedirect)
	}
	return userId
}
```

**`requireAnonymous`**：
```typescript
// app/utils/auth.server.ts:69-74
export async function requireAnonymous(request: Request) {
	const userId = await getUserId(request)
	if (userId) {
		throw redirect('/')
	}
}
```

**`requireUserWithRole`**：
```typescript
// app/utils/permissions.server.ts:43-60
export async function requireUserWithRole(request: Request, name: string) {
	const userId = await requireUserId(request)
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

**`requireUserWithPermission`**：
```typescript
// app/utils/permissions.server.ts:6-41
export async function requireUserWithPermission(
	request: Request,
	permission: PermissionString,
) {
	const userId = await requireUserId(request)
	const permissionData = parsePermissionString(permission)
	const user = await prisma.user.findFirst({
		select: { id: true },
		where: {
			id: userId,
			roles: {
				some: {
					permissions: {
						some: {
							...permissionData,
							access: permissionData.access
								? { in: permissionData.access }
								: undefined,
						},
					},
				},
			},
		},
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

**`requireRecentVerification`**：
```typescript
// app/routes/_auth/verify.server.ts:58-74
export async function requireRecentVerification(request: Request) {
	const userId = await requireUserId(request)
	const shouldReverify = await shouldRequestTwoFA(request)
	if (shouldReverify) {
		const reqUrl = new URL(request.url)
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

**`shouldRequestTwoFA`**：
```typescript
// app/routes/_auth/login.server.ts:139-158
export async function shouldRequestTwoFA(request: Request) {
	const authSession = await authSessionStorage.getSession(
		request.headers.get('cookie'),
	)
	const verifySession = await verifySessionStorage.getSession(
		request.headers.get('cookie'),
	)
	if (verifySession.has(unverifiedSessionIdKey)) return true
	const userId = await getUserId(request)
	if (!userId) return false
	// if it's over two hours since they last verified, we should request 2FA again
	const userHasTwoFA = await prisma.verification.findUnique({
		select: { id: true },
		where: { target_type: { target: userId, type: twoFAVerificationType } },
	})
	if (!userHasTwoFA) return false
	const verifiedTime = authSession.get(verifiedTimeKey) ?? new Date(0)
	const twoHours = 1000 * 60 * 2
	return Date.now() - verifiedTime > twoHours
}
```

---

## 3. 实际受保护路由示例分析

### 3.1 Admin 路由（角色校验）

**文件路径**：`app/routes/admin/cache/index.tsx`

这是一个典型的**角色校验**示例，只有 admin 角色的用户才能访问。

#### 3.1.1 Loader 中的校验

```typescript
// app/routes/admin/cache/index.tsx:34-57
export async function loader({ request }: Route.LoaderArgs) {
	await requireUserWithRole(request, 'admin')
	const searchParams = new URL(request.url).searchParams
	const query = searchParams.get('query')
	if (query === '') {
		searchParams.delete('query')
		return redirect(`/admin/cache?${searchParams.toString()}`)
	}
	const limit = Number(searchParams.get('limit') ?? 100)

	const currentInstanceInfo = await getInstanceInfo()
	const instance =
		searchParams.get('instance') ?? currentInstanceInfo.currentInstance
	const instances = await getAllInstances()
	await ensureInstance(instance)

	let cacheKeys: { sqlite: Array<string>; lru: Array<string> }
	if (typeof query === 'string') {
		cacheKeys = await searchCacheKeys(query, limit)
	} else {
		cacheKeys = await getAllCacheKeys(limit)
	}
	return { cacheKeys, instance, instances, currentInstanceInfo }
}
```

#### 3.1.2 Action 中的校验

```typescript
// app/routes/admin/cache/index.tsx:59-86
export async function action({ request }: Route.ActionArgs) {
	await requireUserWithRole(request, 'admin')
	const formData = await request.formData()
	const key = formData.get('cacheKey')
	const { currentInstance } = await getInstanceInfo()
	const instance = formData.get('instance') ?? currentInstance
	const type = formData.get('type')

	invariantResponse(typeof key === 'string', 'cacheKey must be a string')
	invariantResponse(typeof type === 'string', 'type must be a string')
	invariantResponse(typeof instance === 'string', 'instance must be a string')
	await ensureInstance(instance)

	switch (type) {
		case 'sqlite': {
			await cache.delete(key)
			break
		}
		case 'lru': {
			lruCache.delete(key)
			break
		}
		default: {
			throw new Error(`Unknown cache type: ${type}`)
		}
	}
	return { success: true }
}
```

**Admin 路由的校验特点：**
- **校验位置**：`loader` 和 `action` 中都进行校验
- **校验级别**：角色级别的校验（`requireUserWithRole`）
- **校验内容**：用户必须具有 `'admin'` 角色
- **调用链**：
  ```
  requireUserWithRole(request, 'admin')
    → requireUserId(request)
      → getUserId(request)
  ```

#### 3.1.3 错误处理

```typescript
// app/routes/admin/cache/index.tsx:234-244
export function ErrorBoundary() {
	return (
		<GeneralErrorBoundary
			statusHandlers={{
				403: ({ error }) => (
					<p>You are not allowed to do that: {error?.data.message}</p>
				),
			}}
		/>
	)
}
```

当用户没有 admin 角色时，`requireUserWithRole` 会抛出 403 错误，ErrorBoundary 会捕获并显示友好的错误信息。

---

### 3.2 Notes 路由（权限校验）

**文件路径**：`app/routes/users/$username/notes/$noteId.tsx`

这是一个典型的**权限校验**示例，不同的用户对笔记有不同的操作权限。

#### 3.2.1 Loader 中的校验

```typescript
// app/routes/users/$username/notes/$noteId.tsx:24-49
export async function loader({ params }: Route.LoaderArgs) {
	const note = await prisma.note.findUnique({
		where: { id: params.noteId },
		select: {
			id: true,
			title: true,
			content: true,
			ownerId: true,
			updatedAt: true,
			images: {
				select: {
					id: true,
					altText: true,
					objectKey: true,
				},
			},
		},
	})

	invariantResponse(note, 'Not found', { status: 404 })

	const date = new Date(note.updatedAt)
	const timeAgo = formatDistanceToNow(date)

	return { note, timeAgo }
}
```

**注意**：`loader` 中**没有**调用任何身份验证函数！这意味着：
- 任何人都可以查看笔记（公开访问）
- 身份验证只在需要修改数据的 `action` 中进行

#### 3.2.2 Action 中的校验

```typescript
// app/routes/users/$username/notes/$noteId.tsx:56-90
export async function action({ request }: Route.ActionArgs) {
	const userId = await requireUserId(request)
	const formData = await request.formData()
	const submission = parseWithZod(formData, {
		schema: DeleteFormSchema,
	})
	if (submission.status !== 'success') {
		return data(
			{ result: submission.reply() },
			{ status: submission.status === 'error' ? 400 : 200 },
		)
	}

	const { noteId } = submission.value

	const note = await prisma.note.findFirst({
		select: { id: true, ownerId: true, owner: { select: { username: true } } },
		where: { id: noteId },
	})
	invariantResponse(note, 'Not found', { status: 404 })

	const isOwner = note.ownerId === userId
	await requireUserWithPermission(
		request,
		isOwner ? `delete:note:own` : `delete:note:any`,
	)

	await prisma.note.delete({ where: { id: note.id } })

	return redirectWithToast(`/users/${note.owner.username}/notes`, {
		type: 'success',
		title: 'Success',
		description: 'Your note has been deleted.',
	})
}
```

**Notes 路由的校验特点：**
- **校验位置**：只在 `action` 中进行校验
- **校验级别**：
  1. 首先调用 `requireUserId` 确保用户已登录
  2. 然后根据业务逻辑调用 `requireUserWithPermission`
- **动态权限**：
  - 如果是笔记所有者：需要 `delete:note:own` 权限
  - 如果不是所有者：需要 `delete:note:any` 权限（通常只有管理员有）

#### 3.2.3 前端权限检查

```typescript
// app/routes/users/$username/notes/$noteId.tsx:92-170
export default function NoteRoute({
	loaderData,
	actionData,
}: Route.ComponentProps) {
	const user = useOptionalUser()
	const isOwner = user?.id === loaderData.note.ownerId
	const canDelete = userHasPermission(
		user,
		isOwner ? `delete:note:own` : `delete:note:any`,
	)
	const displayBar = canDelete || isOwner

	// ... 组件渲染逻辑

	return (
		// ...
		<div className={floatingToolbarClassName}>
			{/* ... */}
			<div className="grid flex-1 grid-cols-2 justify-end gap-2 min-[525px]:flex md:gap-4">
				{canDelete ? (
					<DeleteNote id={loaderData.note.id} actionData={actionData} />
				) : null}
				<Button
					asChild
					className="min-[525px]:max-md:aspect-square min-[525px]:max-md:px-0"
				>
					<Link to="edit">
						<Icon name="pencil-1" className="scale-125 max-md:scale-150">
							<span className="max-md:hidden">Edit</span>
						</Icon>
					</Link>
				</Button>
			</div>
		</div>
		// ...
	)
}
```

**前端权限检查特点：**
- 使用 `useOptionalUser()` 从根 loader 获取用户信息
- 使用 `userHasPermission()` 检查权限（纯前端逻辑，不调用后端）
- 根据权限决定是否显示删除按钮
- **重要**：前端检查仅用于 UI 展示，后端 `action` 中仍有完整的权限校验

---

### 3.3 Change Email 路由（近期验证校验）

**文件路径**：`app/routes/settings/profile/change-email.server.tsx`

这是一个**高敏感操作**的示例，需要用户近期验证过身份。

#### 3.3.1 校验实现

```typescript
// app/routes/settings/profile/change-email.server.tsx:14-69
export async function handleVerification({
	request,
	submission,
}: VerifyFunctionArgs) {
	await requireRecentVerification(request)
	invariant(
		submission.status === 'success',
		'Submission should be successful by now',
	)

	const verifySession = await verifySessionStorage.getSession(
		request.headers.get('cookie'),
	)
	const newEmail = verifySession.get(newEmailAddressSessionKey)
	if (!newEmail) {
		return data(
			{
				result: submission.reply({
					formErrors: [
						'You must submit the code on the same device that requested the email change.',
					],
				}),
			},
			{ status: 400 },
		)
	}
	// ... 执行邮箱更改逻辑
}
```

**Change Email 路由的校验特点：**
- **校验位置**：在处理验证的函数中
- **校验级别**：`requireRecentVerification`
- **校验逻辑**：
  1. 首先调用 `requireUserId` 确保登录
  2. 检查用户是否启用了 2FA
  3. 如果启用了 2FA，检查最近 2 小时内是否验证过
  4. 如果需要重新验证，重定向到 2FA 验证页面

---

## 4. 完整调用链路总结

### 4.1 登录流程（无 2FA）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           用户访问 /login                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  login.tsx loader()                                                           │
│    → requireAnonymous(request)                                                │
│      → getUserId(request)                                                     │
│        → 从 Cookie 获取 en_session                                            │
│        → 无 sessionId，返回 null                                              │
│      → 用户未登录，继续加载登录页面                                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ 用户输入用户名密码，提交表单
┌─────────────────────────────────────────────────────────────────────────────┐
│  login.tsx action()                                                           │
│    → requireAnonymous(request)  [确保未登录]                                  │
│    → checkHoneypot(formData)    [反机器人检查]                                │
│    → login({ username, password })                                            │
│      → verifyUserPassword()     [验证密码]                                    │
│      → prisma.session.create()   [创建数据库会话]                             │
│        → sessionId = 123                                                       │
│        → expirationDate = 30天后                                               │
│    → handleNewSession({ session, remember })                                  │
│      → 检查用户是否启用 2FA → 未启用                                           │
│      → authSession.set(sessionKey, sessionId)                                 │
│      → authSessionStorage.commitSession()                                      │
│        → 设置 en_session Cookie                                                │
│    → redirect(redirectTo)                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           用户已登录，访问受保护路由                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 登录流程（有 2FA）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        用户访问 /login（已启用 2FA）                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ 输入用户名密码，提交表单
┌─────────────────────────────────────────────────────────────────────────────┐
│  login.tsx action()                                                           │
│    → login() 成功，创建数据库会话 sessionId=123                               │
│    → handleNewSession()                                                       │
│      → 检查用户是否启用 2FA → 已启用                                           │
│      → verifySession.set(unverifiedSessionIdKey, 123)                       │
│      → verifySession.set(rememberKey, true)                                  │
│      → verifySessionStorage.commitSession()                                   │
│        → 设置 en_verification Cookie                                          │
│      → 重定向到 /verify?type=2fa&target=userId                               │
│      │ 注意：此时 en_session Cookie 未设置！                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ 用户被重定向到 /verify
┌─────────────────────────────────────────────────────────────────────────────┐
│  verify.tsx loader()                                                          │
│    → 显示 2FA 代码输入页面                                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ 用户输入 2FA 代码，提交
┌─────────────────────────────────────────────────────────────────────────────┐
│  verify.tsx action()                                                          │
│    → validateRequest()                                                         │
│      → isCodeValid()  [验证 2FA 代码]                                         │
│      → handleLoginTwoFactorVerification()  [关键！]                           │
│        → 从 verifySession 获取 unverifiedSessionId=123                        │
│        → authSession.set(sessionKey, 123)  [转移到正式会话]                   │
│        → authSession.set(verifiedTimeKey, Date.now())                        │
│        → authSessionStorage.commitSession()                                    │
│          → 设置 en_session Cookie                                              │
│        → verifySessionStorage.destroySession()                                 │
│          → 清除 en_verification Cookie                                        │
│        → 重定向到目标页面                                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        2FA 验证完成，用户正式登录                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 后续请求鉴权流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        用户访问 /admin/cache（需要 admin 角色）                │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  root.tsx loader()  [全局执行]                                                │
│    → getUserId(request)                                                       │
│      → 从 Cookie 获取 en_session                                               │
│      → sessionId = 123                                                         │
│      → prisma.session.findUnique({                                             │
│          where: { id: 123, expirationDate: { gt: now } }                     │
│        })                                                                       │
│      → 返回 userId                                                              │
│    → 获取用户详情（包含角色和权限）                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  admin/cache/index.tsx loader()  [路由级执行]                                 │
│    → requireUserWithRole(request, 'admin')                                    │
│      → requireUserId(request)                                                  │
│        → getUserId(request) → 已登录，返回 userId                              │
│      → prisma.user.findFirst({                                                 │
│          where: { id: userId, roles: { some: { name: 'admin' } } }           │
│        })                                                                       │
│      → 用户有 admin 角色，继续执行                                              │
│    → 加载缓存管理页面数据                                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ 用户执行删除缓存操作
┌─────────────────────────────────────────────────────────────────────────────┐
│  admin/cache/index.tsx action()                                               │
│    → requireUserWithRole(request, 'admin')  [再次校验]                        │
│      → 校验通过                                                                 │
│    → 执行缓存删除操作                                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.4 权限检查调用链对比

| 路由 | 验证函数 | 调用链 | 错误处理 |
|------|----------|--------|----------|
| **登录页面** | `requireAnonymous` | `getUserId` → 已登录则重定向 | 重定向到首页 |
| **普通受保护页面** | `requireUserId` | `getUserId` → 未登录则重定向 | 重定向到登录页 |
| **Admin 页面** | `requireUserWithRole` | `requireUserId` → 检查角色 → 无角色则 403 | 返回 403 错误 |
| **笔记删除** | `requireUserWithPermission` | `requireUserId` → 检查权限 → 无权限则 403 | 返回 403 错误 |
| **修改邮箱** | `requireRecentVerification` | `requireUserId` → 检查 2FA 验证时间 | 重定向到 2FA 验证 |

---

## 5. 关键设计决策分析

### 5.1 为什么使用显式调用而非中间件？

Epic Stack 采用**显式调用**验证函数的方式，而非传统的中间件模式。这种设计的优点：

1. **灵活性**：不同路由可以根据需要选择不同的验证级别
   - 公开页面：不需要任何验证
   - 登录页面：需要 `requireAnonymous`
   - 普通用户页面：需要 `requireUserId`
   - 管理后台：需要 `requireUserWithRole`
   - 高敏感操作：需要 `requireRecentVerification`

2. **可读性**：在路由代码中明确看到调用了什么验证函数，一目了然

3. **可测试性**：验证函数是独立的，可以单独测试

4. **类型安全**：TypeScript 可以准确推断验证函数的返回值

### 5.2 为什么使用双重存储（Cookie + 数据库）？

1. **安全性**：
   - Cookie 中只存储会话 ID，不存储敏感信息
   - 实际的会话数据（用户 ID、过期时间）存储在数据库中
   - 服务器可以随时撤销会话（删除数据库记录）

2. **可扩展性**：
   - 数据库存储支持跨服务器会话共享
   - 可以轻松实现"设备管理"功能（查看和撤销其他设备的登录）

3. **可靠性**：
   - 即使 Cookie 被篡改，数据库中的记录才是权威的
   - 每次请求都验证数据库中的会话是否有效

### 5.3 为什么 2FA 流程使用两个独立的 Cookie？

| Cookie 名称 | 用途 | 生命周期 |
|-------------|------|----------|
| `en_session` | 正式登录会话 | 可配置（默认 30 天或会话） |
| `en_verification` | 临时验证会话 | 10 分钟 |

这种设计的优点：

1. **安全性**：
   - 2FA 验证过程中，用户尚未完全认证
   - 不应该设置正式的登录 Cookie
   - 使用独立的临时 Cookie 限制风险

2. **清晰的状态管理**：
   - 有 `en_verification` 但没有 `en_session`：用户正在进行 2FA 验证
   - 有 `en_session`：用户已完全登录
   - 状态判断简单明了

3. **自动清理**：
   - 验证会话只有 10 分钟有效期
   - 如果用户放弃验证，Cookie 会自动过期
   - 不会留下孤儿状态

### 5.4 为什么 loader 和 action 都要进行验证？

在 Admin 路由的例子中，`loader` 和 `action` 都调用了 `requireUserWithRole`。这是因为：

1. **独立执行**：在 React Router 中，`loader` 和 `action` 是独立执行的
   - 页面加载时执行 `loader`
   - 表单提交时执行 `action`（可能不经过 `loader`）

2. **安全原则**：
   - 每个入口点都应该进行自己的安全检查
   - 不要假设其他函数已经做了验证
   - 深度防御原则

3. **不同的验证需求**：
   - 某些路由可能 `loader` 需要一种验证，`action` 需要另一种
   - 例如：公开页面的 `loader` 不需要验证，但 `action`（提交评论）需要验证

---

## 6. 代码索引

### 6.1 核心文件

| 文件路径 | 功能描述 |
|----------|----------|
| `app/utils/session.server.ts` | 会话存储配置（`en_session` Cookie） |
| `app/utils/verification.server.ts` | 验证会话存储配置（`en_verification` Cookie） |
| `app/utils/auth.server.ts` | 核心认证函数（`getUserId`, `requireUserId`, `login`, `logout`） |
| `app/utils/permissions.server.ts` | 权限检查函数（`requireUserWithRole`, `requireUserWithPermission`） |
| `app/routes/_auth/login.server.ts` | 登录流程处理（`handleNewSession`, `handleVerification`） |
| `app/routes/_auth/verify.server.ts` | 验证流程处理（`validateRequest`, `requireRecentVerification`） |
| `app/root.tsx` | 全局身份验证（`loader` 中调用 `getUserId`） |

### 6.2 关键函数位置

| 函数名 | 文件路径 | 行号 |
|--------|----------|------|
| `getUserId` | `app/utils/auth.server.ts` | 29-47 |
| `requireUserId` | `app/utils/auth.server.ts` | 49-67 |
| `requireAnonymous` | `app/utils/auth.server.ts` | 69-74 |
| `login` | `app/utils/auth.server.ts` | 76-93 |
| `logout` | `app/utils/auth.server.ts` | 203-231 |
| `requireUserWithRole` | `app/utils/permissions.server.ts` | 43-60 |
| `requireUserWithPermission` | `app/utils/permissions.server.ts` | 6-41 |
| `handleNewSession` | `app/routes/_auth/login.server.ts` | 17-81 |
| `handleVerification` | `app/routes/_auth/login.server.ts` | 83-137 |
| `shouldRequestTwoFA` | `app/routes/_auth/login.server.ts` | 139-158 |
| `requireRecentVerification` | `app/routes/_auth/verify.server.ts` | 58-74 |
| `validateRequest` | `app/routes/_auth/verify.server.ts` | 140-199 |

### 6.3 示例路由

| 路由路径 | 验证方式 | 说明 |
|----------|----------|------|
| `/login` | `requireAnonymous` | 登录页面，只允许未登录用户访问 |
| `/admin/cache` | `requireUserWithRole('admin')` | 管理后台，需要 admin 角色 |
| `/users/:username/notes/:noteId` | `requireUserId` + `requireUserWithPermission` | 笔记详情，删除操作需要权限 |
| `/settings/profile/change-email` | `requireRecentVerification` | 修改邮箱，需要近期验证 |
