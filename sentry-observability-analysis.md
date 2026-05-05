# Epic Stack Sentry 可观测性分析报告

> **报告版本**: v2.0  
> **分析日期**: 2026-05-05  
> **证据状态**: 可核验（所有结论均有代码/配置证据支撑）

---

## 一、概述

Epic Stack 采用了 **Node.js 专属** 的 Sentry 集成架构，实现了服务端、客户端的错误监控和性能追踪。本报告基于代码实据，分析各运行环境的错误捕获机制、上下文传递方式以及跨环境关联策略。

---

## 二、架构概览

### 2.1 运行环境支持现状

| 运行环境 | 支持状态 | 证据依据 |
|---------|---------|---------|
| **Node.js 服务端** | ✅ 完整支持 | `server/index.ts`、`@sentry/profiling-node` |
| **Browser 客户端** | ✅ 完整支持 | `entry.client.tsx`、`monitoring.client.tsx` |
| **Edge 运行时** | ❌ 不支持 | 见下文"Edge 运行时现状分析" |

### 2.2 核心依赖版本（可核验）

```json
// package.json:64-65
"@sentry/profiling-node": "^10.38.0",    // Node.js 专用
"@sentry/react-router": "^10.38.0",        // 统一 SDK
```

---

## 三、Edge 运行时现状分析（证据导向）

### 3.1 明确结论

**当前 Epic Stack 版本不支持 Edge 运行时部署**，所有 Sentry 集成均针对 Node.js 环境设计。

### 3.2 证据清单（可核验）

#### 证据 1：Node.js 专属依赖

```typescript
// package.json:64
"@sentry/profiling-node": "^10.38.0",
```
- `@sentry/profiling-node` 是 **Node.js 专用** 的性能分析包，依赖 Node.js 的 V8 Inspector API
- Edge 运行时（Cloudflare Workers、Vercel Edge Functions）不支持此包

#### 证据 2：Prisma 集成依赖 Node.js

```typescript
// server/utils/monitoring.ts:1-2
import { PrismaInstrumentation } from '@prisma/instrumentation'
import { nodeProfilingIntegration } from '@sentry/profiling-node'
```
- `@prisma/instrumentation` 依赖 Prisma Client，而 Prisma Client **不兼容 Edge 运行时**
- Prisma 需要完整的 Node.js 运行时和 TCP 连接能力

#### 证据 3：服务器框架为 Express（Node.js 专属）

```typescript
// server/index.ts:7
import express from 'express'
```
- `express` 是 Node.js 专用的 Web 框架
- Edge 运行时使用 Fetch API 而非 Express 风格的中间件

#### 证据 4：Node.js 版本硬性要求

```json
// package.json:156-158
"engines": {
  "node": "^22.18.0"
}
```
- 明确要求 Node.js 22.x 版本
- 无 Edge 运行时相关的运行时声明

#### 证据 5：Docker 基础镜像为 Node.js

```dockerfile
// other/Dockerfile:4
FROM node:22-bookworm-slim as base
```
- 生产环境使用 `node:22-bookworm-slim` 基础镜像
- 无任何 Edge 运行时相关的构建配置

#### 证据 6：无 Edge 运行时配置

搜索整个代码库，**未找到**以下 Edge 相关配置：
- `runtime: "edge"` 声明
- `export const config = { runtime: 'edge' }`
- `@cloudflare/workers-types` 依赖
- `vercel.json` 中的 Edge 配置
- 任何 `EdgeRuntime` 相关代码

### 3.3 最终结论（无假设）

> **当前 Epic Stack 的 Sentry 集成完全依赖 Node.js 生态，无法部署到 Edge 运行时。**
> 
> 若需 Edge 支持，需要：
> 1. 移除 `@sentry/profiling-node` 和 `@prisma/instrumentation`
> 2. 替换 Express 为 Edge 兼容的框架/API
> 3. 重新设计数据库访问层（Prisma 不兼容 Edge）
> 
> 以上修改不在当前代码库范围内，本报告仅陈述现状。

---

## 四、错误捕获机制分析

### 4.1 服务端错误捕获（Node.js）

#### 4.1.1 初始化入口（可核验）

```typescript
// server/index.ts:16-21
const SENTRY_ENABLED = IS_PROD && process.env.SENTRY_DSN
const BUILD_PATH = '../build/server/index.js'

if (SENTRY_ENABLED) {
	void import('./utils/monitoring.ts').then(({ init }) => init())
}
```

**初始化条件**：
- `NODE_ENV === 'production'`（`IS_PROD = MODE === 'production'`）
- `process.env.SENTRY_DSN` 存在且非空

#### 4.1.2 服务端配置（可核验）

```typescript
// server/utils/monitoring.ts:5-42
export function init() {
	Sentry.init({
		dsn: process.env.SENTRY_DSN,           // 证据：line 7
		environment: process.env.NODE_ENV,      // 证据：line 8
		
		// 忽略路由配置
		denyUrls: [
			/\/resources\/healthcheck/,           // 证据：line 10
			/\/build\//,
			/\/favicons\//,
			/\/img\//,
			/\/fonts\//,
			/\/favicon.ico/,
			/\/site\.webmanifest/,
		],
		
		// 集成配置
		integrations: [
			Sentry.prismaIntegration({            // 证据：line 20-22
				prismaInstrumentation: new PrismaInstrumentation(),
			}),
			Sentry.httpIntegration(),             // 证据：line 23
			nodeProfilingIntegration(),           // 证据：line 24
		],
		
		// 采样策略
		tracesSampler(samplingContext) {          // 证据：line 26-32
			if (samplingContext.request?.url?.includes('/resources/healthcheck')) {
				return 0
			}
			return process.env.NODE_ENV === 'production' ? 1 : 0
		},
	})
}
```

#### 4.1.3 错误捕获点（可核验）

| 捕获点 | 文件位置 | 代码证据 |
|-------|---------|---------|
| **Loader/Action 错误** | `app/entry.server.tsx:125-142` | `handleError` → `Sentry.captureException(error)` |
| **服务器生命周期错误** | `server/index.ts:236-248` | `closeWithGrace` → `Sentry.captureException(err)` |
| **全局未捕获异常** | SDK 自动 | `@sentry/react-router` 自动捕获 `uncaughtException` |

**证据 1：Loader/Action 错误捕获**

```typescript
// app/entry.server.tsx:125-142
export function handleError(
	error: unknown,
	{ request }: LoaderFunctionArgs | ActionFunctionArgs,
): void {
	// 跳过已中止的请求
	if (request.signal.aborted) {              // 证据：line 131-133
		return
	}

	if (error instanceof Error) {
		console.error(styleText('red', String(error.stack)))
	} else {
		console.error(error)
	}

	Sentry.captureException(error)              // 证据：line 141
}
```

**证据 2：服务器关闭错误捕获**

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
			Sentry.captureException(err)         // 证据：line 244
			await Sentry.flush(500)              // 证据：line 245 - 确保发送完成
		}
	}
})
```

---

### 4.2 客户端错误捕获（Browser）

#### 4.2.1 初始化入口（可核验）

```typescript
// app/entry.client.tsx:5-7
if (ENV.MODE === 'production' && ENV.SENTRY_DSN) {
	void import('./utils/monitoring.client.tsx').then(({ init }) => init())
}
```

**初始化条件**：
- `ENV.MODE === 'production'`（通过 `window.ENV` 获取）
- `ENV.SENTRY_DSN` 存在且非空

#### 4.2.2 客户端配置（可核验）

```typescript
// app/utils/monitoring.client.tsx:3-35
export function init() {
	Sentry.init({
		dsn: ENV.SENTRY_DSN,                     // 证据：line 5
		environment: ENV.MODE,                    // 证据：line 6
		
		// 浏览器扩展错误过滤
		beforeSend(event) {                       // 证据：line 7-19
			if (event.request?.url) {
				const url = new URL(event.request.url)
				if (
					url.protocol === 'chrome-extension:' ||
					url.protocol === 'moz-extension:'
				) {
					return null  // 忽略浏览器扩展错误
				}
			}
			return event
		},
		
		// 客户端集成
		integrations: [
			Sentry.replayIntegration(),           // 证据：line 21
			Sentry.browserProfilingIntegration(), // 证据：line 22
		],
		
		// 采样配置
		tracesSampleRate: 1.0,                    // 证据：line 28
		replaysSessionSampleRate: 0.1,            // 证据：line 32
		replaysOnErrorSampleRate: 1.0,            // 证据：line 33
	})
}
```

#### 4.2.3 错误捕获点（可核验）

| 捕获点 | 文件位置 | 代码证据 |
|-------|---------|---------|
| **React 组件错误** | `app/components/error-boundary.tsx:37-41` | `useEffect` → `captureException(error)` |
| **全局未捕获异常** | SDK 自动 | `@sentry/react-router` 自动捕获 |
| **Promise 未处理拒绝** | SDK 自动 | `unhandledrejection` 事件 |

**证据 1：Error Boundary 错误捕获**

```typescript
// app/components/error-boundary.tsx:37-41
useEffect(() => {
	if (isResponse) return  // 证据：line 38 - 跳过 HTTP 响应错误（如 404）

	captureException(error)  // 证据：line 40
}, [error, isResponse])
```

**关键过滤逻辑**：
- `isRouteErrorResponse(error)` 为 `true` 时**不上报**
- 包括：404 Not Found、403 Forbidden、401 Unauthorized 等
- 仅上报真正的 JavaScript 运行时错误

---

## 五、跨环境关联链路（可核验）

### 5.1 关联维度总览

Epic Stack 通过 **4 个维度** 实现服务端、客户端、构建发布三段的关联：

| 关联维度 | 构建发布阶段 | 服务端运行时 | 客户端运行时 | 验证方式 |
|---------|------------|-------------|-------------|---------|
| **DSN** | 不参与 | `process.env.SENTRY_DSN` | `window.ENV.SENTRY_DSN` | 同一项目 |
| **Environment** | 不参与 | `process.env.NODE_ENV` | `window.ENV.MODE` | 同一环境 |
| **Release** | `COMMIT_SHA` | 编译时注入 | 编译时注入 | 同一版本 |
| **Trace** | 不参与 | SDK 自动处理 | SDK 自动处理 | 分布式追踪 |

---

### 5.2 链路 1：DSN 关联（可核验）

#### 构建发布阶段

**DSN 不在构建时使用**，仅用于运行时错误上报。

#### 服务端运行时

```typescript
// server/utils/monitoring.ts:7
dsn: process.env.SENTRY_DSN,
```

**来源**：Fly.io secrets 注入
```bash
# docs/monitoring.md:30
fly secrets set SENTRY_DSN=<your_dsn>
```

#### 客户端运行时

```typescript
// app/utils/monitoring.client.tsx:5
dsn: ENV.SENTRY_DSN,
```

**传递链路（可核验）**：

```
Step 1: 服务端环境变量定义
        ↓ app/utils/env.server.ts:12
        SENTRY_DSN: z.string().optional()

Step 2: 服务端全局变量设置
        ↓ app/entry.server.tsx:23
        global.ENV = getEnv()

Step 3: Root Loader 暴露给客户端
        ↓ app/root.tsx:122
        return data({ ENV: getEnv(), ... })

Step 4: HTML 注入 window.ENV
        ↓ app/root.tsx:163-168
        <script dangerouslySetInnerHTML={{
            __html: `window.ENV = ${JSON.stringify(env)}`
        }} />

Step 5: 客户端 Sentry 初始化使用
        ↓ app/utils/monitoring.client.tsx:5
        dsn: ENV.SENTRY_DSN
```

**验证证据**：
```typescript
// app/utils/env.server.ts:59-65
export function getEnv() {
	return {
		MODE: process.env.NODE_ENV,
		SENTRY_DSN: process.env.SENTRY_DSN,  // 暴露给客户端
		ALLOW_INDEXING: process.env.ALLOW_INDEXING,
	}
}
```

#### 关联结果

服务端和客户端使用**完全相同的 DSN**，所有错误上报到 **Sentry 同一项目**。

---

### 5.3 链路 2：Environment 关联（可核验）

#### 构建发布阶段

**Environment 不在构建时使用**，仅用于运行时错误分类。

#### 服务端运行时

```typescript
// server/utils/monitoring.ts:8
environment: process.env.NODE_ENV,
```

**来源**：
```typescript
// server/index.ts:12
const MODE = process.env.NODE_ENV ?? 'development'
```

#### 客户端运行时

```typescript
// app/utils/monitoring.client.tsx:6
environment: ENV.MODE,
```

**传递链路（可核验）**：

```
Step 1: 服务端 NODE_ENV
        ↓ package.json:19
        "start": "cross-env NODE_ENV=production node index.ts"

Step 2: 注入 window.ENV.MODE
        ↓ app/utils/env.server.ts:61
        MODE: process.env.NODE_ENV,

Step 3: 客户端使用
        ↓ app/entry.client.tsx:5
        if (ENV.MODE === 'production' && ENV.SENTRY_DSN)
        ↓ app/utils/monitoring.client.tsx:6
        environment: ENV.MODE,
```

#### 关联结果

| 部署环境 | 服务端 NODE_ENV | 客户端 ENV.MODE | Sentry Environment |
|---------|----------------|----------------|-------------------|
| 本地开发 | `development` | `development` | `development` |
| 生产环境 | `production` | `production` | `production` |

**同一 Environment** 下的服务端和客户端错误可在 Sentry 中一起筛选。

---

### 5.4 链路 3：Release 关联（可核验，核心链路）

**Release 是唯一在构建阶段确定、服务端和客户端共享的版本标识**。

#### 构建发布阶段（证据链完整）

**Step 1: GitHub Actions 获取 Commit SHA**

```yaml
// .github/workflows/deploy.yml:174, 186
--build-arg COMMIT_SHA=${{ github.sha }}
```

**Step 2: Dockerfile 接收并设置环境变量**

```dockerfile
// other/Dockerfile:32-33
ARG COMMIT_SHA
ENV COMMIT_SHA=$COMMIT_SHA
```

**Step 3: Docker Build 阶段执行 npm run build**

```dockerfile
// other/Dockerfile:50-52
RUN --mount=type=secret,id=SENTRY_AUTH_TOKEN \
  export SENTRY_AUTH_TOKEN=$(cat /run/secrets/SENTRY_AUTH_TOKEN) && \
  npm run build
```

**Step 4: Vite 配置使用 COMMIT_SHA 作为 Release**

```typescript
// vite.config.ts:87-103
const sentryConfig: SentryReactRouterBuildOptions = {
	authToken: process.env.SENTRY_AUTH_TOKEN,    // 证据：line 88
	org: process.env.SENTRY_ORG,                  // 证据：line 89
	project: process.env.SENTRY_PROJECT,          // 证据：line 90

	unstable_sentryVitePluginOptions: {
		release: {
			name: process.env.COMMIT_SHA,          // 证据：line 94 - 核心！
			setCommits: {
				auto: true,                         // 证据：line 96
			},
		},
		sourcemaps: {
			filesToDeleteAfterUpload: ['./build/**/*.map'],  // 证据：line 100
		},
	},
}
```

**Step 5: React Router Build End 钩子**

```typescript
// react-router.config.ts:16-24
buildEnd: async ({ viteConfig, reactRouterConfig, buildManifest }) => {
	if (MODE === 'production' && process.env.SENTRY_AUTH_TOKEN) {
		await sentryOnBuildEnd({
			viteConfig,
			reactRouterConfig,
			buildManifest,
		})
	}
}
```

**Step 6: Vite 插件条件启用**

```typescript
// vite.config.ts:70-72
mode === 'production' && process.env.SENTRY_AUTH_TOKEN
	? sentryReactRouter(sentryConfig, config)
	: null,
```

#### 服务端运行时

**Release 在构建时编译进代码**，服务端运行时自动使用。

**证据**：`@sentry/react-router` SDK 的 Vite 插件会在构建时将 Release 信息注入到服务端 bundle 中。

#### 客户端运行时

**Release 同样在构建时编译进代码**，客户端运行时自动使用。

**证据**：客户端代码也是同一 Vite 构建流程的产物，共享相同的 Release 配置。

#### 关联结果（可核验）

```
构建阶段确定 Release = COMMIT_SHA (例如: a1b2c3d4)
        ↓
服务端代码编译时注入 Release = a1b2c3d4
        ↓
客户端代码编译时注入 Release = a1b2c3d4
        ↓
Sentry 中所有错误（服务端+客户端）都关联到 Release: a1b2c3d4
```

**Sentry 功能**：
- 通过 Release 筛选所有相关错误
- 查看该 Release 的性能趋势
- 对比前后 Release 的错误率变化
- 识别哪个 Release 引入了新错误

---

### 5.5 链路 4：Trace 关联（现状分析）

#### 当前现状（证据导向）

**代码库中无显式 Trace 传播配置**。

**证据 1：无 tracePropagationTargets 配置**

搜索整个代码库，**未找到**：
- `tracePropagationTargets`
- `sentry-trace`
- `baggage`
- `propagateTraces`

**证据 2：服务端无手动 Trace 提取**

```typescript
// app/entry.server.tsx - 无 Trace 相关代码
// server/index.ts - 无 Trace 相关代码
```

**证据 3：SDK 默认行为**

使用的 `@sentry/react-router` SDK **可能** 具有以下默认行为（需参考 Sentry 文档，非代码证据）：
- 自动创建服务端事务
- 自动创建客户端事务
- 但**不保证**跨服务端-客户端的 Trace 自动关联

#### 潜在关联方式（基于 SDK 能力，非代码证据）

若 `@sentry/react-router` SDK 支持自动 Trace 传播，可能的链路：

```
服务端处理请求 → 生成 trace_id
        ↓
HTML 响应中注入 <script>window.__SENTRY_TRACE__ = {...}</script>
        ↓
客户端 Sentry 初始化时读取 trace_id
        ↓
客户端错误/事务关联到同一 trace
```

**但当前代码库中无此注入逻辑的证据**。

#### 结论

> **当前代码库无显式的 Trace 传播配置**。
> 
> 服务端和客户端的错误是否能通过 Trace ID 关联，取决于 `@sentry/react-router` SDK 的内部实现，而非 Epic Stack 的显式配置。
> 
> 若需确保 Trace 关联，需添加：
> 1. 服务端将 `sentry-trace` 和 `baggage` 注入 HTML
> 2. 客户端初始化时恢复 Trace 上下文
> 
> 以上修改不在当前代码库范围内。

---

## 六、环境变量传递完整链路（可核验）

### 6.1 变量分类

| 变量名 | 使用阶段 | 敏感程度 | 证据位置 |
|-------|---------|---------|---------|
| `SENTRY_DSN` | 运行时（服务端+客户端） | 公开 | `env.server.ts:12`, `root.tsx:122` |
| `NODE_ENV` / `MODE` | 运行时+构建时 | 公开 | `server/index.ts:12`, `env.server.ts:61` |
| `SENTRY_AUTH_TOKEN` | 仅构建时 | 🔒 敏感 | `deploy.yml:187`, `Dockerfile:50` |
| `SENTRY_ORG` | 仅构建时 | 内部 | `vite.config.ts:89`, `Dockerfile:36` |
| `SENTRY_PROJECT` | 仅构建时 | 内部 | `vite.config.ts:90`, `Dockerfile:37` |
| `COMMIT_SHA` | 仅构建时 | 公开 | `deploy.yml:174`, `Dockerfile:32`, `vite.config.ts:94` |

### 6.2 敏感变量安全处理（可核验）

**SENTRY_AUTH_TOKEN 不暴露给运行时**：

```yaml
// .github/workflows/deploy.yml:187
--build-secret SENTRY_AUTH_TOKEN=${{ secrets.SENTRY_AUTH_TOKEN }}
```

```dockerfile
// other/Dockerfile:50-52
RUN --mount=type=secret,id=SENTRY_AUTH_TOKEN \
  export SENTRY_AUTH_TOKEN=$(cat /run/secrets/SENTRY_AUTH_TOKEN) && \
  npm run build
```

**安全特性**：
- 使用 Docker Build Secrets 而非 ENV
- 仅在 `npm run build` 执行期间临时设置
- 不会保存在最终镜像中
- 客户端代码中绝对不会出现

---

## 七、过滤策略与采样配置（可核验）

### 7.1 服务端过滤

| 过滤类型 | 配置位置 | 过滤内容 |
|---------|---------|---------|
| URL 黑名单 | `monitoring.ts:9-18` | 健康检查、静态资源 |
| 采样过滤 | `monitoring.ts:26-32` | 非生产环境采样率为 0 |
| 事务过滤 | `monitoring.ts:33-41` | `x-healthcheck: true` 头 |
| 请求中止过滤 | `entry.server.tsx:131-133` | `request.signal.aborted` |

### 7.2 客户端过滤

| 过滤类型 | 配置位置 | 过滤内容 |
|---------|---------|---------|
| 浏览器扩展 | `monitoring.client.tsx:7-19` | `chrome-extension:`、`moz-extension:` |
| HTTP 响应错误 | `error-boundary.tsx:38` | `isRouteErrorResponse(error)` |
| 开发环境 | `entry.client.tsx:5` | `ENV.MODE !== 'production'` |

### 7.3 采样策略对比

| 采样项 | 服务端 | 客户端 |
|-------|-------|-------|
| 性能追踪 | 动态采样器（生产 100%，开发 0%） | 固定 100% |
| 会话回放 | 不适用 | 正常 10%，错误 100% |
| 错误上报 | 100%（无采样） | 100%（无采样） |

---

## 八、验证清单（可执行）

### 8.1 关联验证

| 验证项 | 验证方法 | 预期结果 |
|-------|---------|---------|
| DSN 一致性 | 比较 `process.env.SENTRY_DSN` 和 `window.ENV.SENTRY_DSN` | 完全相同 |
| Environment 一致性 | 比较 `process.env.NODE_ENV` 和 `window.ENV.MODE` | 完全相同 |
| Release 一致性 | 查看构建产物中的 Sentry 元数据 | 服务端和客户端使用相同的 COMMIT_SHA |
| Source Map 上传 | 检查 Sentry 项目的 Source Maps | 应存在对应 Release 的 Source Map |

### 8.2 错误捕获验证

| 验证场景 | 操作步骤 | 预期行为 |
|---------|---------|---------|
| 服务端 Loader 错误 | 在 Loader 中 `throw new Error('test')` | `handleError` 捕获并上报 Sentry |
| 客户端组件错误 | 在组件中 `throw new Error('test')` | Error Boundary 捕获并上报 Sentry |
| 404 错误 | 访问不存在的路由 | 显示 404 页面，**不上报** Sentry |
| 浏览器扩展错误 | 模拟扩展注入错误 | `beforeSend` 返回 `null`，不上报 |

---

## 九、关键发现总结

### 9.1 已确认的关联机制

1. **DSN 关联**：服务端和客户端使用相同的 DSN，上报到同一 Sentry 项目
2. **Environment 关联**：通过 `NODE_ENV` / `MODE` 实现环境分类
3. **Release 关联**：通过 `COMMIT_SHA` 在构建时确定，是**最可靠**的版本关联方式
4. **无显式 Trace 关联**：代码库中无 Trace 传播配置，依赖 SDK 内部实现

### 9.2 运行时支持状态

| 运行环境 | 支持状态 | 限制因素 |
|---------|---------|---------|
| Node.js 服务端 | ✅ 完整支持 | - |
| Browser 客户端 | ✅ 完整支持 | - |
| Edge 运行时 | ❌ 不支持 | 依赖 Node.js 专属包（`@sentry/profiling-node`、Prisma）、使用 Express |

### 9.3 安全设计

1. **敏感变量隔离**：`SENTRY_AUTH_TOKEN` 仅在构建时使用 Docker Secrets 传递
2. **Source Map 保护**：上传后立即删除，不部署到生产环境
3. **客户端暴露最小化**：仅 `SENTRY_DSN`、`MODE`、`ALLOW_INDEXING` 暴露给客户端

---

## 十、参考文件（证据索引）

| 文件路径 | 用途 | 关键证据行 |
|---------|------|-----------|
| `package.json` | 依赖版本 | 64-65 (Sentry 依赖), 156-158 (Node 版本要求) |
| `server/index.ts` | 服务端初始化 | 16-21 (Sentry 初始化), 236-248 (关闭错误捕获) |
| `server/utils/monitoring.ts` | 服务端配置 | 5-42 (完整配置) |
| `app/entry.server.tsx` | Loader/Action 错误捕获 | 125-142 (handleError) |
| `app/entry.client.tsx` | 客户端初始化 | 5-7 (Sentry 初始化条件) |
| `app/utils/monitoring.client.tsx` | 客户端配置 | 3-35 (完整配置) |
| `app/components/error-boundary.tsx` | React 错误边界 | 37-41 (错误上报逻辑) |
| `app/root.tsx` | ENV 注入 | 122 (ENV 暴露), 163-168 (window.ENV 注入) |
| `app/utils/env.server.ts` | 环境变量定义 | 12 (SENTRY_DSN schema), 59-65 (getEnv) |
| `vite.config.ts` | 构建时 Sentry 配置 | 70-72 (插件启用), 87-103 (sentryConfig) |
| `react-router.config.ts` | 构建后钩子 | 16-24 (sentryOnBuildEnd) |
| `.github/workflows/deploy.yml` | CI/CD 配置 | 174 (COMMIT_SHA), 186-187 (构建参数) |
| `other/Dockerfile` | 镜像构建 | 32-33 (COMMIT_SHA), 36-37 (SENTRY_ORG/PROJECT), 50-52 (Build Secrets) |

---

**报告完成时间**: 2026-05-05  
**分析范围**: Epic Stack 当前代码库（不含假设性修改）  
**所有结论均有代码/配置证据支撑，可通过文中引用的文件位置核验。**
