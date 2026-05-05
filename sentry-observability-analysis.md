# Epic Stack Sentry 可观测性分析报告

> **报告版本**: v3.0  
> **分析日期**: 2026-05-05  
> **证据状态**: 仅基于仓内代码/配置证据，无推测性内容

---

## 一、概述

本报告基于 Epic Stack 代码仓库内的实际代码和配置，分析 Sentry 集成的错误捕获机制、上下文传递方式以及跨环境关联策略。所有结论均有仓内证据支撑。

---

## 二、架构概览

### 2.1 运行环境支持现状

| 运行环境 | 仓内配置状态 | 证据依据 |
|---------|-------------|---------|
| **Node.js 服务端** | 有配置 | `server/index.ts`、`server/utils/monitoring.ts` |
| **Browser 客户端** | 有配置 | `app/entry.client.tsx`、`app/utils/monitoring.client.tsx` |
| **Edge 运行时** | 无配置 | 见下文"Edge 运行时现状分析" |

### 2.2 核心依赖（仓内证据）

```json
// package.json:64-65
"@sentry/profiling-node": "^10.38.0",
"@sentry/react-router": "^10.38.0",
```

---

## 三、Edge 运行时现状分析

### 3.1 仓内证据清单

#### 证据 1：依赖包含 `@sentry/profiling-node`

```json
// package.json:64
"@sentry/profiling-node": "^10.38.0",
```

#### 证据 2：服务端配置使用 `nodeProfilingIntegration` 和 `prismaIntegration`

```typescript
// server/utils/monitoring.ts:1-2, 19-24
import { PrismaInstrumentation } from '@prisma/instrumentation'
import { nodeProfilingIntegration } from '@sentry/profiling-node'

integrations: [
	Sentry.prismaIntegration({
		prismaInstrumentation: new PrismaInstrumentation(),
	}),
	Sentry.httpIntegration(),
	nodeProfilingIntegration(),
],
```

#### 证据 3：服务器框架为 `express`

```typescript
// server/index.ts:7
import express from 'express'
```

#### 证据 4：`engines` 声明要求 Node.js

```json
// package.json:156-158
"engines": {
  "node": "^22.18.0"
}
```

#### 证据 5：Docker 基础镜像为 Node.js

```dockerfile
// other/Dockerfile:4
FROM node:22-bookworm-slim as base
```

#### 证据 6：仓内无 Edge 运行时配置

仓内搜索结果：
- 无 `runtime: "edge"` 声明
- 无 `export const config = { runtime: 'edge' }`
- 无 `@cloudflare/workers-types` 依赖
- 无 `vercel.json` 中的 Edge 配置
- 无 `EdgeRuntime` 相关代码

### 3.2 明确结论

> **仓内代码未配置 Edge 运行时的 Sentry 集成。**

---

## 四、错误捕获机制分析

### 4.1 服务端错误捕获

#### 4.1.1 初始化入口

```typescript
// server/index.ts:16-21
const SENTRY_ENABLED = IS_PROD && process.env.SENTRY_DSN
const BUILD_PATH = '../build/server/index.js'

if (SENTRY_ENABLED) {
	void import('./utils/monitoring.ts').then(({ init }) => init())
}
```

#### 4.1.2 服务端配置

```typescript
// server/utils/monitoring.ts:5-42
export function init() {
	Sentry.init({
		dsn: process.env.SENTRY_DSN,
		environment: process.env.NODE_ENV,
		denyUrls: [
			/\/resources\/healthcheck/,
			/\/build\//,
			/\/favicons\//,
			/\/img\//,
			/\/fonts\//,
			/\/favicon.ico/,
			/\/site\.webmanifest/,
		],
		integrations: [
			Sentry.prismaIntegration({
				prismaInstrumentation: new PrismaInstrumentation(),
			}),
			Sentry.httpIntegration(),
			nodeProfilingIntegration(),
		],
		tracesSampler(samplingContext) {
			if (samplingContext.request?.url?.includes('/resources/healthcheck')) {
				return 0
			}
			return process.env.NODE_ENV === 'production' ? 1 : 0
		},
		beforeSendTransaction(event) {
			if (event.request?.headers?.['x-healthcheck'] === 'true') {
				return null
			}
			return event
		},
	})
}
```

#### 4.1.3 错误捕获点

| 捕获点 | 文件位置 | 代码证据 |
|-------|---------|---------|
| **Loader/Action 错误** | `app/entry.server.tsx:125-142` | `handleError` 函数 |
| **服务器关闭错误** | `server/index.ts:236-248` | `closeWithGrace` 回调 |

**证据 1：Loader/Action 错误捕获**

```typescript
// app/entry.server.tsx:125-142
export function handleError(
	error: unknown,
	{ request }: LoaderFunctionArgs | ActionFunctionArgs,
): void {
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
			Sentry.captureException(err)
			await Sentry.flush(500)
		}
	}
})
```

---

### 4.2 客户端错误捕获

#### 4.2.1 初始化入口

```typescript
// app/entry.client.tsx:5-7
if (ENV.MODE === 'production' && ENV.SENTRY_DSN) {
	void import('./utils/monitoring.client.tsx').then(({ init }) => init())
}
```

#### 4.2.2 客户端配置

```typescript
// app/utils/monitoring.client.tsx:3-35
export function init() {
	Sentry.init({
		dsn: ENV.SENTRY_DSN,
		environment: ENV.MODE,
		beforeSend(event) {
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
		tracesSampleRate: 1.0,
		replaysSessionSampleRate: 0.1,
		replaysOnErrorSampleRate: 1.0,
	})
}
```

#### 4.2.3 错误捕获点

| 捕获点 | 文件位置 | 代码证据 |
|-------|---------|---------|
| **React 组件错误** | `app/components/error-boundary.tsx:37-41` | `useEffect` 内 `captureException` |

**证据**：

```typescript
// app/components/error-boundary.tsx:37-41
useEffect(() => {
	if (isResponse) return

	captureException(error)
}, [error, isResponse])
```

---

## 五、跨环境关联链路（可核验）

### 5.1 关联维度总览

| 关联维度 | 构建发布阶段 | 服务端运行时 | 客户端运行时 | 仓内证据 |
|---------|------------|-------------|-------------|---------|
| **DSN** | 无配置 | `process.env.SENTRY_DSN` | `ENV.SENTRY_DSN` | 见链路 1 |
| **Environment** | 无配置 | `process.env.NODE_ENV` | `ENV.MODE` | 见链路 2 |
| **Release** | `COMMIT_SHA` | 构建时配置 | 构建时配置 | 见链路 3 |
| **Trace** | 无配置 | 无显式配置 | 无显式配置 | 仓内无证据 |

---

### 5.2 链路 1：DSN 关联（可核验）

#### 服务端配置

```typescript
// server/utils/monitoring.ts:7
dsn: process.env.SENTRY_DSN,
```

#### 客户端配置

```typescript
// app/utils/monitoring.client.tsx:5
dsn: ENV.SENTRY_DSN,
```

#### 传递链路

```
Step 1: 环境变量 Schema 定义
        ↓ app/utils/env.server.ts:12
        SENTRY_DSN: z.string().optional()

Step 2: getEnv() 暴露给客户端
        ↓ app/utils/env.server.ts:59-65
        export function getEnv() {
          return {
            MODE: process.env.NODE_ENV,
            SENTRY_DSN: process.env.SENTRY_DSN,
            ALLOW_INDEXING: process.env.ALLOW_INDEXING,
          }
        }

Step 3: Root Loader 返回 ENV
        ↓ app/root.tsx:111-133
        return data({
          user,
          requestInfo: { ... },
          ENV: getEnv(),
          toast,
          honeyProps,
        })

Step 4: HTML 注入 window.ENV
        ↓ app/root.tsx:163-168
        <script dangerouslySetInnerHTML={{
            __html: `window.ENV = ${JSON.stringify(env)}`
        }} />

Step 5: 客户端使用
        ↓ app/utils/monitoring.client.tsx:5
        dsn: ENV.SENTRY_DSN
```

#### 结论

仓内代码显示：服务端和客户端使用同一来源的 `SENTRY_DSN` 配置。

---

### 5.3 链路 2：Environment 关联（可核验）

#### 服务端配置

```typescript
// server/utils/monitoring.ts:8
environment: process.env.NODE_ENV,
```

#### 客户端配置

```typescript
// app/utils/monitoring.client.tsx:6
environment: ENV.MODE,
```

#### 传递链路

```
Step 1: 服务端 NODE_ENV
        ↓ app/utils/env.server.ts:61
        MODE: process.env.NODE_ENV

Step 2: 客户端 ENV.MODE
        ↓ app/entry.client.tsx:5
        if (ENV.MODE === 'production' && ENV.SENTRY_DSN)
        ↓ app/utils/monitoring.client.tsx:6
        environment: ENV.MODE
```

#### 结论

仓内代码显示：服务端 `NODE_ENV` 与客户端 `ENV.MODE` 来源相同。

---

### 5.4 链路 3：Release 关联（可核验）

#### 构建发布阶段配置

**Step 1: GitHub Actions 传递 COMMIT_SHA**

```yaml
// .github/workflows/deploy.yml:174, 186
--build-arg COMMIT_SHA=${{ github.sha }}
```

**Step 2: Dockerfile 接收并设置**

```dockerfile
// other/Dockerfile:32-33
ARG COMMIT_SHA
ENV COMMIT_SHA=$COMMIT_SHA
```

**Step 3: Docker Build 阶段执行构建**

```dockerfile
// other/Dockerfile:50-52
RUN --mount=type=secret,id=SENTRY_AUTH_TOKEN \
  export SENTRY_AUTH_TOKEN=$(cat /run/secrets/SENTRY_AUTH_TOKEN) && \
  npm run build
```

**Step 4: Vite 配置使用 COMMIT_SHA**

```typescript
// vite.config.ts:87-103
const sentryConfig: SentryReactRouterBuildOptions = {
	authToken: process.env.SENTRY_AUTH_TOKEN,
	org: process.env.SENTRY_ORG,
	project: process.env.SENTRY_PROJECT,

	unstable_sentryVitePluginOptions: {
		release: {
			name: process.env.COMMIT_SHA,
			setCommits: {
				auto: true,
			},
		},
		sourcemaps: {
			filesToDeleteAfterUpload: ['./build/**/*.map'],
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

#### 结论

仓内代码显示：
- 构建阶段使用 `COMMIT_SHA` 作为 Release 名称
- 服务端和客户端代码均由同一 Vite 构建流程生成

---

### 5.5 链路 4：Trace 关联（仓内证据检查）

#### 仓内搜索结果

在整个代码库中搜索以下关键词，**均未找到**：
- `tracePropagationTargets`
- `sentry-trace`
- `baggage`
- `propagateTraces`
- `setHttpStatus`
- `startTransaction`
- `continueTrace`

#### 明确结论

> **仓内无 Trace 传播相关的显式配置，不作结论。**

---

## 六、环境变量配置（可核验）

### 6.1 变量清单

| 变量名 | 配置位置 | 仓内证据 |
|-------|---------|---------|
| `SENTRY_DSN` | 运行时 | `env.server.ts:12`, `monitoring.ts:7`, `monitoring.client.tsx:5` |
| `NODE_ENV` / `MODE` | 运行时 | `env.server.ts:61`, `monitoring.ts:8`, `monitoring.client.tsx:6` |
| `SENTRY_AUTH_TOKEN` | 仅构建时 | `deploy.yml:187`, `Dockerfile:50`, `vite.config.ts:88` |
| `SENTRY_ORG` | 仅构建时 | `vite.config.ts:89`, `Dockerfile:36` |
| `SENTRY_PROJECT` | 仅构建时 | `vite.config.ts:90`, `Dockerfile:37` |
| `COMMIT_SHA` | 仅构建时 | `deploy.yml:174`, `Dockerfile:32-33`, `vite.config.ts:94` |

### 6.2 敏感变量处理

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

---

## 七、过滤与采样配置（可核验）

### 7.1 服务端过滤

| 过滤类型 | 配置位置 | 仓内证据 |
|---------|---------|---------|
| URL 黑名单 | `monitoring.ts:9-18` | `denyUrls` 数组 |
| 采样过滤 | `monitoring.ts:26-32` | `tracesSampler` 函数 |
| 事务过滤 | `monitoring.ts:33-41` | `beforeSendTransaction` 函数 |
| 请求中止过滤 | `entry.server.tsx:131-133` | `request.signal.aborted` 检查 |

### 7.2 客户端过滤

| 过滤类型 | 配置位置 | 仓内证据 |
|---------|---------|---------|
| 浏览器扩展 | `monitoring.client.tsx:7-19` | `beforeSend` 检查协议 |
| HTTP 响应错误 | `error-boundary.tsx:38` | `isResponse` 检查 |
| 开发环境 | `entry.client.tsx:5` | `ENV.MODE === 'production'` 检查 |

### 7.3 采样配置

| 采样项 | 服务端配置 | 客户端配置 |
|-------|-----------|-----------|
| 性能追踪 | `tracesSampler` 动态函数 | `tracesSampleRate: 1.0` |
| 会话回放 | 无配置 | `replaysSessionSampleRate: 0.1`, `replaysOnErrorSampleRate: 1.0` |

---

## 八、关键结论汇总

### 8.1 已确认的关联机制

| 关联维度 | 状态 | 仓内证据 |
|---------|------|---------|
| **DSN** | 服务端和客户端使用同一来源配置 | `env.server.ts`、`root.tsx` |
| **Environment** | 服务端和客户端来源相同 | `env.server.ts:61` |
| **Release** | 构建时使用 `COMMIT_SHA` 配置 | `vite.config.ts:94`、`deploy.yml:174` |
| **Trace** | 仓内无显式配置 | 搜索结果为空 |

### 8.2 运行时配置状态

| 运行环境 | 状态 | 仓内证据 |
|---------|------|---------|
| **Node.js 服务端** | 有完整配置 | `server/utils/monitoring.ts` |
| **Browser 客户端** | 有完整配置 | `app/utils/monitoring.client.tsx` |
| **Edge 运行时** | 无配置 | 搜索结果为空 |

---

## 九、参考文件（证据索引）

| 文件路径 | 证据类型 | 关键行号 |
|---------|---------|---------|
| `package.json` | 依赖声明 | 64-65 (Sentry 依赖), 156-158 (Node 版本) |
| `server/index.ts` | 服务端初始化 | 16-21 (Sentry 初始化), 236-248 (关闭错误捕获) |
| `server/utils/monitoring.ts` | 服务端配置 | 5-42 (完整配置) |
| `app/entry.server.tsx` | 服务端错误捕获 | 125-142 (handleError) |
| `app/entry.client.tsx` | 客户端初始化 | 5-7 (初始化条件) |
| `app/utils/monitoring.client.tsx` | 客户端配置 | 3-35 (完整配置) |
| `app/components/error-boundary.tsx` | 客户端错误捕获 | 37-41 (captureException) |
| `app/root.tsx` | ENV 注入 | 122 (ENV 暴露), 163-168 (window.ENV 注入) |
| `app/utils/env.server.ts` | 环境变量定义 | 12 (SENTRY_DSN), 59-65 (getEnv) |
| `vite.config.ts` | 构建时配置 | 70-72 (插件启用), 87-103 (sentryConfig) |
| `react-router.config.ts` | 构建后钩子 | 16-24 (sentryOnBuildEnd) |
| `.github/workflows/deploy.yml` | CI/CD 配置 | 174 (COMMIT_SHA), 186-187 (构建参数) |
| `other/Dockerfile` | 镜像构建 | 32-33 (COMMIT_SHA), 36-37 (SENTRY_ORG/PROJECT), 50-52 (Build Secrets) |

---

**报告完成时间**: 2026-05-05  
**分析范围**: 仅基于仓内实际代码和配置  
**所有结论均有仓内证据支撑，无推测性内容。**
