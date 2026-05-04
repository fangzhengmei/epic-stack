# Error Boundary 与 Sentry 集成分析

本文档分析 Epic Stack 中三个核心错误处理机制：

1. **未匹配路由如何通过 splat loader 抛 404**
2. **路由错误和未知异常如何在通用边界中渲染**
3. **客户端、服务端和进程关闭阶段如何把错误送到 Sentry**

---

## 一、未匹配路由通过 Splat Loader 抛 404

### 1.1 Splat Route 定义

在 `app/routes/$.tsx` 中定义了一个 **splat route（通配符路由）**：

```typescript
// app/routes/$.tsx:1-6
// This is called a "splat route" and as it's in the root `/app/routes/`
// directory, it's a catchall. If no other routes match, this one will and we
// can know that the user is hitting a URL that doesn't exist. By throwing a
// 404 from the loader, we can force the error boundary to render which will
// ensure the user gets the right status code and we can display a nicer error
// message for them than the Remix and/or browser default.
```

### 1.2 Loader 抛出 404 响应

`loader` 和 `action` 都直接抛出 `Response` 对象，状态码为 404：

```typescript
// app/routes/$.tsx:12-18
export function loader() {
	throw new Response('Not found', { status: 404 })
}

export function action() {
	throw new Response('Not found', { status: 404 })
}
```

### 1.3 ErrorBoundary 捕获并渲染

路由定义了自己的 `ErrorBoundary` 组件，使用 `GeneralErrorBoundary` 并针对 404 状态码定制渲染：

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

### 1.4 工作流程

```
用户访问 /does-not-exist
        ↓
React Router 路由匹配失败
        ↓
Splat route ($.tsx) 作为 catchall 匹配
        ↓
loader() 执行，抛出 Response(status=404)
        ↓
ErrorBoundary 被触发
        ↓
GeneralErrorBoundary 检测到 isRouteErrorResponse=true
        ↓
调用 statusHandlers[404] 渲染友好的 404 页面
```

---

## 二、路由错误和未知异常在通用边界中渲染

### 2.1 GeneralErrorBoundary 组件

核心组件位于 `app/components/error-boundary.tsx`，处理两种类型的错误：

1. **路由响应错误** (`isRouteErrorResponse`) - 如 `throw new Response()`
2. **未知异常** - 普通 JavaScript Error 对象

### 2.2 类型定义

```typescript
// app/components/error-boundary.tsx:11-14
type StatusHandler = (info: {
	error: ErrorResponse
	params: Record<string, string | undefined>
}) => ReactElement | null
```

### 2.3 组件实现

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
	const isResponse = isRouteErrorResponse(error)

	if (typeof document !== 'undefined') {
		console.error(error)
	}

	useEffect(() => {
		if (isResponse) return
		captureException(error)
	}, [error, isResponse])

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

### 2.4 渲染逻辑

| 错误类型 | 判断条件 | 渲染方式 |
|---------|---------|---------|
| 路由响应错误 | `isRouteErrorResponse(error) === true` | `statusHandlers[status]` 或 `defaultStatusHandler` |
| 未知异常 | `isRouteErrorResponse(error) === false` | `unexpectedErrorHandler` |

### 2.5 Root ErrorBoundary 兜底

`app/root.tsx` 定义了最后的兜底错误边界：

```typescript
// app/root.tsx:261-263
// this is a last resort error boundary. There's not much useful information we
// can offer at this level.
export const ErrorBoundary = GeneralErrorBoundary
```

---

## 三、错误上报 Sentry 机制

### 3.1 初始化配置

#### 服务端初始化

```typescript
// server/utils/monitoring.ts:1-42
export function init() {
	Sentry.init({
		dsn: process.env.SENTRY_DSN,
		environment: process.env.NODE_ENV,
		denyUrls: [
			/\/resources\/healthcheck/,
			/\/build\//,
			/\/favicons\//,
			// ...
		],
		integrations: [
			Sentry.prismaIntegration({
				prismaInstrumentation: new PrismaInstrumentation(),
			}),
			Sentry.httpIntegration(),
			nodeProfilingIntegration(),
		],
		// ...
	})
}
```

#### 客户端初始化

```typescript
// app/utils/monitoring.client.tsx:1-34
export function init() {
	Sentry.init({
		dsn: ENV.SENTRY_DSN,
		environment: ENV.MODE,
		beforeSend(event) {
			// 过滤浏览器扩展错误
			if (event.request?.url) {
				const url = new URL(event.request.url)
				if (
					url.protocol === 'chrome-extension:' ||
					url.protocol === 'moz-extension:'
				) {
					return null
				}
			}
			return event
		},
		integrations: [
			Sentry.replayIntegration(),
			Sentry.browserProfilingIntegration(),
		],
		// ...
	})
}
```

### 3.2 条件初始化

#### 服务端

```typescript
// server/index.ts:16-21
const SENTRY_ENABLED = IS_PROD && process.env.SENTRY_DSN
const BUILD_PATH = '../build/server/index.js'

if (SENTRY_ENABLED) {
	void import('./utils/monitoring.ts').then(({ init }) => init())
}
```

#### 客户端

```typescript
// app/entry.client.tsx:5-7
if (ENV.MODE === 'production' && ENV.SENTRY_DSN) {
	void import('./utils/monitoring.client.tsx').then(({ init }) => init())
}
```

### 3.3 三阶段错误上报

#### 阶段 1: 服务端 Loader/Action 错误

在 `entry.server.tsx` 的 `handleError` 钩子中捕获：

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

	Sentry.captureException(error)
}
```

**触发时机**：当 loader 或 action 抛出异常时，React Router 会调用此钩子。

#### 阶段 2: 客户端 ErrorBoundary 中的未知异常

在 `GeneralErrorBoundary` 的 `useEffect` 中捕获：

```typescript
// app/components/error-boundary.tsx:37-41
useEffect(() => {
	if (isResponse) return  // 路由响应错误不上报
	captureException(error)   // 只有未知异常上报
}, [error, isResponse])
```

**关键点**：
- `isResponse` 判断：`throw new Response()` 类型的错误**不上报** Sentry
- 只有非响应错误（普通 Error 对象）才会上报

#### 阶段 3: 进程优雅关闭阶段

使用 `close-with-grace` 库处理优雅关闭：

```typescript
// server/index.ts:236-248
closeWithGrace(async ({ err }) => {
	// 1. 关闭 HTTP 服务器
	await new Promise((resolve, reject) => {
		server.close((e) => (e ? reject(e) : resolve('ok')))
	})
	
	// 2. 如果有关闭错误，上报 Sentry
	if (err) {
		console.error(styleText('red', String(err)))
		console.error(styleText('red', String(err.stack)))
		if (SENTRY_ENABLED) {
			Sentry.captureException(err)
			await Sentry.flush(500)  // 确保事件发送完成
		}
	}
})
```

**关键点**：
- `Sentry.flush(500)` - 等待最多 500ms 确保事件发送完成
- 只有在 `SENTRY_ENABLED` 为 true 时才执行

### 3.4 错误上报流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        错误来源                                   │
├─────────────────────┬─────────────────────┬─────────────────────┤
│   Loader/Action     │   客户端渲染        │   进程关闭          │
│   (服务端)          │   (客户端)          │   (服务端)          │
├─────────────────────┼─────────────────────┼─────────────────────┤
│  entry.server.tsx   │  error-boundary.tsx │  server/index.tsx   │
│  handleError()      │  useEffect()        │  closeWithGrace()   │
├─────────────────────┼─────────────────────┼─────────────────────┤
│                     │                     │                     │
│  Sentry.capture     │  Sentry.capture     │  Sentry.capture     │
│  Exception(error)   │  Exception(error)   │  Exception(err)     │
│                     │  (仅非响应错误)      │  + flush(500)       │
│                     │                     │                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │   Sentry Server │
                    └─────────────────┘
```

---

## 四、关键设计决策

### 4.1 为什么 Splat Route 用 throw Response 而不是直接渲染？

```
优势：
1. ✅ 正确的 HTTP 状态码（404 而非 200）
2. ✅ 统一走 ErrorBoundary 机制
3. ✅ 可以利用 GeneralErrorBoundary 的 statusHandlers
4. ✅ 搜索引擎爬虫能正确识别 404
```

### 4.2 为什么路由响应错误不上报 Sentry？

```typescript
// app/components/error-boundary.tsx:37-41
useEffect(() => {
	if (isResponse) return  // 跳过
	captureException(error)
}, [error, isResponse])
```

**原因**：
- `throw new Response(status=404)` 是**预期行为**，不是程序错误
- 404、403 等状态码是业务逻辑的正常分支
- 只有真正的 `Error` 对象（如 `throw new Error("bug")`）才是需要排查的问题

### 4.3 为什么进程关闭需要 flush？

```typescript
await Sentry.flush(500)
```

**原因**：
- Sentry SDK 是异步发送事件的
- 进程即将退出，事件队列可能还没处理
- `flush` 等待事件发送完成（最多 500ms）
- 确保关闭错误不会丢失

---

## 五、相关文件清单

| 文件 | 职责 |
|------|------|
| `app/routes/$.tsx` | Splat route，404 处理入口 |
| `app/components/error-boundary.tsx` | 通用错误边界组件 |
| `app/root.tsx` | 根路由，兜底 ErrorBoundary |
| `server/utils/monitoring.ts` | 服务端 Sentry 初始化 |
| `app/utils/monitoring.client.tsx` | 客户端 Sentry 初始化 |
| `app/entry.server.tsx` | 服务端入口，handleError 钩子 |
| `app/entry.client.tsx` | 客户端入口，条件初始化 Sentry |
| `server/index.ts` | Express 服务器，优雅关闭处理 |

---

## 六、测试验证

E2E 测试验证 404 机制：

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
1. HTTP 状态码为 404
2. 页面显示友好的 404 消息
