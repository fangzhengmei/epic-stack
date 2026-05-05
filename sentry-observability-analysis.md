# Epic Stack Sentry 可观测性分析报告

## 一、概述

Epic Stack 采用了分层式的 Sentry 集成架构，实现了服务端、客户端的全链路错误监控和性能追踪。本报告深入分析各运行环境的错误捕获机制、上下文传递方式以及跨环境关联策略。

---

## 二、架构概览

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Sentry 集成架构                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌──────────────┐          ┌─────────────────┐          ┌───────────┐  │
│  │  客户端 (Browser) │          │   构建管道 (CI/CD)  │          │ 服务端(Node) │  │
│  └──────┬───────┘          └────────┬────────┘          └─────┬─────┘  │
│         │                            │                         │         │
│         ▼                            ▼                         ▼         │
│  ┌─────────────────┐         ┌───────────────┐        ┌────────────────┐│
│  │ monitoring.client.tsx  │         │ vite.config.ts  │        │ monitoring.ts  ││
│  │ entry.client.tsx      │         │ react-router.config.ts │    │ server/index.ts││
│  │ error-boundary.tsx    │         │               │        │ entry.server.tsx││
│  └─────────────────┘         └───────────────┘        └────────────────┘│
│         │                            │                         │         │
│         └────────────────────────────┼─────────────────────────┘         │
│                                      ▼                                   │
│                          ┌─────────────────────┐                          │
│                          │    Sentry Cloud     │                          │
│                          │ (错误聚合 + 性能监控)  │                          │
│                          └─────────────────────┘                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心依赖

```typescript
// 主 SDK
@sentry/react-router  // 统一的 React Router SDK，服务端客户端共用

// 服务端专用
@sentry/profiling-node    // Node.js 性能分析
@prisma/instrumentation    // Prisma 数据库查询追踪

// 构建阶段
@sentry/react-router (Vite 插件)  // Source Map 上传、Release 管理
```

---

## 三、运行环境错误捕获机制

### 3.1 服务端 (Node.js 环境)

#### 3.1.1 初始化入口

服务端 Sentry 初始化位于 `server/index.ts`：

```typescript
// server/index.ts:16-21
const SENTRY_ENABLED = IS_PROD && process.env.SENTRY_DSN
const BUILD_PATH = '../build/server/index.js'

if (SENTRY_ENABLED) {
	void import('./utils/monitoring.ts').then(({ init }) => init())
}
```

**关键特性**：
- 仅在生产环境且配置了 `SENTRY_DSN` 时才初始化
- 采用动态导入 (`import()`) 避免开发环境依赖

#### 3.1.2 服务端配置详解

```typescript
// server/utils/monitoring.ts
export function init() {
	Sentry.init({
		dsn: process.env.SENTRY_DSN,
		environment: process.env.NODE_ENV,
		
		// 忽略静态资源和健康检查路由
		denyUrls: [
			/\/resources\/healthcheck/,
			/\/build\//,
			/\/favicons\//,
			/\/img\//,
			/\/fonts\//,
			/\/favicon.ico/,
			/\/site\.webmanifest/,
		],
		
		// 核心集成
		integrations: [
			// Prisma 数据库查询追踪
			Sentry.prismaIntegration({
				prismaInstrumentation: new PrismaInstrumentation(),
			}),
			// HTTP 请求追踪
			Sentry.httpIntegration(),
			// Node.js 性能分析
			nodeProfilingIntegration(),
		],
		
		// 采样策略
		tracesSampler(samplingContext) {
			// 忽略健康检查
			if (samplingContext.request?.url?.includes('/resources/healthcheck')) {
				return 0
			}
			// 生产环境 100% 采样，开发环境 0%
			return process.env.NODE_ENV === 'production' ? 1 : 0
		},
		
		// 事务过滤
		beforeSendTransaction(event) {
			if (event.request?.headers?.['x-healthcheck'] === 'true') {
				return null  // 过滤健康检查事务
			}
			return event
		},
	})
}
```

#### 3.1.3 Loader/Action 错误捕获

服务端路由的 Loader 和 Action 错误通过 `entry.server.tsx` 的 `handleError` 函数捕获：

```typescript
// app/entry.server.tsx:125-142
export function handleError(
	error: unknown,
	{ request }: LoaderFunctionArgs | ActionFunctionArgs,
): void {
	// 跳过已中止的请求（遵循 Remix 文档建议）
	if (request.signal.aborted) {
		return
	}

	if (error instanceof Error) {
		console.error(styleText('red', String(error.stack)))
	} else {
		console.error(error)
	}

	// 上报到 Sentry
	Sentry.captureException(error)
}
```

**捕获时机**：
- 当 Loader 抛出异常时
- 当 Action 抛出异常时
- React Router 自动调用此钩子

#### 3.1.4 服务器启动/关闭错误捕获

```typescript
// server/index.ts:236-248
closeWithGrace(async ({ err }) => {
	await new Promise((resolve, reject) => {
		server.close((e) => (e ? reject(e) : resolve('ok')))
	})
	if (err) {
		console.error(styleText('red', String(err)))
		console.error(styleText('red', String(err.stack)))
		if (SENTRY_ENABLED) {
			Sentry.captureException(err)
			await Sentry.flush(500)  // 确保在进程退出前发送完毕
		}
	}
})
```

---

### 3.2 客户端 (Browser 环境)

#### 3.2.1 初始化入口

客户端 Sentry 初始化位于 `entry.client.tsx`：

```typescript
// app/entry.client.tsx:5-7
if (ENV.MODE === 'production' && ENV.SENTRY_DSN) {
	void import('./utils/monitoring.client.tsx').then(({ init }) => init())
}
```

**关键特性**：
- 同样仅在生产环境初始化
- 使用 `window.ENV` 中的环境变量（服务端注入）
- 动态导入避免开发环境加载

#### 3.2.2 客户端配置详解

```typescript
// app/utils/monitoring.client.tsx
export function init() {
	Sentry.init({
		dsn: ENV.SENTRY_DSN,
		environment: ENV.MODE,
		
		// 前置钩子：过滤浏览器扩展错误
		beforeSend(event) {
			if (event.request?.url) {
				const url = new URL(event.request.url)
				if (
					url.protocol === 'chrome-extension:' ||
					url.protocol === 'moz-extension:'
				) {
					// 忽略浏览器扩展引发的错误
					return null
				}
			}
			return event
		},
		
		// 客户端专用集成
		integrations: [
			Sentry.replayIntegration(),           // 会话回放
			Sentry.browserProfilingIntegration(), // 浏览器性能分析
		],
		
		// 性能采样：100% 捕获
		tracesSampleRate: 1.0,
		
		// 会话回放采样策略
		replaysSessionSampleRate: 0.1,      // 10% 的正常会话
		replaysOnErrorSampleRate: 1.0,      // 100% 的错误会话
	})
}
```

#### 3.2.3 React Error Boundary 错误捕获

客户端使用自定义的 `GeneralErrorBoundary` 组件捕获 React 组件树中的错误：

```typescript
// app/components/error-boundary.tsx
export function GeneralErrorBoundary({
	defaultStatusHandler,
	statusHandlers,
	unexpectedErrorHandler,
}: {
	// ...
}) {
	const error = useRouteError()
	const params = useParams()
	const isResponse = isRouteErrorResponse(error)

	if (typeof document !== 'undefined') {
		console.error(error)
	}

	// 仅上报非响应类型的错误（如 404 等响应错误不上报）
	useEffect(() => {
		if (isResponse) return  // 跳过 HTTP 响应错误（如 404）

		captureException(error)  // 上报到 Sentry
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

**错误分类处理**：
| 错误类型 | 是否上报 Sentry | 处理方式 |
|---------|----------------|---------|
| 404 Not Found | ❌ 否 | 显示友好页面 |
| 403 Forbidden | ❌ 否 | 显示权限提示 |
| 500 Server Error | ✅ 是 | 上报 + 显示错误 |
| JS Runtime Error | ✅ 是 | 上报 + 显示错误 |

---

### 3.3 Edge 运行时支持分析

#### 3.3.1 当前状态

经过代码分析，当前 Epic Stack 的 Sentry 集成**主要面向 Node.js 环境**，未显式配置 Edge 运行时（如 Cloudflare Workers、Vercel Edge Functions）的特殊处理。

#### 3.3.2 潜在的 Edge 支持方式

若需要支持 Edge 运行时，需考虑以下调整：

```typescript
// 潜在的 Edge 环境检测
const isEdge = typeof EdgeRuntime !== 'undefined' || 
               typeof WebSocketPair !== 'undefined'

// Edge 专用初始化（假设）
if (isEdge) {
	Sentry.init({
		dsn: process.env.SENTRY_DSN,
		environment: process.env.NODE_ENV,
		// Edge 环境不支持某些 Node.js 集成
		integrations: [
			// 仅使用兼容 Edge 的集成
		],
	})
}
```

#### 3.3.3 依赖兼容性检查

| 依赖包 | Node.js 兼容 | Edge 兼容 |
|-------|-------------|----------|
| @sentry/react-router | ✅ | ✅ (部分功能) |
| @sentry/profiling-node | ✅ | ❌ |
| @prisma/instrumentation | ✅ | ❌ |
| Sentry.httpIntegration | ✅ | ✅ |

---

## 四、上下文传递机制

### 4.1 环境变量传递链

#### 4.1.1 服务端环境变量

```typescript
// app/utils/env.server.ts
const schema = z.object({
	// ...
	// SENTRY_DSN 是可选的，实际使用时需移除 .optional()
	SENTRY_DSN: z.string().optional(),
	// ...
})

// 仅暴露给客户端的环境变量
export function getEnv() {
	return {
		MODE: process.env.NODE_ENV,
		SENTRY_DSN: process.env.SENTRY_DSN,  // 暴露给客户端
		ALLOW_INDEXING: process.env.ALLOW_INDEXING,
	}
}
```

#### 4.1.2 构建时环境变量

```typescript
// vite.config.ts:87-103
const sentryConfig: SentryReactRouterBuildOptions = {
	authToken: process.env.SENTRY_AUTH_TOKEN,  // CI/CD 注入
	org: process.env.SENTRY_ORG,
	project: process.env.SENTRY_PROJECT,

	unstable_sentryVitePluginOptions: {
		release: {
			name: process.env.COMMIT_SHA,  // Git Commit SHA 作为 Release 名称
			setCommits: {
				auto: true,  // 自动关联提交信息
			},
		},
		sourcemaps: {
			filesToDeleteAfterUpload: ['./build/**/*.map'],  // 上传后删除 Source Map
		},
	},
}
```

#### 4.1.3 客户端环境变量注入

```typescript
// entry.server.tsx:22-23
init()                     // 初始化环境变量校验
global.ENV = getEnv()      // 设置全局 ENV

// 在 HTML 模板中注入（通过 React Router 内部机制）
// 客户端通过 window.ENV 访问
```

### 4.2 Release 管理与 Source Map

#### 4.2.1 构建时集成

```typescript
// react-router.config.ts
import { sentryOnBuildEnd } from '@sentry/react-router'

export default {
	// ...
	buildEnd: async ({ viteConfig, reactRouterConfig, buildManifest }) => {
		if (MODE === 'production' && process.env.SENTRY_AUTH_TOKEN) {
			await sentryOnBuildEnd({
				viteConfig,
				reactRouterConfig,
				buildManifest,
			})
		}
	},
}
```

#### 4.2.2 Vite 插件配置

```typescript
// vite.config.ts:70-72
mode === 'production' && process.env.SENTRY_AUTH_TOKEN
	? sentryReactRouter(sentryConfig, config)
	: null,
```

**构建流程**：
```
1. Vite 编译代码 → 生成 Source Map
2. Sentry 插件自动上传 Source Map 到 Sentry
3. 上传完成后删除本地 .map 文件（安全考虑）
4. 创建 Release 并关联 Git Commits
```

### 4.3 请求上下文追踪

#### 4.3.1 服务端请求上下文

Sentry 的 React Router SDK 自动追踪以下请求信息：

- HTTP 方法 (GET/POST/PUT/DELETE)
- 请求 URL
- 请求头 (Headers)
- 用户代理 (User-Agent)
- IP 地址（通过 `X-Forwarded-For` 或 `fly-client-ip`）

#### 4.3.2 Fly.io 部署上下文

```typescript
// entry.server.tsx:33-36
responseHeaders.set('fly-region', process.env.FLY_REGION ?? 'unknown')
responseHeaders.set('fly-app', process.env.FLY_APP_NAME ?? 'unknown')
responseHeaders.set('fly-primary-instance', primaryInstance)
responseHeaders.set('fly-instance', currentInstance)
```

**这些信息会被 Sentry 捕获，用于**：
- 识别错误发生的地理区域
- 区分主从实例
- 定位特定部署的问题

---

## 五、跨环境错误关联机制

### 5.1 统一的 DSN 和 Environment

服务端和客户端使用**相同的 Sentry DSN**：

| 环境 | 配置来源 | Environment 值 |
|-----|---------|---------------|
| 服务端 | `process.env.SENTRY_DSN` | `process.env.NODE_ENV` |
| 客户端 | `window.ENV.SENTRY_DSN` | `window.ENV.MODE` |

**关联方式**：通过相同的 DSN 和 Environment，Sentry 自动将服务端和客户端的错误聚合到同一个项目中。

### 5.2 Release 版本关联

```typescript
// 构建时使用 COMMIT_SHA 作为 Release 名称
release: {
	name: process.env.COMMIT_SHA,
	setCommits: {
		auto: true,
	},
}
```

**关联机制**：
1. CI/CD 构建时注入 `COMMIT_SHA` 环境变量
2. 服务端和客户端代码都包含相同的 Release 标识
3. Sentry 通过 Release ID 关联服务端和客户端的错误和性能数据

### 5.3 Trace ID 传播（潜在机制）

虽然代码中未显式配置 Trace ID 传播，但 Sentry 的 React Router SDK 可能自动支持：

**服务端 → 客户端的 Trace 传播**：
```
1. 服务端处理请求时生成 trace_id
2. 通过 HTML 模板或响应头传递给客户端
3. 客户端初始化时继承 trace_id
4. 客户端错误和性能事件关联到同一 trace
```

**HTTP 请求的 Trace 传播**：
```
客户端发起请求 → Sentry 自动添加 sentry-trace 头
服务端接收请求 → Sentry 提取 trace_id 并继续追踪
形成完整的分布式追踪链
```

---

## 六、采样策略与性能优化

### 6.1 服务端采样策略

```typescript
// server/utils/monitoring.ts:26-32
tracesSampler(samplingContext) {
	// 完全忽略健康检查
	if (samplingContext.request?.url?.includes('/resources/healthcheck')) {
		return 0
	}
	// 生产环境 100%，开发环境 0%
	return process.env.NODE_ENV === 'production' ? 1 : 0
}
```

### 6.2 客户端采样策略

```typescript
// app/utils/monitoring.client.tsx
tracesSampleRate: 1.0,           // 性能追踪 100% 采样
replaysSessionSampleRate: 0.1,   // 正常会话 10% 回放
replaysOnErrorSampleRate: 1.0,   // 错误会话 100% 回放
```

### 6.3 错误过滤机制

| 过滤点 | 过滤内容 | 实现位置 |
|-------|---------|---------|
| 服务端 | 健康检查、静态资源 | `denyUrls` |
| 服务端 | `x-healthcheck` 头 | `beforeSendTransaction` |
| 客户端 | 浏览器扩展错误 | `beforeSend` |
| Error Boundary | HTTP 响应错误 (404/403) | `isResponse` 检查 |
| `handleError` | 已中止的请求 | `request.signal.aborted` |

---

## 七、安全考虑

### 7.1 Source Map 安全

```typescript
// vite.config.ts:99-101
sourcemaps: {
	filesToDeleteAfterUpload: ['./build/**/*.map'],
}
```

**安全措施**：
- 构建后删除本地 Source Map 文件
- Source Map 仅上传到 Sentry，不部署到生产环境
- 防止用户通过浏览器开发者工具查看源码

### 7.2 环境变量安全

```typescript
// app/utils/env.server.ts
// SENTRY_AUTH_TOKEN、SENTRY_ORG、SENTRY_PROJECT 
// 仅在构建时使用，不暴露给客户端或运行时

// 仅以下变量暴露给客户端：
export function getEnv() {
	return {
		MODE: process.env.NODE_ENV,
		SENTRY_DSN: process.env.SENTRY_DSN,  // 可公开的 DSN
		ALLOW_INDEXING: process.env.ALLOW_INDEXING,
	}
}
```

### 7.3 CSP 策略配置

```typescript
// entry.server.tsx:74-78
'connect-src': [
	MODE === 'development' ? 'ws:' : undefined,
	process.env.SENTRY_DSN ? '*.sentry.io' : undefined,  // 允许 Sentry
	"'self'",
],
```

---

## 八、配置清单

### 8.1 必需的环境变量

| 变量名 | 用途 | 必需时机 |
|-------|-----|---------|
| `SENTRY_DSN` | 错误上报地址 | 运行时（服务端+客户端） |
| `SENTRY_AUTH_TOKEN` | Source Map 上传权限 | 构建时（CI/CD） |
| `SENTRY_ORG` | Sentry 组织标识 | 构建时 |
| `SENTRY_PROJECT` | Sentry 项目标识 | 构建时 |
| `COMMIT_SHA` | Release 版本标识 | 构建时 |
| `NODE_ENV` | 环境标识 | 运行时+构建时 |

### 8.2 部署架构示例

```
┌──────────────────────────────────────────────────────────────┐
│                        GitHub Actions                          │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  构建阶段                                                 │  │
│  │  - npm run build                                         │  │
│  │  - Sentry Vite 插件自动：                                │  │
│  │    • 上传 Source Maps                                    │  │
│  │    • 创建 Release (COMMIT_SHA)                           │  │
│  │    • 关联 Git Commits                                    │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────┬───────────────────────────────────┘
                           │ 部署
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                        Fly.io (生产环境)                        │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  运行时环境                                               │  │
│  │  - SENTRY_DSN (通过 fly secrets 注入)                    │  │
│  │  - NODE_ENV=production                                   │  │
│  │  - FLY_REGION, FLY_APP_NAME 等部署信息                   │  │
│  │                                                          │  │
│  │  服务端错误捕获：                                         │  │
│  │  - Loader/Action 错误 → handleError → Sentry            │  │
│  │  - 服务器启动/关闭错误 → closeWithGrace → Sentry         │  │
│  │                                                          │  │
│  │  客户端错误捕获：                                         │  │
│  │  - React Error Boundary → captureException → Sentry      │  │
│  │  - 全局 uncaught exception → Sentry 自动捕获              │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## 九、总结

### 9.1 核心设计原则

1. **环境隔离**：仅生产环境启用 Sentry，避免开发环境污染
2. **动态初始化**：使用 `import()` 延迟加载，减少开发环境依赖
3. **智能过滤**：多层过滤机制（健康检查、静态资源、浏览器扩展）
4. **安全优先**：Source Map 不上线、敏感变量不暴露
5. **统一关联**：通过 DSN + Release 实现服务端客户端错误关联

### 9.2 各环境捕获方式对比

| 维度 | 服务端 (Node.js) | 客户端 (Browser) | Edge 运行时 |
|-----|-----------------|-----------------|------------|
| **初始化入口** | `server/index.ts` | `entry.client.tsx` | 未配置 |
| **配置文件** | `monitoring.ts` | `monitoring.client.tsx` | 无 |
| **核心集成** | Prisma, HTTP, Node Profiling | Replay, Browser Profiling | - |
| **错误捕获** | `handleError` 钩子 | Error Boundary + 全局 | - |
| **采样策略** | 动态采样器 | 固定采样率 | - |

### 9.3 上下文传递链路

```
构建时 (CI/CD)
    │
    ▼
┌─────────────────────┐
│  SENTRY_AUTH_TOKEN  │ ──► Source Map 上传
│  SENTRY_ORG         │ ──► Release 创建
│  SENTRY_PROJECT     │ ──► 项目关联
│  COMMIT_SHA         │ ──► Release 命名
└─────────────────────┘
    │
    ▼
运行时 (服务端)
    │
    ▼
┌─────────────────────┐
│  SENTRY_DSN         │ ──► 错误上报
│  NODE_ENV           │ ──► Environment 标签
│  FLY_REGION         │ ──► 部署上下文
│  FLY_APP_NAME       │ ──► 应用标识
└─────────────────────┘
    │
    ▼ (通过 window.ENV 注入)
运行时 (客户端)
    │
    ▼
┌─────────────────────┐
│  ENV.SENTRY_DSN     │ ──► 错误上报
│  ENV.MODE           │ ──► Environment 标签
│  Release (编译时注入)│ ──► 版本关联
└─────────────────────┘
```

### 9.4 建议与优化方向

1. **Edge 运行时支持**：若需要部署到 Cloudflare Workers 等 Edge 环境，需：
   - 移除不兼容的集成（`@prisma/instrumentation`, `@sentry/profiling-node`）
   - 添加 Edge 专用的初始化逻辑

2. **用户上下文增强**：当前代码未显式设置用户信息，建议添加：
   ```typescript
   Sentry.setUser({
     id: user.id,
     email: user.email,
     username: user.username,
   })
   ```

3. **自定义标签**：可添加业务相关的标签便于排查：
   ```typescript
   Sentry.setTag('feature_flag_variant', variant)
   Sentry.setTag('user_segment', segment)
   ```

4. **分布式追踪完善**：确保服务端和客户端的 Trace ID 正确传播，形成完整的调用链。

---

## 十、参考文件

| 文件路径 | 用途 |
|---------|------|
| `server/utils/monitoring.ts` | 服务端 Sentry 配置 |
| `app/utils/monitoring.client.tsx` | 客户端 Sentry 配置 |
| `server/index.ts` | 服务端初始化入口 |
| `app/entry.client.tsx` | 客户端初始化入口 |
| `app/entry.server.tsx` | 服务端请求处理 + `handleError` |
| `app/components/error-boundary.tsx` | React 错误边界 |
| `vite.config.ts` | 构建时 Sentry 插件配置 |
| `react-router.config.ts` | `buildEnd` 钩子配置 |
| `app/utils/env.server.ts` | 环境变量 Schema |
| `docs/decisions/015-monitoring.md` | 架构决策记录 |
| `docs/monitoring.md` | 部署配置指南 |
