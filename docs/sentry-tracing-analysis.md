# Epic Stack Sentry 分布式追踪分析报告

## 概述

本报告详细分析 Epic Stack 中 Sentry 在三个执行环境（客户端、服务端、边缘中间件）的初始化配置差异，以及如何通过追踪关联机制将不同环境中的错误关联到同一次请求。

---

## 一、执行环境与 SDK 初始化差异

### 1.1 客户端环境 (Browser)

**文件位置**: `app/utils/monitoring.client.tsx`

**核心配置**:

```typescript
import * as Sentry from '@sentry/react-router'

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

**关键特性**:
- **SDK**: `@sentry/react-router` (统一 SDK)
- **环境变量**: 使用 `ENV.SENTRY_DSN` 和 `ENV.MODE`
- **集成项**:
  - `replayIntegration()`: 会话重放，用于录制用户交互
  - `browserProfilingIntegration()`: 浏览器性能分析
- **过滤机制**: `beforeSend` 钩子过滤浏览器扩展产生的错误
- **采样配置**:
  - 追踪采样率: 100%
  - 会话重放采样率: 10%
  - 错误会话重放采样率: 100%

**初始化时机**: `app/entry.client.tsx`

```typescript
if (ENV.MODE === 'production' && ENV.SENTRY_DSN) {
    void import('./utils/monitoring.client.tsx').then(({ init }) => init())
}
```

---

### 1.2 服务端环境 (Node.js)

**文件位置**: `server/utils/monitoring.ts`

**核心配置**:

```typescript
import { PrismaInstrumentation } from '@prisma/instrumentation'
import { nodeProfilingIntegration } from '@sentry/profiling-node'
import * as Sentry from '@sentry/react-router'

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

**关键特性**:
- **SDK**: `@sentry/react-router` (统一 SDK)
- **环境变量**: 使用 `process.env.SENTRY_DSN` 和 `process.env.NODE_ENV`
- **集成项**:
  - `prismaIntegration()`: Prisma ORM 数据库操作追踪
  - `httpIntegration()`: HTTP 请求/响应追踪
  - `nodeProfilingIntegration()`: Node.js 性能分析
- **过滤机制**:
  - `denyUrls`: URL 黑名单，忽略健康检查和静态资源请求
  - `tracesSampler`: 动态采样决策，健康检查请求采样率为 0
  - `beforeSendTransaction`: 事务发送前过滤，忽略健康检查事务
- **采样策略**: 生产环境 100% 采样，开发环境 0%

**初始化时机**: `server/index.ts`

```typescript
const SENTRY_ENABLED = IS_PROD && process.env.SENTRY_DSN

if (SENTRY_ENABLED) {
    void import('./utils/monitoring.ts').then(({ init }) => init())
}
```

---

### 1.3 边缘中间件 (Edge Middleware)

**现状分析**:

当前 Epic Stack 代码库中**未发现独立的边缘中间件配置** (`middleware.ts`)。这可能是因为：

1. **架构选择**: 项目采用传统 Node.js 服务端架构，未使用 Vercel Edge Runtime
2. **统一 SDK**: `@sentry/react-router` SDK 设计上已经统一了浏览器、Node.js 和 Edge 环境的 API

**如果需要边缘中间件配置，参考配置如下**:

```typescript
// middleware.ts (参考配置)
import * as Sentry from '@sentry/react-router'

export function init() {
    Sentry.init({
        dsn: process.env.SENTRY_DSN,
        environment: process.env.NODE_ENV,
        tracesSampleRate: 1.0,
        // Edge 环境不支持某些集成
        integrations: [
            // Edge 环境可用的集成
        ],
    })
}
```

---

## 二、三个环境初始化关键差异对比

| 特性 | 客户端 (Browser) | 服务端 (Node.js) | 边缘中间件 (Edge) |
|------|------------------|------------------|-------------------|
| **SDK 包** | `@sentry/react-router` | `@sentry/react-router` | `@sentry/react-router` |
| **环境变量** | `ENV.*` (客户端注入) | `process.env.*` | `process.env.*` |
| **主要集成** | replay, browserProfiling | prisma, http, nodeProfiling | 受限 (Edge Runtime 限制) |
| **采样配置** | `tracesSampleRate: 1.0` | `tracesSampler` 动态 | 类似服务端 |
| **错误过滤** | `beforeSend` (扩展过滤) | `denyUrls`, `beforeSendTransaction` | 类似服务端 |
| **Profiling** | 浏览器性能分析 | Node.js 性能分析 | 不支持 |
| **Replay** | 支持会话重放 | 不适用 | 不适用 |
| **初始化时机** | `entry.client.tsx` | `server/index.ts` | `middleware.ts` |

---

## 三、分布式追踪关联机制

### 3.1 核心原理

Sentry 通过 **Trace Context** 标准实现跨环境的请求追踪关联。核心概念包括：

1. **Trace ID**: 整个请求链路的唯一标识符
2. **Span ID**: 单个操作的标识符
3. **Parent Span ID**: 父操作标识符，构建调用链
4. **Baggage Header**: 携带附加元数据

### 3.2 自动关联机制

Epic Stack 使用 `@sentry/react-router` SDK，该 SDK 自动处理以下关联：

#### 3.2.1 服务端 → 客户端 关联

**机制**: 服务端渲染 (SSR) 期间，Sentry 自动在 HTML 中注入追踪上下文

```html
<!-- 自动注入的 meta 标签 -->
<meta name="sentry-trace" content="trace-id-span-id-flags">
<meta name="baggage" content="sentry-environment=production,...">
```

**流程**:
1. 服务端接收请求，创建 Trace
2. 服务端渲染 HTML，注入 `sentry-trace` 和 `baggage` meta 标签
3. 客户端 Sentry SDK 初始化时读取这些 meta 标签
4. 客户端错误自动关联到同一 Trace

#### 3.2.2 客户端 → 服务端 API 调用关联

**机制**: Sentry 自动在 Fetch/XHR 请求中添加追踪头

```http
# 客户端发起请求时自动添加的头
sentry-trace: {trace_id}-{span_id}-{sampled}
baggage: sentry-environment=production,sentry-public_key=...
```

**流程**:
1. 客户端发起 API 请求
2. Sentry SDK 自动注入 `sentry-trace` 和 `baggage` 头
3. 服务端 Sentry SDK 读取这些头
4. 服务端操作作为子 Span 关联到客户端 Trace

#### 3.2.3 服务端内部操作关联

**机制**: 通过 OpenTelemetry 风格的上下文传播

**关键集成**:
- **Prisma 集成**: `prismaIntegration()` 自动追踪数据库查询
- **HTTP 集成**: `httpIntegration()` 自动追踪 HTTP 调用

**示例链路**:
```
客户端请求 (Span A)
  ↓
服务端 Loader/Action (Span B, 父: A)
  ↓
Prisma 查询 (Span C, 父: B)
  ↓
HTTP 调用外部 API (Span D, 父: B)
```

### 3.3 手动关联机制

#### 3.3.1 服务端错误捕获

**文件位置**: `app/entry.server.tsx:125-141`

```typescript
export function handleError(
    error: unknown,
    { request }: LoaderFunctionArgs | ActionFunctionArgs,
): void {
    // 跳过已中止的请求
    if (request.signal.aborted) {
        return
    }

    if (error instanceof Error) {
        console.error(styleText('red', String(error.stack)))
    } else {
        console.error(error)
    }

    // 手动捕获异常，自动关联当前 Trace
    Sentry.captureException(error)
}
```

**关键点**:
- `Sentry.captureException(error)` 自动使用当前活动的 Span 上下文
- 错误会关联到当前请求的 Trace

#### 3.3.2 进程优雅关闭时的错误捕获

**文件位置**: `server/index.ts:236-248`

```typescript
closeWithGrace(async ({ err }) => {
    await new Promise((resolve, reject) => {
        server.close((e) => (e ? reject(e) : resolve('ok')))
    })
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

### 3.4 构建时配置

**文件位置**: `vite.config.ts:87-103`

```typescript
const sentryConfig: SentryReactRouterBuildOptions = {
    authToken: process.env.SENTRY_AUTH_TOKEN,
    org: process.env.SENTRY_ORG,
    project: process.env.SENTRY_PROJECT,

    unstable_sentryVitePluginOptions: {
        release: {
            name: process.env.COMMIT_SHA,  // 使用 commit SHA 作为版本名
            setCommits: {
                auto: true,  // 自动关联提交
            },
        },
        sourcemaps: {
            filesToDeleteAfterUpload: ['./build/**/*.map'],  // 上传后删除源码映射
        },
    },
}
```

**构建时关联的作用**:
1. **Release 关联**: 使用 `COMMIT_SHA` 作为版本名，错误可关联到具体代码版本
2. **Source Maps**: 上传源码映射，将压缩后的错误堆栈映射到原始代码
3. **Commit 关联**: 自动关联 Git 提交，便于追踪错误引入的代码变更

---

## 四、完整请求链路示例

### 4.1 典型 SSR + CSR 混合场景

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户浏览器                                  │
│  ┌──────────────┐     ┌─────────────────────────────────────┐  │
│  │ 初始页面加载  │────▶│ Sentry: 创建 Trace (trace_id: abc123) │  │
│  └──────────────┘     └─────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      HTTP 请求 (带追踪头)                         │
│  sentry-trace: abc123-span001-1                                │
│  baggage: sentry-environment=production                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Node.js 服务端                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Sentry: 继续 Trace (trace_id: abc123)                   │   │
│  │   - Span: HTTP 请求处理 (span_id: span001)              │   │
│  │   - Span: Loader 执行 (span_id: span002, 父: span001)  │   │
│  │   - Span: Prisma 查询 (span_id: span003, 父: span002)  │   │
│  │   - Span: 渲染 HTML (span_id: span004, 父: span001)    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 注入到 HTML 的 meta 标签:                                  │   │
│  │   <meta name="sentry-trace" content="abc123-span005-1">│   │
│  │   <meta name="baggage" content="sentry-environment=...">│   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    HTML 返回给浏览器                              │
│  包含 sentry-trace 和 baggage meta 标签                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        用户浏览器 (Hydration)                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Sentry: 读取 meta 标签，继续同一 Trace                     │   │
│  │   - Trace ID: abc123 (继续服务端的 Trace)                │   │
│  │   - Span: 客户端初始化 (span_id: span006)                │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    后续客户端 API 调用                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 自动添加追踪头:                                            │   │
│  │   sentry-trace: abc123-span007-1                        │   │
│  │   baggage: sentry-environment=production                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 服务端接收:                                                │   │
│  │   - 识别 trace_id: abc123                                │   │
│  │   - 作为子 Span 继续同一 Trace                             │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 错误关联场景

假设在客户端点击按钮触发 API 调用，服务端 Loader 执行时数据库查询失败：

```
Sentry UI 中看到的完整链路:

Trace ID: abc123
┌──────────────────────────────────────────────────────────────┐
│ Span 007: 客户端按钮点击 (Browser)                            │
│   - 操作: click on button                                     │
│   - 用户: user@example.com                                    │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ Span 008: Fetch API 调用 (Browser → Server)                  │
│   - URL: /api/data                                            │
│   - Method: POST                                              │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ Span 009: HTTP 请求处理 (Node.js)                             │
│   - 路由: /api/data                                           │
│   - 状态码: 500                                               │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ Span 010: Action 执行 (Node.js)                               │
│   - 函数: action                                               │
│   - 持续时间: 150ms                                            │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ Span 011: Prisma 查询 (Node.js) ──── 错误发生点               │
│   - 查询: SELECT * FROM users WHERE id = ?                   │
│   - 错误: Connection timeout                                  │
│   - 堆栈: db.ts:42                                            │
└──────────────────────────────────────────────────────────────┘

关联的错误事件:
┌──────────────────────────────────────────────────────────────┐
│ Error: Database connection timeout                            │
│   - Trace ID: abc123                                          │
│   - Span ID: span011                                          │
│   - 环境: production                                           │
│   - 版本: commit-xyz123                                        │
│   - 用户: user@example.com (来自 baggage)                      │
└──────────────────────────────────────────────────────────────┘
```

---

## 五、关键设计决策分析

### 5.1 统一 SDK 策略

Epic Stack 选择使用 `@sentry/react-router` 统一 SDK，而非分别使用 `@sentry/browser`、`@sentry/node` 等独立包：

**优点**:
1. **简化配置**: 一套 API，多环境运行
2. **自动适配**: SDK 内部根据运行时环境自动选择正确的集成
3. **易于维护**: 减少重复配置代码

**实现方式**:
```typescript
// 客户端自动使用浏览器集成
// 服务端自动使用 Node.js 集成
import * as Sentry from '@sentry/react-router'
```

### 5.2 条件初始化

三个环境都采用条件初始化策略：

```typescript
// 客户端
if (ENV.MODE === 'production' && ENV.SENTRY_DSN) {
    void import('./utils/monitoring.client.tsx').then(({ init }) => init())
}

// 服务端
const SENTRY_ENABLED = IS_PROD && process.env.SENTRY_DSN
if (SENTRY_ENABLED) {
    void import('./utils/monitoring.ts').then(({ init }) => init())
}
```

**设计考虑**:
1. **开发环境性能**: 开发环境不初始化 Sentry，提升热重载速度
2. **按需加载**: 使用动态 `import()`，减少初始 bundle 大小
3. **依赖 DSN**: 仅在配置了 DSN 时才初始化

### 5.3 采样策略分层

| 环境 | 采样策略 | 目的 |
|------|----------|------|
| 客户端 | 100% traces, 10% replays | 全面追踪，经济会话重放 |
| 服务端 | 动态采样 (健康检查 0%) | 忽略噪音，聚焦业务请求 |
| 错误会话 | 100% replay | 错误时完整录制便于调试 |

### 5.4 错误边界与手动捕获

Epic Stack 结合了 Sentry 的自动错误边界和手动捕获：

1. **自动捕获**: `@sentry/react-router` 自动集成 React Router 的错误边界
2. **手动捕获**: `entry.server.tsx` 中的 `handleError` 函数手动调用 `Sentry.captureException()`

**双保险机制**:
```typescript
// 自动: React Router 错误边界捕获 UI 错误
// 手动: handleError 捕获 Loader/Action 错误
export function handleError(error: unknown, { request }: ...) {
    // ...
    Sentry.captureException(error)
}
```

---

## 六、最佳实践总结

### 6.1 配置最佳实践

1. **环境变量隔离**: 使用不同的 Sentry DSN 或 environment 区分开发、测试、生产环境
2. **源码映射**: 生产环境必须上传 sourcemaps，便于定位错误
3. **Release 命名**: 使用 commit SHA 或语义化版本，便于追踪代码变更

### 6.2 追踪关联最佳实践

1. **不修改追踪头**: 避免在中间件中修改或删除 `sentry-trace` 和 `baggage` 头
2. **跨服务调用**: 确保内部服务调用时传递这些头
3. **用户上下文**: 使用 `Sentry.setUser()` 设置用户信息，便于追踪用户行为

### 6.3 性能考虑

1. **采样率调整**: 高流量环境降低 `tracesSampleRate`
2. **健康检查过滤**: 使用 `denyUrls` 和 `tracesSampler` 过滤健康检查
3. **动态导入**: 生产环境才加载 Sentry，减少开发环境开销

---

## 七、附录：相关文件索引

| 文件路径 | 说明 |
|----------|------|
| `app/utils/monitoring.client.tsx` | 客户端 Sentry 初始化配置 |
| `server/utils/monitoring.ts` | 服务端 Sentry 初始化配置 |
| `app/entry.client.tsx` | 客户端入口，条件初始化 Sentry |
| `app/entry.server.tsx` | 服务端入口，handleError 错误捕获 |
| `server/index.ts` | Express 服务启动，服务端 Sentry 初始化 |
| `vite.config.ts` | Sentry Vite 插件配置，构建时上传 sourcemaps |
| `docs/monitoring.md` | 官方监控配置文档 |

---

**报告生成时间**: 2026-05-05  
**分析版本**: Epic Stack (基于当前代码库)  
**Sentry SDK 版本**: `@sentry/react-router@^10.38.0`
