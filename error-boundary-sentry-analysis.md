# Error Boundary 与 Sentry 集成分析

本文档深入分析 Epic Stack 中三个核心错误处理机制：

1. **404 场景的完整匹配链路**（两种场景：路由不匹配 vs 资源不存在）
2. **各层 ErrorBoundary 的分工、优先级和冒泡机制**
3. **三阶段 Sentry 错误上报的准确文件位置和触发时机**

---

## 一、404 匹配链路深度分析

### 1.1 两种 404 场景

在 Epic Stack 中，404 错误有两种**完全不同**的触发链路：

| 场景 | 触发原因 | 匹配路由 | 抛出方式 | 错误信息示例 |
|------|---------|---------|---------|-------------|
| **A. 路由不匹配** | URL 无任何路由定义匹配 | `$.tsx` (splat) | `throw new Response()` | "We can't find this page: /xxx" |
| **B. 资源不存在** | 路由匹配但数据库查无此资源 | 具体路由 (如 `$noteId.tsx`) | `invariantResponse()` | "No note with the id 'xxx' exists" |

---

### 1.2 场景 A：路由不匹配 → Splat Route 捕获

#### 路由定义

`app/routes/$.tsx` 是一个**根级 splat route（通配符路由）**，作为所有未匹配路由的最后选择：

```typescript
// app/routes/$.tsx:1-6
// This is called a "splat route" and as it's in the root `/app/routes/`
// directory, it's a catchall. If no other routes match, this one will and we
// can know that the user is hitting a URL that doesn't exist. By throwing a
// 404 from the loader, we can force the error boundary to render which will
// ensure the user gets the right status code and we can display a nicer error
// message for them than the Remix and/or browser default.
```

#### Loader 抛出 404

`splat route` 的 `loader` 和 `action` 都直接抛出 `Response` 对象：

```typescript
// app/routes/$.tsx:12-18
export function loader() {
	throw new Response('Not found', { status: 404 })
}

export function action() {
	throw new Response('Not found', { status: 404 })
}
```

#### 完整链路

```
用户访问 /does-not-exist
        ↓
React Router 遍历所有路由定义
        ↓
无精确路由匹配，无动态路由匹配
        ↓
匹配根级 splat route ($.tsx)
        ↓
$.tsx.loader() 执行
        ↓
throw new Response('Not found', { status: 404 })
        ↓
触发当前路由 ($.tsx) 的 ErrorBoundary
        ↓
$.tsx.ErrorBoundary() 被调用
        ↓
渲染 "We can't find this page: /does-not-exist"
```

---

### 1.3 场景 B：路由匹配但资源不存在 → invariantResponse

#### 典型示例：笔记详情页

以 `app/routes/users/$username/notes/$noteId.tsx` 为例：

```typescript
// app/routes/users/$username/notes/$noteId.tsx:24-49
export async function loader({ params }: Route.LoaderArgs) {
	const note = await prisma.note.findUnique({
		where: { id: params.noteId },
		select: { id, title, content, ownerId, updatedAt, images },
	})

	// 关键：使用 invariantResponse 抛出
	invariantResponse(note, 'Not found', { status: 404 })

	return { note, timeAgo }
}
```

#### invariantResponse 是什么？

`invariantResponse` 来自 `@epic-web/invariant` 包，本质上是：
- 如果条件为 falsy，**抛出一个 Response 对象**
- 等价于：`if (!note) throw new Response('Not found', { status: 404 })`

#### 层级路由示例

考虑嵌套路由结构：

```
app/routes/
├── root.tsx                    # 根路由
├── users/
│   ├── index.tsx               # /users
│   └── $username/
│       ├── index.tsx           # /users/:username
│       └── notes/
│           ├── _layout.tsx     # 布局路由 (loader 检查用户是否存在)
│           └── $noteId.tsx     # 叶子路由 (loader 检查笔记是否存在)
└── $.tsx                        # splat route
```

#### 完整链路（笔记不存在场景）

```
用户访问 /users/alice/notes/non-existent-id
        ↓
React Router 路由匹配成功
  → 匹配 routes/users/$username/notes/$noteId.tsx
  → params = { username: 'alice', noteId: 'non-existent-id' }
        ↓
先执行父路由 loader（布局路由）
  → _layout.tsx.loader() 检查用户 alice 是否存在
  → 假设用户存在，继续
        ↓
执行叶子路由 loader
  → $noteId.tsx.loader() 执行
  → prisma.note.findUnique({ id: 'non-existent-id' })
  → 返回 null
        ↓
invariantResponse(note, 'Not found', { status: 404 })
  → 条件不满足 (note 为 null)
  → throw new Response('Not found', { status: 404 })
        ↓
触发 ErrorBoundary 冒泡查找
        ↓
找到最近的 ErrorBoundary: $noteId.tsx.ErrorBoundary()
        ↓
渲染 "No note with the id 'non-existent-id' exists"
```

---

### 1.4 两种 404 场景的关键区别

| 维度 | 场景 A: 路由不匹配 | 场景 B: 资源不存在 |
|------|------------------|------------------|
| **路由匹配阶段** | 路由匹配失败后才匹配 splat | 路由匹配成功 |
| **抛出位置** | `$.tsx` 的 loader | 具体路由的 loader |
| **抛出方式** | `throw new Response()` | `invariantResponse()` (本质相同) |
| **触发的 ErrorBoundary** | `$.tsx.ErrorBoundary` | 具体路由的 `ErrorBoundary` |
| **错误信息** | 显示完整路径名 | 显示具体资源 ID |
| **是否有 params** | 无 (splat route 无命名参数) | 有 (如 `params.noteId`) |

---

## 二、ErrorBoundary 层级、分工与优先级

### 2.1 ErrorBoundary 层级结构

在 React Router 中，每个路由都可以定义自己的 `ErrorBoundary`。层级结构如下：

```
┌─────────────────────────────────────────────────────────────┐
│                    root.tsx ErrorBoundary                    │
│  (最后兜底，使用通用 GeneralErrorBoundary，无定制 statusHandlers)│
└─────────────────────────────────────────────────────────────┘
                              ↑
                              │ 冒泡
                              │
┌─────────────────────────────────────────────────────────────┐
│              布局路由 ErrorBoundary (如 _layout.tsx)          │
│  (处理该布局下的资源不存在，定制用户级 404 信息)                │
└─────────────────────────────────────────────────────────────┘
                              ↑
                              │ 冒泡
                              │
┌─────────────────────────────────────────────────────────────┐
│              叶子路由 ErrorBoundary (如 $noteId.tsx)          │
│  (处理具体资源不存在，定制笔记级 404/403 信息)                  │
└─────────────────────────────────────────────────────────────┘
                              ↑
                              │ 冒泡 (如果叶子路由没有 ErrorBoundary)
                              │
┌─────────────────────────────────────────────────────────────┐
│              splat route ErrorBoundary ($.tsx)               │
│  (仅处理路由不匹配的 404，不是层级结构的一部分，是单独的路由)    │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 冒泡机制与优先级

#### 核心规则

**React Router 的 ErrorBoundary 查找规则**：

1. **从错误抛出的路由开始，向上查找父路由链**
2. **使用第一个找到的 ErrorBoundary**
3. **如果当前路由和所有祖先都没有 ErrorBoundary，使用 root 路由的 ErrorBoundary**

#### 优先级顺序（从高到低）

```
1. 抛出错误的路由自己的 ErrorBoundary (最高优先级)
        ↓
2. 直接父布局路由的 ErrorBoundary
        ↓
3. 更上层的布局路由的 ErrorBoundary
        ↓
4. root.tsx 的 ErrorBoundary (最低优先级，兜底)
```

#### 实际示例分析

以路由 `/users/alice/notes/123` 为例，路由层级：

```
root.tsx
    → users/$username/notes/_layout.tsx (布局路由)
        → users/$username/notes/$noteId.tsx (叶子路由)
```

**场景 1：叶子路由有 ErrorBoundary，笔记不存在**

```
$noteId.tsx.loader() 抛出 Response(404)
        ↓
检查 $noteId.tsx 是否有 ErrorBoundary → 有
        ↓
使用 $noteId.tsx.ErrorBoundary()
        ↓
渲染 "No note with the id '123' exists"
```

**场景 2：叶子路由没有 ErrorBoundary，笔记不存在**

```
$noteId.tsx.loader() 抛出 Response(404)
        ↓
检查 $noteId.tsx 是否有 ErrorBoundary → 没有
        ↓
向上查找父路由 _layout.tsx → 有 ErrorBoundary
        ↓
使用 _layout.tsx.ErrorBoundary()
        ↓
渲染 "No user with the username 'alice' exists" (注意：这是错误的！)
```

**关键点**：这就是为什么**每个路由都应该定义自己的 ErrorBoundary**，否则会显示错误的错误信息。

---

### 2.3 各层 ErrorBoundary 的分工

#### 叶子路由 ErrorBoundary

**职责**：处理具体资源的业务错误

**示例** (`$noteId.tsx`):

```typescript
// app/routes/users/$username/notes/$noteId.tsx:226-237
export function ErrorBoundary() {
	return (
		<GeneralErrorBoundary
			statusHandlers={{
				403: () => <p>You are not allowed to do that</p>,
				404: ({ params }) => (
					<p>No note with the id "{params.noteId}" exists</p>
				),
			}}
		/>
	)
}
```

**特点**：
- 定制化程度最高
- 使用 `params` 显示具体资源 ID
- 处理多种状态码（403、404 等）

#### 布局路由 ErrorBoundary

**职责**：处理布局级别的资源错误

**示例** (`_layout.tsx`):

```typescript
// app/routes/users/$username/notes/_layout.tsx:92-102
export function ErrorBoundary() {
	return (
		<GeneralErrorBoundary
			statusHandlers={{
				404: ({ params }) => (
					<p>No user with the username "{params.username}" exists</p>
				),
			}}
		/>
	)
}
```

**特点**：
- 检查父资源是否存在（如用户是否存在）
- 错误信息是用户级别的
- 作为叶子路由的备份

#### Splat Route ErrorBoundary

**职责**：仅处理路由不匹配的 404

**示例** (`$.tsx`):

```typescript
// app/routes/$.tsx:26-47
export function ErrorBoundary() {
	const location = useLocation()
	return (
		<GeneralErrorBoundary
			statusHandlers={{
				404: () => (
					<div className="flex flex-col gap-6">
						<div className="flex flex-col gap-3">
							<h1>We can't find this page:</h1>
							<pre className="text-body-lg break-all whitespace-pre-wrap">
								{location.pathname}
							</pre>
						</div>
						<Link to="/" className="text-body-md underline">
							<Icon name="arrow-left">Back to home</Icon>
						</Link>
					</div>
				),
			}}
		/>
	)
}
```

**特点**：
- 使用 `useLocation()` 获取完整路径
- 显示 "We can't find this page" 而非具体资源
- 提供返回首页的链接

#### Root ErrorBoundary

**职责**：最后的兜底

**示例** (`root.tsx`):

```typescript
// app/root.tsx:261-263
// this is a last resort error boundary. There's not much useful information we
// can offer at this level.
export const ErrorBoundary = GeneralErrorBoundary
```

**特点**：
- 直接使用 `GeneralErrorBoundary`，无任何定制
- 错误信息是最通用的：`{error.status} {error.data}`
- 只有当所有其他 ErrorBoundary 都不存在时才会触发

---

### 2.4 GeneralErrorBoundary 的两种错误类型处理

`GeneralErrorBoundary` 是所有错误边界的核心组件，区分两种错误类型：

```typescript
// app/components/error-boundary.tsx:16-53
export function GeneralErrorBoundary({
	defaultStatusHandler = ({ error }) => (
		<p>
			{error.status} {error.data}
		</p>
	),
	statusHandlers,
	unexpectedErrorHandler = (error) => <p>{getErrorMessage(error)}</p>,
}: {
	defaultStatusHandler?: StatusHandler
	statusHandlers?: Record<number, StatusHandler>
	unexpectedErrorHandler?: (error: unknown) => ReactElement | null
}) {
	const error = useRouteError()
	const params = useParams()
	const isResponse = isRouteErrorResponse(error)  // 关键判断

	// 控制台输出
	if (typeof document !== 'undefined') {
		console.error(error)
	}

	// 仅非响应错误上报 Sentry
	useEffect(() => {
		if (isResponse) return
		captureException(error)
	}, [error, isResponse])

	// 渲染逻辑
	return (
		<div className="text-h2 container flex items-center justify-center p-20">
			{isResponse
				? (statusHandlers?.[error.status] ?? defaultStatusHandler)({
						error,
						params,
					})
				: unexpectedErrorHandler(error)}
		</div>
	)
}
```

#### 两种错误类型对比

| 维度 | 路由响应错误 (Route Error Response) | 未知异常 (Unexpected Error) |
|------|-------------------------------------|-----------------------------|
| **判断条件** | `isRouteErrorResponse(error) === true` | `isRouteErrorResponse(error) === false` |
| **触发方式** | `throw new Response()` 或 `invariantResponse()` | `throw new Error()` 或 `throw "string"` 或运行时异常 |
| **包含属性** | `status`, `statusText`, `data` | 任意 JavaScript 值 |
| **渲染方式** | `statusHandlers[status]` 或 `defaultStatusHandler` | `unexpectedErrorHandler` |
| **是否上报 Sentry** | ❌ 不上报（预期行为） | ✅ 上报（程序错误） |

#### isRouteErrorResponse 的判断逻辑

`isRouteErrorResponse` 是 React Router 提供的工具函数，判断标准：

```typescript
// 伪代码
function isRouteErrorResponse(error: unknown): boolean {
	return (
		typeof error === 'object' &&
		error !== null &&
		'status' in error &&
		typeof error.status === 'number' &&
		'statusText' in error &&
		typeof error.statusText === 'string' &&
		'data' in error
	)
}
```

**关键结论**：`throw new Response()` 抛出的对象会被 React Router 包装成 `ErrorResponse` 对象，满足上述条件。

---

## 三、三阶段 Sentry 错误上报机制

### 3.1 准确的文件位置

用户指出的问题：**进程关闭阶段的文件信息需要澄清**

让我明确各阶段的**准确文件位置**：

| 阶段 | 文件 | 说明 |
|------|------|------|
| **阶段 1: 服务端 Loader/Action 错误** | `app/entry.server.tsx` | React Router 的 `handleError` 钩子 |
| **阶段 2: 客户端 ErrorBoundary 错误** | `app/components/error-boundary.tsx` | `GeneralErrorBoundary` 中的 `useEffect` |
| **阶段 3: 进程优雅关闭错误** | `server/index.ts` | Express 服务器入口，`closeWithGrace` 回调 |

**重要澄清**：
- `server/index.ts` 是**开发环境和生产环境的服务器入口文件**
- 生产环境构建时，`server/index.ts` 会被编译到 `build/server/index.js`
- 但 `closeWithGrace` 的逻辑**始终在 `server/index.ts` 中定义**

---

### 3.2 阶段 1：服务端 Loader/Action 错误

#### 文件位置

```
app/entry.server.tsx
```

#### 触发时机

当**服务端渲染**时，loader 或 action 抛出任何错误（包括 `throw Response` 和 `throw Error`），React Router 会调用 `handleError` 钩子。

#### 代码实现

```typescript
// app/entry.server.tsx:125-142
export function handleError(
	error: unknown,
	{ request }: LoaderFunctionArgs | ActionFunctionArgs,
): void {
	// 跳过请求被中止的情况（Remix 文档建议）
	if (request.signal.aborted) {
		return
	}

	if (error instanceof Error) {
		console.error(styleText('red', String(error.stack)))
	} else {
		console.error(error)
	}

	Sentry.captureException(error)  // 所有错误都上报！
}
```

#### 关键点

| 特性 | 说明 |
|------|------|
| **触发条件** | 仅服务端渲染时 |
| **上报范围** | **所有错误**（包括 `Response` 和 `Error`） |
| **过滤条件** | 仅跳过 `request.signal.aborted`（请求被用户中止） |
| **与阶段 2 的区别** | 阶段 2 只上报非响应错误，阶段 1 上报所有错误 |

**为什么阶段 1 上报所有错误？**

- 服务端错误需要全面监控
- 即使是 404，服务端也可能需要知道有多少无效请求
- `request.signal.aborted` 是唯一例外，因为这是用户主动行为

---

### 3.3 阶段 2：客户端 ErrorBoundary 中的未知异常

#### 文件位置

```
app/components/error-boundary.tsx
```

#### 触发时机

当**客户端导航**时发生错误，或者服务端渲染的错误在客户端 hydration 后显示时。

#### 代码实现

```typescript
// app/components/error-boundary.tsx:36-41
useEffect(() => {
	if (isResponse) return  // 路由响应错误不上报
	captureException(error)   // 只有未知异常上报
}, [error, isResponse])
```

#### 关键点

| 特性 | 说明 |
|------|------|
| **触发条件** | 客户端渲染/导航时 |
| **上报范围** | **仅非响应错误**（`isRouteErrorResponse(error) === false`） |
| **过滤条件** | `throw Response` / `invariantResponse()` 不上报 |
| **与阶段 1 的区别** | 阶段 1 是服务端钩子，阶段 2 是客户端 React 组件 |

**为什么阶段 2 不上报响应错误？**

- `throw Response(status=404/403)` 是**预期的业务行为**，不是程序 bug
- 客户端 404 通常是用户输入错误或链接失效，不需要告警
- 只有真正的 `throw new Error()` 才是需要排查的问题

---

### 3.4 阶段 3：进程优雅关闭错误

#### 文件位置

```
server/index.ts
```

#### 触发时机

当进程收到 `SIGTERM` 或 `SIGINT` 信号（如 `Ctrl+C` 或部署平台通知）时，`close-with-grace` 库会触发回调。

#### 代码实现

```typescript
// server/index.ts:5
import closeWithGrace from 'close-with-grace'

// server/index.ts:236-248
closeWithGrace(async ({ err }) => {
	// 步骤 1: 优雅关闭 HTTP 服务器
	await new Promise((resolve, reject) => {
		server.close((e) => (e ? reject(e) : resolve('ok')))
	})
	
	// 步骤 2: 如果有关闭错误，上报 Sentry
	if (err) {
		console.error(styleText('red', String(err)))
		console.error(styleText('red', String(err.stack)))
		if (SENTRY_ENABLED) {
			Sentry.captureException(err)
			await Sentry.flush(500)  // 关键：确保事件发送完成
		}
	}
})
```

#### `closeWithGrace` 的 `err` 参数来源

`close-with-grace` 库的 `err` 参数在以下情况有值：

1. **未捕获的异常** (`uncaughtException`)
2. **未处理的 Promise rejection** (`unhandledRejection`)
3. **其他致命错误**

#### 关键点

| 特性 | 说明 |
|------|------|
| **触发条件** | 进程收到 `SIGTERM` / `SIGINT` 信号 |
| **上报范围** | 仅关闭过程中发生的错误（`err` 参数） |
| **特殊处理** | `Sentry.flush(500)` - 等待最多 500ms 确保事件发送 |
| **为什么需要 flush** | 进程即将退出，Sentry 的异步事件队列可能还未处理 |

#### 与 `tests/mocks/index.ts` 的区别

项目中还有另一处 `closeWithGrace`：

```typescript
// tests/mocks/index.ts:36-38
closeWithGrace(() => {
	server.close()
})
```

**区别**：
- `server/index.ts`: **生产/开发服务器**的优雅关闭，有 Sentry 上报
- `tests/mocks/index.ts`: **测试 Mock 服务器**的关闭，仅关闭 MSW，无 Sentry

---

### 3.5 三阶段完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           错误场景与上报阶段                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────┐                                                        │
│  │  服务端渲染场景    │                                                        │
│  └────────┬─────────┘                                                        │
│           │                                                                   │
│           ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐          │
│  │ 阶段 1: entry.server.tsx handleError()                       │          │
│  │                                                                │          │
│  │ 触发: loader/action 抛出任何错误                               │          │
│  │ 上报: Sentry.captureException(error) ← 所有错误都上报         │          │
│  │ 过滤: 仅跳过 request.signal.aborted                           │          │
│  └──────────────────────────────────────────────────────────────┘          │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────┐                                                        │
│  │  客户端导航场景    │                                                        │
│  └────────┬─────────┘                                                        │
│           │                                                                   │
│           ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐          │
│  │ 阶段 2: error-boundary.tsx useEffect()                        │          │
│  │                                                                │          │
│  │ 触发: useRouteError() 捕获到错误                              │          │
│  │ 判断: isRouteErrorResponse(error)                             │          │
│  │       ├── true  → throw Response() → 不上报 (预期行为)        │          │
│  │       └── false → throw Error()   → 上报 (程序错误)          │          │
│  │ 上报: Sentry.captureException(error) ← 仅非响应错误           │          │
│  └──────────────────────────────────────────────────────────────┘          │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────┐                                                        │
│  │  进程关闭场景      │                                                        │
│  └────────┬─────────┘                                                        │
│           │                                                                   │
│           ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐          │
│  │ 阶段 3: server/index.ts closeWithGrace()                      │          │
│  │                                                                │          │
│  │ 触发: 收到 SIGTERM/SIGINT 信号                                 │          │
│  │ 参数: err ← 未捕获异常 / unhandled rejection                  │          │
│  │ 步骤: 1. server.close()                                       │          │
│  │       2. if (err) {                                           │          │
│  │            Sentry.captureException(err)                       │          │
│  │            await Sentry.flush(500) ← 等待事件发送             │          │
│  │          }                                                     │          │
│  └──────────────────────────────────────────────────────────────┘          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 3.6 阶段 1 与阶段 2 的协作

**重要理解**：一个错误可能被**两个阶段都处理**，也可能**只被一个阶段处理**。

#### 场景 A：服务端渲染时 loader 抛出 Error

```
用户首次访问页面 (SSR)
        ↓
服务端执行 loader
        ↓
throw new Error("数据库连接失败")
        ↓
阶段 1: entry.server.tsx handleError() 被调用
        ↓
Sentry.captureException(error) ← 阶段 1 上报
        ↓
服务端渲染 ErrorBoundary 页面返回给浏览器
        ↓
浏览器 hydration
        ↓
阶段 2: error-boundary.tsx useEffect() 执行
        ↓
isRouteErrorResponse(error) === false
        ↓
Sentry.captureException(error) ← 阶段 2 也上报！
```

**结果**：同一个错误被上报两次。

**为什么这样设计？**

- 阶段 1：确保服务端错误立即被捕获
- 阶段 2：确保客户端导航时的错误也能被捕获
- 重复上报在 Sentry 中会被自动去重（基于错误指纹）

#### 场景 B：服务端渲染时 loader 抛出 Response

```
用户首次访问页面 (SSR)
        ↓
服务端执行 loader
        ↓
throw new Response("Not found", { status: 404 })
        ↓
阶段 1: entry.server.tsx handleError() 被调用
        ↓
Sentry.captureException(error) ← 阶段 1 上报（所有错误都上报）
        ↓
服务端渲染 ErrorBoundary 页面返回给浏览器
        ↓
浏览器 hydration
        ↓
阶段 2: error-boundary.tsx useEffect() 执行
        ↓
isRouteErrorResponse(error) === true
        ↓
return  ← 阶段 2 不上报！
```

**结果**：只在阶段 1 上报一次。

---

## 四、关键设计决策深度解析

### 4.1 为什么用 throw Response 而不是直接渲染？

**核心原因：HTTP 状态码的正确性**

```
传统方式（错误）：
用户访问 /non-existent
        ↓
React Router 匹配 splat route
        ↓
loader 返回数据，组件渲染 404 页面
        ↓
HTTP 状态码 = 200 ❌
        ↓
搜索引擎认为这是一个有效页面
        ↓
SEO 灾难！
```

```
Epic Stack 方式（正确）：
用户访问 /non-existent
        ↓
React Router 匹配 splat route
        ↓
loader throw Response(status=404)
        ↓
ErrorBoundary 渲染
        ↓
HTTP 状态码 = 404 ✅
        ↓
搜索引擎正确识别为不存在的页面
```

**额外优势**：
1. **统一错误处理**：所有错误都走 ErrorBoundary 机制
2. **层级冒泡**：利用 React Router 的 ErrorBoundary 冒泡机制
3. **状态码语义**：404、403、500 等状态码有明确语义

---

### 4.2 为什么路由响应错误不上报 Sentry（阶段 2）？

```typescript
// app/components/error-boundary.tsx:37-41
useEffect(() => {
	if (isResponse) return  // 跳过
	captureException(error)
}, [error, isResponse])
```

**哲学区分：预期行为 vs 程序错误**

| 类型 | 示例 | 是否上报 | 原因 |
|------|------|---------|------|
| **预期行为** | `throw Response(404)` | ❌ | 用户访问不存在的资源是正常的 |
| **预期行为** | `throw Response(403)` | ❌ | 权限不足是业务逻辑的正常分支 |
| **程序错误** | `throw new Error("bug")` | ✅ | 代码有 bug，需要修复 |
| **程序错误** | `undefined.property` | ✅ | 运行时异常，需要修复 |

**为什么阶段 1 又上报？**

- 服务端需要全面监控
- 即使是 404，也可能是配置错误或攻击
- 服务端和客户端有不同的监控策略

---

### 4.3 为什么进程关闭需要 flush？

```typescript
await Sentry.flush(500)
```

**问题：Sentry SDK 是异步的**

```
正常运行时：
throw new Error("some error")
        ↓
Sentry.captureException(error)
        ↓
SDK 将事件放入内存队列
        ↓
Sentry.captureException() 立即返回（不等待）
        ↓
后台线程异步发送到 Sentry 服务器
        ↓
事件最终到达
```

```
进程关闭时：
closeWithGrace 回调执行
        ↓
Sentry.captureException(err)
        ↓
SDK 将事件放入内存队列
        ↓
Sentry.captureException() 立即返回
        ↓
进程退出 ❌
        ↓
内存队列被销毁
        ↓
事件丢失！
```

**解决方案：flush**

```
await Sentry.flush(500)
        ↓
等待队列中的事件发送完成
        ↓
最多等待 500ms
        ↓
返回 Promise（resolved = 发送完成，rejected = 超时）
        ↓
然后进程才退出
        ↓
事件不丢失 ✅
```

---

## 五、完整文件清单与职责

| 文件路径 | 核心职责 | 关键函数/导出 |
|---------|---------|--------------|
| `app/routes/$.tsx` | Splat 路由，处理路由不匹配的 404 | `loader()`, `ErrorBoundary()` |
| `app/routes/*/$noteId.tsx` | 叶子路由，处理具体资源错误 | `loader()`, `ErrorBoundary()` |
| `app/routes/*/_layout.tsx` | 布局路由，处理父资源错误 | `loader()`, `ErrorBoundary()` |
| `app/root.tsx` | 根路由，兜底 ErrorBoundary | `ErrorBoundary = GeneralErrorBoundary` |
| `app/components/error-boundary.tsx` | 通用错误边界组件 | `GeneralErrorBoundary()` |
| `app/entry.server.tsx` | 服务端入口，错误钩子 | `handleError()` |
| `app/entry.client.tsx` | 客户端入口，Sentry 初始化 | 条件导入 `monitoring.client` |
| `server/index.ts` | Express 服务器入口 | `closeWithGrace()` |
| `server/utils/monitoring.ts` | 服务端 Sentry 初始化 | `init()` |
| `app/utils/monitoring.client.tsx` | 客户端 Sentry 初始化 | `init()` |
| `tests/mocks/index.ts` | 测试 Mock 服务器 | `closeWithGrace()` (无 Sentry) |

---

## 六、测试验证

### 6.1 E2E 测试验证 404 机制

```typescript
// tests/e2e/error-boundary.test.ts:1-9
test('Test root error boundary caught', async ({ page, navigate }) => {
	const pageUrl = '/does-not-exist'
	const res = await navigate(pageUrl as any)

	expect(res?.status()).toBe(404)
	await expect(page.getByText(/We can't find this page/i)).toBeVisible()
})
```

**验证点**：
1. ✅ HTTP 状态码为 404（不是 200）
2. ✅ 页面显示 splat route 的 404 信息

### 6.2 两种 404 场景的测试建议

```typescript
// 场景 A：路由不匹配
test('404 for non-matching route', async ({ page }) => {
	const res = await page.goto('/this-route-does-not-exist')
	expect(res?.status()).toBe(404)
	await expect(page.getByText(/We can't find this page/i)).toBeVisible()
})

// 场景 B：资源不存在
test('404 for non-existing note', async ({ page }) => {
	// 路由匹配成功，但笔记 ID 不存在
	const res = await page.goto('/users/alice/notes/non-existent-id')
	expect(res?.status()).toBe(404)
	await expect(page.getByText(/No note with the id/i)).toBeVisible()
})
```

---

## 七、总结

### 7.1 核心架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              404 错误链路                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  场景 A: 路由不匹配                    场景 B: 资源不存在                      │
│  ┌─────────────────────┐              ┌─────────────────────┐               │
│  │  用户访问 /invalid   │              │ 用户访问 /users/...  │               │
│  └──────────┬──────────┘              └──────────┬──────────┘               │
│             │                                      │                            │
│             ▼                                      ▼                            │
│  ┌─────────────────────┐              ┌─────────────────────┐               │
│  │  路由匹配失败         │              │  路由匹配成功         │               │
│  └──────────┬──────────┘              └──────────┬──────────┘               │
│             │                                      │                            │
│             ▼                                      ▼                            │
│  ┌─────────────────────┐              ┌─────────────────────┐               │
│  │  匹配 $.tsx (splat) │              │  执行具体 loader     │               │
│  └──────────┬──────────┘              └──────────┬──────────┘               │
│             │                                      │                            │
│             ▼                                      ▼                            │
│  ┌─────────────────────┐              ┌─────────────────────┐               │
│  │  $.tsx.loader()     │              │  invariantResponse() │               │
│  │  throw Response(404)│              │  throw Response(404) │               │
│  └──────────┬──────────┘              └──────────┬──────────┘               │
│             │                                      │                            │
│             ▼                                      ▼                            │
│  ┌─────────────────────┐              ┌─────────────────────┐               │
│  │  $.tsx.ErrorBoundary│              │  $noteId.ErrorBoundary│              │
│  │  "We can't find..." │              │  "No note with id..."│              │
│  └─────────────────────┘              └─────────────────────┘               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                           ErrorBoundary 层级                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   优先级从低到高：                                                            │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  root.tsx ErrorBoundary (兜底)                                        │  │
│   │  export const ErrorBoundary = GeneralErrorBoundary                   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                       ↑                                       │
│                                       │ 冒泡                                  │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  布局路由 _layout.tsx ErrorBoundary                                   │  │
│   │  404: "No user with the username 'xxx' exists"                       │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                       ↑                                       │
│                                       │ 冒泡                                  │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  叶子路由 $noteId.tsx ErrorBoundary (最高优先级)                      │  │
│   │  403: "You are not allowed to do that"                               │  │
│   │  404: "No note with the id 'xxx' exists"                             │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                           Sentry 三阶段上报                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  阶段 1: app/entry.server.tsx handleError()                          │  │
│   │  触发: 服务端 loader/action 抛出任何错误                               │  │
│   │  上报: 所有错误 (包括 Response 和 Error)                               │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  阶段 2: app/components/error-boundary.tsx useEffect()               │  │
│   │  触发: 客户端 useRouteError() 捕获到错误                              │  │
│   │  上报: 仅非响应错误 (isRouteErrorResponse === false)                  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  阶段 3: server/index.ts closeWithGrace()                             │  │
│   │  触发: 收到 SIGTERM/SIGINT 信号                                       │  │
│   │  上报: err 参数 (未捕获异常 / unhandled rejection)                    │  │
│   │  特殊: await Sentry.flush(500) 确保事件发送                           │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 关键要点速查

| 问题 | 答案 |
|------|------|
| 404 有几种场景？ | 2 种：路由不匹配 ($.tsx)、资源不存在 (具体路由) |
| ErrorBoundary 优先级？ | 叶子路由 → 布局路由 → root (从高到低) |
| 阶段 1 上报哪些错误？ | 所有错误（包括 Response） |
| 阶段 2 上报哪些错误？ | 仅非响应错误（isRouteErrorResponse === false） |
| 为什么需要 flush？ | 进程即将退出，确保异步事件发送完成 |
| 进程关闭的文件位置？ | `server/index.ts`（不是构建产物） |
