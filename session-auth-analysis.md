# Epic Stack 会话建立与验证机制分析

## 1. 会话存储方式

### 1.1 基于 Cookie 的会话存储

Epic Stack 使用 React Router 的 `createCookieSessionStorage` 来创建会话存储，配置如下：

```typescript
// app/utils/session.server.ts:3-12
export const authSessionStorage = createCookieSessionStorage({
	cookie: {
		name: 'en_session',
		sameSite: 'lax', // CSRF protection is advised if changing to 'none'
		path: '/',
		httpOnly: true,
		secrets: process.env.SESSION_SECRET.split(','),
		secure: process.env.NODE_ENV === 'production',
	},
})
```

**配置说明：**
- **name**: 会话 cookie 名称为 `en_session`
- **sameSite**: 设置为 `lax`，提供一定的 CSRF 保护
- **path**: 设置为 `/`，对整个站点有效
- **httpOnly**: 设置为 `true`，防止 JavaScript 访问，增强安全性
- **secrets**: 从环境变量 `SESSION_SECRET` 中获取，用于签名 cookie
- **secure**: 生产环境下为 `true`，确保 cookie 仅通过 HTTPS 传输

### 1.2 会话过期时间管理

为了解决每次提交会话时覆盖过期时间的问题，Epic Stack 重写了 `commitSession` 方法：

```typescript
// app/utils/session.server.ts:14-38
const originalCommitSession = authSessionStorage.commitSession

Object.defineProperty(authSessionStorage, 'commitSession', {
	value: async function commitSession(
		...args: Parameters<typeof originalCommitSession>
	) {
		const [session, options] = args
		if (options?.expires) {
			session.set('expires', options.expires)
		}
		if (options?.maxAge) {
			session.set('expires', new Date(Date.now() + options.maxAge * 1000))
		}
		const expires = session.has('expires')
			? new Date(session.get('expires'))
			: undefined
		const setCookieHeader = await originalCommitSession(session, {
			...options,
			expires,
		})
		return setCookieHeader
	},
})
```

**实现逻辑：**
- 将会话过期时间存储在会话数据中
- 每次提交会话时，从会话数据中读取过期时间并应用
- 确保会话过期时间不会在每次提交时被重置

### 1.3 数据库中的会话记录

除了 cookie 存储外，Epic Stack 还在数据库中存储会话记录：

```typescript
// app/utils/auth.server.ts:14-18
export const SESSION_EXPIRATION_TIME = 1000 * 60 * 60 * 24 * 30
export const getSessionExpirationDate = () =>
	new Date(Date.now() + SESSION_EXPIRATION_TIME)

export const sessionKey = 'sessionId'
```

**会话记录包含：**
- **id**: 会话唯一标识符
- **userId**: 关联的用户 ID
- **expirationDate**: 会话过期时间（默认 30 天）
- **createdAt**: 会话创建时间

## 2. 登录流程与会话建立

### 2.1 登录验证流程

登录流程主要在 `app/routes/_auth/login.tsx` 和 `app/routes/_auth/login.server.ts` 中实现：

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

**登录步骤：**
1. **验证用户密码**: 调用 `verifyUserPassword` 检查用户名和密码是否匹配
2. **创建会话记录**: 如果验证成功，在数据库中创建会话记录
3. **处理新会话**: 调用 `handleNewSession` 处理会话建立

### 2.2 会话建立过程

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
		// 处理双因素认证
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
	} else {
		// 直接建立会话
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

**会话建立逻辑：**
1. **检查双因素认证**: 查看用户是否启用了双因素认证
2. **双因素认证流程**:
   - 如果启用了双因素认证，将未验证的会话 ID 存储在验证会话中
   - 重定向到双因素认证页面
3. **直接建立会话**:
   - 如果未启用双因素认证，将会话 ID 存储在认证会话中
   - 提交会话到 cookie，设置过期时间（根据 "记住我" 选项）
   - 重定向到目标页面

### 2.3 注册与会话建立

注册流程也会创建会话：

```typescript
// app/utils/auth.server.ts:115-149
export async function signup({
	email,
	username,
	password,
	name,
}: {
	email: User['email']
	username: User['username']
	name: User['name']
	password: string
}) {
	const hashedPassword = await getPasswordHash(password)

	const session = await prisma.session.create({
		data: {
			expirationDate: getSessionExpirationDate(),
			user: {
				create: {
					email: email.toLowerCase(),
					username: username.toLowerCase(),
					name,
					roles: { connect: { name: 'user' } },
					password: {
						create: {
							hash: hashedPassword,
						},
					},
				},
			},
		},
		select: { id: true, expirationDate: true },
	})

	return session
}
```

**注册流程特点：**
- 使用 Prisma 的嵌套创建功能，同时创建用户和会话
- 自动为新用户分配 "user" 角色
- 创建用户后立即创建会话，实现注册即登录

## 3. 每次请求时的身份校验

### 3.1 获取用户 ID

每次请求时，通过 `getUserId` 函数验证用户身份：

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

**身份校验步骤：**
1. **从 Cookie 中获取会话**: 解析请求中的 Cookie，获取认证会话
2. **提取会话 ID**: 从会话中获取 `sessionId`
3. **验证数据库会话**: 在数据库中查找有效的会话记录
   - 检查会话 ID 是否匹配
   - 检查会话是否未过期（`expirationDate > now`）
4. **处理无效会话**:
   - 如果会话无效，重定向到首页
   - 销毁客户端的会话 Cookie

### 3.2 要求用户登录

对于需要登录才能访问的路由，使用 `requireUserId` 函数：

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

**实现逻辑：**
1. **调用 getUserId**: 首先验证用户身份
2. **处理未登录情况**:
   - 如果用户未登录，构建登录重定向 URL
   - 保存当前请求路径作为 `redirectTo` 参数，以便登录后返回
   - 抛出重定向到登录页面

### 3.3 要求用户匿名

对于登录页面等需要用户未登录才能访问的路由，使用 `requireAnonymous` 函数：

```typescript
// app/utils/auth.server.ts:69-74
export async function requireAnonymous(request: Request) {
	const userId = await getUserId(request)
	if (userId) {
		throw redirect('/')
	}
}
```

**实现逻辑：**
1. **调用 getUserId**: 检查用户是否已登录
2. **处理已登录情况**:
   - 如果用户已登录，重定向到首页

### 3.4 根路由中的身份校验

在根路由的 loader 中，会进行全局身份校验：

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

**全局身份校验逻辑：**
1. **获取用户 ID**: 调用 `getUserId` 验证用户身份
2. **获取用户信息**: 如果用户已登录，从数据库获取用户详细信息
   - 包括用户基本信息、头像、角色和权限
3. **处理异常情况**:
   - 如果有用户 ID 但找不到用户记录，注销用户
   - 这可能是因为用户被删除但会话未过期

## 4. 权限判断与路由处理链路

### 4.1 权限系统架构

Epic Stack 采用基于角色的访问控制（RBAC）模型：

- **用户 (User)**: 可以有多个角色
- **角色 (Role)**: 可以有多个权限
- **权限 (Permission)**: 定义对特定实体的操作权限

### 4.2 权限检查函数

#### 4.2.1 检查特定权限

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

**权限检查逻辑：**
1. **要求用户登录**: 首先调用 `requireUserId` 确保用户已登录
2. **解析权限字符串**: 将权限字符串解析为权限数据对象
3. **查询用户权限**: 在数据库中查找具有指定权限的用户
   - 检查用户的角色是否包含所需权限
   - 权限匹配包括实体、操作和访问级别
4. **处理无权限情况**:
   - 如果用户没有所需权限，返回 403 错误
   - 包含错误信息和所需权限详情

#### 4.2.2 检查特定角色

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

**角色检查逻辑：**
1. **要求用户登录**: 首先调用 `requireUserId` 确保用户已登录
2. **查询用户角色**: 在数据库中查找具有指定角色的用户
3. **处理无权限情况**:
   - 如果用户没有所需角色，返回 403 错误
   - 包含错误信息和所需角色详情

### 4.3 路由处理链路上的权限附加

#### 4.3.1 登录页面路由

```typescript
// app/routes/_auth/login.tsx:40-83
export async function loader({ request }: Route.LoaderArgs) {
	await requireAnonymous(request)
	return {}
}

export async function action({ request }: Route.ActionArgs) {
	await requireAnonymous(request)
	// ... 登录处理逻辑
}
```

**实现方式：**
- 在 loader 和 action 中调用 `requireAnonymous`
- 确保只有未登录用户才能访问登录页面

#### 4.3.2 需要登录的路由

以用户设置页面为例：

```typescript
// 从代码结构推断，类似以下实现
export async function loader({ request }: Route.LoaderArgs) {
	await requireUserId(request)
	// ... 加载用户设置数据
}
```

**实现方式：**
- 在 loader 中调用 `requireUserId`
- 确保只有登录用户才能访问

#### 4.3.3 需要特定权限的路由

以管理后台路由为例：

```typescript
// 从代码结构推断，类似以下实现
export async function loader({ request }: Route.LoaderArgs) {
	await requireUserWithPermission(request, 'admin:read')
	// ... 加载管理后台数据
}
```

**实现方式：**
- 在 loader 中调用 `requireUserWithPermission`
- 确保只有具有特定权限的用户才能访问

### 4.4 前端权限检查

前端组件也可以通过 `useOptionalUser` 等钩子获取用户信息，进行权限判断：

```typescript
// app/root.tsx:188-235
function App() {
	const data = useLoaderData<typeof loader>()
	const user = useOptionalUser()
	// ... 其他逻辑
	
	return (
		// ... 组件结构
		<div className="flex items-center gap-10">
			{user ? (
				<UserDropdown />
			) : (
				<Button asChild variant="default" size="lg">
					<Link to="/login">Log In</Link>
				</Button>
			)}
		</div>
		// ... 其他组件
	)
}
```

**前端权限检查特点：**
- 从根 loader 获取用户信息
- 使用 `useOptionalUser` 钩子获取当前用户
- 根据用户是否登录显示不同的 UI 元素
- 前端权限检查仅用于 UI 展示，后端仍需进行权限验证

## 5. 登出流程

### 5.1 登出实现

```typescript
// app/utils/auth.server.ts:203-231
export async function logout(
	{
		request,
		redirectTo = '/',
	}: {
		request: Request
		redirectTo?: string
	},
	responseInit?: ResponseInit,
) {
	const authSession = await authSessionStorage.getSession(
		request.headers.get('cookie'),
	)
	const sessionId = authSession.get(sessionKey)
	// if this fails, we still need to delete the session from the user's browser
	// and it doesn't do any harm staying in the db anyway.
	if (sessionId) {
		// the .catch is important because that's what triggers the query.
		// learn more about PrismaPromise: https://www.prisma.io/docs/orm/reference/prisma-client-reference#prismapromise-behavior
		void prisma.session.deleteMany({ where: { id: sessionId } }).catch(() => {})
	}
	throw redirect(safeRedirect(redirectTo), {
		...responseInit,
		headers: combineHeaders(
			{ 'set-cookie': await authSessionStorage.destroySession(authSession) },
			responseInit?.headers,
		),
	})
}
```

**登出步骤：**
1. **获取会话**: 从 Cookie 中获取认证会话
2. **删除数据库会话**:
   - 尝试从数据库中删除会话记录
   - 使用 `void` 关键字，即使删除失败也不影响登出流程
3. **销毁客户端会话**:
   - 销毁客户端的会话 Cookie
   - 重定向到指定页面（默认为首页）

## 6. 总结

### 6.1 会话管理机制

| 方面 | 实现方式 |
|------|----------|
| 存储位置 | Cookie + 数据库双重存储 |
| Cookie 名称 | `en_session` |
| 会话过期时间 | 默认 30 天 |
| 安全特性 | HttpOnly、SameSite=Lax、生产环境 Secure |

### 6.2 身份验证流程

1. **登录时**:
   - 验证用户名和密码
   - 在数据库中创建会话记录
   - 将会话 ID 存储到 Cookie 中
   - 处理双因素认证（如果启用）

2. **每次请求时**:
   - 从 Cookie 中提取会话 ID
   - 在数据库中验证会话是否有效
   - 检查会话是否过期
   - 获取用户 ID 并返回

3. **登出时**:
   - 从数据库中删除会话记录
   - 销毁客户端的会话 Cookie
   - 重定向到首页

### 6.3 权限控制机制

| 层级 | 实现方式 |
|------|----------|
| 登录验证 | `requireUserId` 函数 |
| 匿名验证 | `requireAnonymous` 函数 |
| 权限验证 | `requireUserWithPermission` 函数 |
| 角色验证 | `requireUserWithRole` 函数 |

### 6.4 路由处理链路

1. **根路由 loader**:
   - 全局身份验证
   - 获取用户信息（包括角色和权限）
   - 处理异常情况

2. **子路由 loader/action**:
   - 根据路由需求调用相应的验证函数
   - `requireAnonymous`：登录页面等
   - `requireUserId`：需要登录的页面
   - `requireUserWithPermission/requireUserWithRole`：需要特定权限的页面

3. **前端组件**:
   - 从根 loader 获取用户信息
   - 根据用户状态显示不同的 UI
   - 前端权限检查仅用于展示，后端仍需验证

## 7. 安全特性

1. **Cookie 安全**:
   - HttpOnly：防止 XSS 攻击
   - SameSite=Lax：防止 CSRF 攻击
   - 生产环境 Secure：仅通过 HTTPS 传输
   - 签名：使用 `SESSION_SECRET` 签名 Cookie

2. **会话管理**:
   - 数据库存储会话，可随时撤销
   - 会话过期时间管理
   - 登出时同时删除数据库和客户端会话

3. **权限控制**:
   - 基于角色的访问控制（RBAC）
   - 后端权限验证，前端仅用于 UI 展示
   - 细粒度的权限控制（实体、操作、访问级别）

4. **双因素认证**:
   - 支持双因素认证
   - 登录时检查是否启用双因素认证
   - 未验证的会话需要进一步验证

这种设计既保证了安全性，又提供了良好的用户体验，同时具有良好的可扩展性和可维护性。
