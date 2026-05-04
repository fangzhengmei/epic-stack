# Express 中间件管道分析

本文档详细分析了 Epic Stack 中 Express 服务器的中间件处理流程，包括 HTTPS 重定向、压缩、安全头、限流、静态资源处理，以及请求如何最终传递给 React Router。

## 目录

- [中间件执行顺序](#中间件执行顺序)
- [HTTPS 重定向](#https-重定向)
- [URL 规范化（尾部斜杠处理）](#url-规范化尾部斜杠处理)
- [压缩](#压缩)
- [安全头](#安全头)
- [静态资源 404 快速处理](#静态资源-404-快速处理)
- [请求日志](#请求日志)
- [速率限制](#速率限制)
- [搜索引擎索引控制](#搜索引擎索引控制)
- [Dev vs Prod 模式分叉](#dev-vs-prod-模式分叉)
- [请求传递给 React Router](#请求传递给-react-router)
- [完整流程图](#完整流程图)

## 中间件执行顺序

Express 中间件按照 `app.use()` 调用的顺序执行。以下是完整的执行顺序：

| 顺序 | 中间件/处理 | 文件位置 | 说明 |
|------|------------|---------|------|
| 1 | Trust Proxy 设置 | `server/index.ts:29` | 信任代理服务器的 X-Forwarded-* 头 |
| 2 | HTTPS 重定向 | `server/index.ts:31-42` | HTTP 请求重定向到 HTTPS |
| 3 | URL 尾部斜杠处理 | `server/index.ts:46-54` | SEO 优化，移除尾部斜杠 |
| 4 | 压缩 | `server/index.ts:56` | gzip 压缩响应 |
| 5 | 禁用 X-Powered-By | `server/index.ts:59` | 隐藏服务器信息 |
| 6 | Helmet 安全头 | `server/index.ts:61-65` | 设置安全相关的 HTTP 头 |
| 7 | 图片资源 404 | `server/index.ts:67-71` | 快速处理缺失的图片资源 |
| 8 | Morgan 日志 | `server/index.ts:73-87` | 请求日志记录 |
| 9 | 速率限制 | `server/index.ts:89-149` | 多层级限流策略 |
| 10 | 搜索引擎控制 | `server/index.ts:151-156` | X-Robots-Tag 头 |
| 11 | **Dev/Prod 分叉** | `server/index.ts:158-195` | 开发模式 vs 生产模式 |
| 12 | React Router 处理 | `server/app.ts:15-23` | 最终请求处理 |

## HTTPS 重定向

### 实现位置
`server/index.ts:31-42`

### 代码分析
```typescript
// ensure HTTPS only (X-Forwarded-Proto comes from Fly)
app.use((req, res, next) => {
    if (req.method !== 'GET') return next()
    const proto = req.get('X-Forwarded-Proto')
    const host = getHost(req)
    if (proto === 'http') {
        res.set('X-Forwarded-Proto', 'https')
        res.redirect(`https://${host}${req.originalUrl}`)
        return
    }
    next()
})
```

### 关键要点
1. **只处理 GET 请求**：非 GET 请求直接跳过，因为 POST 等请求的重定向会丢失请求体
2. **依赖 X-Forwarded-Proto**：因为应用运行在 Fly.io 代理后面，需要读取代理设置的协议头
3. **重定向前设置头**：设置 `X-Forwarded-Proto: https` 防止循环重定向
4. **302 临时重定向**：使用默认的 302 状态码

### 前提条件
```typescript
app.set('trust proxy', true)
```
必须信任代理才能正确读取 `X-Forwarded-*` 头。

## URL 规范化（尾部斜杠处理）

### 实现位置
`server/index.ts:46-54`

### 代码分析
```typescript
// no ending slashes for SEO reasons
// https://github.com/epicweb-dev/epic-stack/discussions/108
app.get(/.*/, (req, res, next) => {
    if (req.path.endsWith('/') && req.path.length > 1) {
        const query = req.url.slice(req.path.length)
        const safepath = req.path.slice(0, -1).replace(/\/+/g, '/')
        res.redirect(302, safepath + query)
    } else {
        next()
    }
})
```

### 关键要点
1. **只处理 GET 请求**：使用 `app.get()` 而不是 `app.use()`
2. **SEO 优化**：避免 `/about` 和 `/about/` 被视为两个不同的页面
3. **保留查询参数**：正确处理路径后的查询字符串
4. **规范化多斜杠**：将 `//about//` 转换为 `/about`
5. **排除根路径**：`/` 路径不进行重定向

## 压缩

### 实现位置
`server/index.ts:56`

### 代码分析
```typescript
app.use(compression())
```

### 依赖库
使用 `compression` npm 包

### 默认行为
- 支持 gzip 和 deflate 压缩
- 自动根据 `Accept-Encoding` 请求头决定是否压缩
- 默认阈值：1KB（小于 1KB 的响应不压缩）

## 安全头

### 实现位置
`server/index.ts:58-65`

### 代码分析
```typescript
// http://expressjs.com/en/advanced/best-practice-security.html#at-a-minimum-disable-x-powered-by-header
app.disable('x-powered-by')

app.use((_, res, next) => {
    // The referrerPolicy breaks our redirectTo logic
    helmet(res, { general: { referrerPolicy: false } })
    next()
})
```

### 依赖库
使用 `@nichtsam/helmet/node-http`

### 安全措施
1. **禁用 X-Powered-By**：防止暴露服务器使用 Express
2. **Helmet 中间件**：设置多个安全相关的 HTTP 头，包括：
   - Content-Security-Policy
   - X-Content-Type-Options
   - X-Frame-Options
   - X-XSS-Protection
   - 等等

### 特殊配置
- **禁用 referrerPolicy**：因为它会破坏应用的 `redirectTo` 逻辑

## 静态资源 404 快速处理

### 实现位置
`server/index.ts:67-71`

### 代码分析
```typescript
app.get([/^\/img\/.*/, /^\/favicons\/.*/], (_req, res) => {
    // if we made it past the express.static for these, then we're missing something.
    // So we'll just send a 404 and won't bother calling other middleware.
    return res.status(404).send('Not found')
})
```

### 关键要点
1. **路径匹配**：使用正则表达式匹配 `/img/*` 和 `/favicons/*` 路径
2. **快速失败**：如果图片/图标资源不存在，直接返回 404
3. **性能优化**：避免对缺失的静态资源执行完整的中间件链

### 重要说明：注册顺序与执行流程

**注意**：这段代码在代码中的位置（第 67 行）早于静态资源中间件的注册位置（第 168/183/193 行）。但根据代码注释的意图，它应该作为静态资源服务的"安全网"。

实际的执行流程需要结合 `express.static` 的 `fallthrough` 选项来理解，请参考下文的 [静态资源分流逻辑详解](#静态资源分流逻辑详解)。

## 请求日志

### 实现位置
`server/index.ts:73-87`

### 代码分析
```typescript
morgan.token('url', (req) => {
    try {
        return decodeURIComponent(req.url ?? '')
    } catch {
        return req.url ?? ''
    }
})
app.use(
    morgan('tiny', {
        skip: (req, res) =>
            res.statusCode === 200 &&
            (req.url?.startsWith('/resources/images') ||
                req.url?.startsWith('/resources/healthcheck')),
    }),
)
```

### 依赖库
使用 `morgan` npm 包

### 配置说明
1. **自定义 URL Token**：对 URL 进行解码，便于阅读
2. **使用 tiny 格式**：简洁的日志格式（`:method :url :status :res[content-length] - :response-time ms`）
3. **跳过健康检查**：不记录成功的图片资源和健康检查请求，减少日志噪音

## 速率限制

### 实现位置
`server/index.ts:89-149`

### 代码分析
```typescript
// When running tests or running in development, we want to effectively disable
// rate limiting because playwright tests are very fast and we don't want to
// have to wait for the rate limit to reset between tests.
const maxMultiple =
    !IS_PROD || process.env.PLAYWRIGHT_TEST_BASE_URL ? 10_000 : 1
const rateLimitDefault = {
    windowMs: 60 * 1000,
    limit: 1000 * maxMultiple,
    standardHeaders: true,
    legacyHeaders: false,
    validate: { trustProxy: false },
    // Malicious users can spoof their IP address which means we should not default
    // to trusting req.ip when hosted on Fly.io. However, users cannot spoof Fly-Client-Ip.
    // When sitting behind a CDN such as cloudflare, replace fly-client-ip with the CDN
    // specific header such as cf-connecting-ip
    keyGenerator: (req: express.Request) => {
        const ip = req.ip ?? req.socket?.remoteAddress
        return req.get('fly-client-ip') ?? ipKeyGenerator(ip ?? '0.0.0.0')
    },
}

const strongestRateLimit = rateLimit({
    ...rateLimitDefault,
    windowMs: 60 * 1000,
    limit: 10 * maxMultiple,
})

const strongRateLimit = rateLimit({
    ...rateLimitDefault,
    windowMs: 60 * 1000,
    limit: 100 * maxMultiple,
})

const generalRateLimit = rateLimit(rateLimitDefault)
app.use((req, res, next) => {
    const strongPaths = [
        '/login',
        '/signup',
        '/verify',
        '/admin',
        '/onboarding',
        '/reset-password',
        '/settings/profile',
        '/resources/login',
        '/resources/verify',
    ]
    if (req.method !== 'GET' && req.method !== 'HEAD') {
        if (strongPaths.some((p) => req.path.includes(p))) {
            return strongestRateLimit(req, res, next)
        }
        return strongRateLimit(req, res, next)
    }

    // the verify route is a special case because it's a GET route that
    // can have a token in the query string
    if (req.path.includes('/verify')) {
        return strongestRateLimit(req, res, next)
    }

    return generalRateLimit(req, res, next)
})
```

### 依赖库
使用 `express-rate-limit` npm 包

### 限流策略

#### 三级限流策略

| 级别 | 限制 (生产) | 限制 (开发/测试) | 适用场景 |
|------|------------|-----------------|---------|
| **最强** (strongest) | 10 req/min | 100,000 req/min | 敏感路径的非 GET 请求 + /verify GET |
| **强** (strong) | 100 req/min | 1,000,000 req/min | 所有非 GET/HEAD 请求 |
| **通用** (general) | 1000 req/min | 10,000,000 req/min | GET/HEAD 请求 |

#### 敏感路径列表
- `/login`
- `/signup`
- `/verify`
- `/admin`
- `/onboarding`
- `/reset-password`
- `/settings/profile`
- `/resources/login`
- `/resources/verify`

### 关键安全特性
1. **IP 防欺骗**：使用 `Fly-Client-Ip` 头而不是 `req.ip`，防止用户在 Fly.io 上伪造 IP
2. **开发/测试友好**：非生产环境或 Playwright 测试时，限制放大 10,000 倍
3. **标准头支持**：设置 `RateLimit-*` 标准头，不设置旧版 `X-RateLimit-*` 头
4. **特殊处理 /verify**：即使是 GET 请求也使用最强限制，因为它包含敏感的 token 参数

## 搜索引擎索引控制

### 实现位置
`server/index.ts:151-156`

### 代码分析
```typescript
if (!ALLOW_INDEXING) {
    app.use((_, res, next) => {
        res.set('X-Robots-Tag', 'noindex, nofollow')
        next()
    })
}
```

### 环境变量
```typescript
const ALLOW_INDEXING = process.env.ALLOW_INDEXING !== 'false'
```

### 行为
- **默认允许索引**：除非 `ALLOW_INDEXING=false`
- **设置 X-Robots-Tag**：当禁用索引时，所有响应都包含 `noindex, nofollow`

## Dev vs Prod 模式分叉

### 实现位置
`server/index.ts:158-195`

### 核心判断
```typescript
const MODE = process.env.NODE_ENV ?? 'development'
const IS_PROD = MODE === 'production'
const IS_DEV = MODE === 'development'
```

### 开发模式 (IS_DEV)

```typescript
if (IS_DEV) {
    console.log('Starting development server')
    const viteDevServer = await import('vite').then((vite) =>
        vite.createServer({
            server: { middlewareMode: true },
            // We tell Vite we are running a custom app instead of
            // the SPA default so it doesn't run HTML middleware
            appType: 'custom',
        }),
    )
    app.use(viteDevServer.middlewares)
    app.use(async (req, res, next) => {
        try {
            const source = await viteDevServer.ssrLoadModule('./server/app.ts')
            return await source.app(req, res, next)
        } catch (error) {
            if (typeof error === 'object' && error instanceof Error) {
                viteDevServer.ssrFixStacktrace(error)
            }
            next(error)
        }
    })
}
```

#### 开发模式特点
1. **Vite 中间件模式**：启动 Vite 开发服务器作为中间件
2. **自定义应用类型**：`appType: 'custom'` 避免 Vite 的默认 HTML 中间件
3. **按需 SSR 加载**：使用 `viteDevServer.ssrLoadModule` 动态加载服务器模块
4. **热模块替换 (HMR)**：Vite 处理代码变更，无需重启服务器
5. **错误堆栈修复**：`ssrFixStacktrace` 提供更好的开发体验

### 生产模式 (!IS_DEV)

```typescript
else {
    console.log('Starting production server')
    // React Router fingerprints its assets so we can cache forever.
    app.use(
        '/assets',
        express.static('build/client/assets', {
            immutable: true,
            maxAge: '1y',
            fallthrough: false,
        }),
    )
    // Everything else (like favicon.ico) is cached for an hour. You may want to be
    // more aggressive with this caching.
    app.use(express.static('build/client', { maxAge: '1h' }))
    app.use(await import(BUILD_PATH).then((mod) => mod.app))
}
```

#### 生产模式特点
1. **预编译构建**：使用 `build/server/index.js` 而不是源代码
2. **静态资源服务**：
   - **Assets** (`/assets`)：`immutable: true` + `maxAge: 1y`（永久缓存，因为文件名带 hash）
   - **其他静态文件**：`maxAge: 1h`
3. **无 Vite 开销**：直接使用编译后的代码，性能最优
4. **无动态加载**：一次性导入构建产物

### 主要差异对比

| 特性 | 开发模式 | 生产模式 |
|------|---------|---------|
| **服务器** | Vite Dev Server (中间件模式) | 纯 Express |
| **模块加载** | `viteDevServer.ssrLoadModule()` (动态) | `import('../build/server/index.js')` (静态) |
| **静态资源** | Vite 中间件处理 | `express.static()` |
| **缓存策略** | 无缓存（开发需要刷新） | 永久缓存 (assets) + 1 小时 (其他) |
| **性能** | 较慢（按需编译） | 最快（预编译） |
| **HMR** | 支持 | 不支持 |
| **错误处理** | 堆栈修复 (`ssrFixStacktrace`) | 标准错误处理 |

## 请求传递给 React Router

### 实现位置
`server/app.ts:1-23`

### 代码分析
```typescript
import { createRequestHandler } from '@react-router/express'
import express from 'express'
import { type ServerBuild } from 'react-router'

declare module 'react-router' {
    interface AppLoadContext {
        serverBuild: ServerBuild
    }
}

export const app = express()

app.use(
    createRequestHandler({
        mode: process.env.NODE_ENV ?? 'development',
        build: () => import('virtual:react-router/server-build'),
        getLoadContext: async () => ({
            serverBuild: await import('virtual:react-router/server-build'),
        }),
    }),
)
```

### 依赖库
使用 `@react-router/express` 包

### 核心概念
`createRequestHandler` 将 Express 请求转换为 React Router 能够处理的格式，实现 SSR（服务端渲染）。

### 配置说明
1. **mode**：传递环境模式，影响 React Router 的错误处理和日志
2. **build**：返回 React Router 服务器构建的函数
   - 开发模式：通过 Vite 的虚拟模块 `virtual:react-router/server-build`
   - 生产模式：通过预编译的 `build/server/index.js`
3. **getLoadContext**：提供额外的上下文给 React Router 的 loaders/actions
   - 这里传递了 `serverBuild`，让应用可以访问构建信息

### 模块声明扩展
```typescript
declare module 'react-router' {
    interface AppLoadContext {
        serverBuild: ServerBuild
    }
}
```
扩展了 React Router 的 `AppLoadContext` 类型，添加 `serverBuild` 属性的类型安全。

## 完整流程图

```
请求进入
    ↓
[1] trust proxy 设置 (app.set('trust proxy', true))
    ↓
[2] HTTPS 重定向
    ├─ 是 GET 且 X-Forwarded-Proto === 'http'?
    │   └─ 是 → 302 重定向到 HTTPS
    │   └─ 否 → 继续
    ↓
[3] URL 尾部斜杠处理
    ├─ 是 GET 且路径以 '/' 结尾且不是 '/'?
    │   └─ 是 → 302 重定向到无斜杠路径
    │   └─ 否 → 继续
    ↓
[4] 压缩 (compression())
    ↓
[5] 禁用 X-Powered-By
    ↓
[6] Helmet 安全头
    ↓
[7] 图片资源 404 快速处理
    ├─ 路径匹配 /img/* 或 /favicons/*?
    │   └─ 是 → 返回 404 (静态资源缺失)
    │   └─ 否 → 继续
    ↓
[8] Morgan 请求日志
    ├─ 跳过条件: 状态码 200 且 URL 以 /resources/images 或 /resources/healthcheck 开头
    ↓
[9] 速率限制 (分级策略)
    ├─ 非 GET/HEAD 请求?
    │   ├─ 路径匹配敏感列表?
    │   │   └─ 是 → strongest (10 req/min 生产)
    │   │   └─ 否 → strong (100 req/min 生产)
    ├─ 是 GET/HEAD 且路径包含 /verify?
    │   └─ 是 → strongest
    └─ 其他 → general (1000 req/min 生产)
    ├─ 超限? → 返回 429 Too Many Requests
    └─ 未超限 → 继续
    ↓
[10] X-Robots-Tag (条件性)
    ├─ ALLOW_INDEXING === false?
    │   └─ 是 → 设置 X-Robots-Tag: noindex, nofollow
    │   └─ 否 → 无操作
    ↓
[11] DEV / PROD 分叉点
    │
    ├────────────────────────────────┬───────────────────────────────┐
    │                                │                               │
    ▼                                ▼                               ▼
[11a] 开发模式                    [11b] 生产模式                   │
    │                                │                               │
    ▼                                ▼                               │
[11a-1] Vite Dev Server          [11b-1] 静态资源服务              │
         中间件                          /assets (1年 immutable)     │
         (HMR, 按需编译)               其他 (1小时)                  │
    │                                │                               │
    ▼                                ▼                               │
[11a-2] 动态 SSR 加载             [11b-2] 预编译构建                │
         viteDevServer.                 import('../build/server')   │
         ssrLoadModule('./server/app.ts')                           │
    │                                │                               │
    └────────────────────────────────┴───────────────────────────────┘
                                    │
                                    ▼
[12] React Router 处理 (createRequestHandler)
    │
    ├─ 解析路由
    ├─ 执行 loaders
    ├─ 执行 actions (如果是表单提交等)
    ├─ 渲染组件 (SSR)
    └─ 返回 HTML 响应
```

## 关键设计亮点

### 1. 安全优先
- **多层级限流**：对敏感路径实施更严格的限制
- **IP 防欺骗**：依赖 Fly.io 的 `Fly-Client-Ip` 头
- **完整的安全头**：Helmet 提供全面的安全防护

### 2. 性能优化
- **gzip 压缩**：减少传输体积
- **分级静态缓存**：assets 永久缓存，其他文件适中缓存
- **快速失败**：缺失的静态资源直接返回 404，不执行完整中间件链

### 3. 开发体验
- **Vite 集成**：HMR 热更新，无需重启
- **堆栈修复**：`ssrFixStacktrace` 提供可读的错误堆栈
- **限流放宽**：开发/测试环境限流放大 10,000 倍

### 4. SEO 友好
- **URL 规范化**：统一尾部斜杠处理
- **HTTPS 强制**：提升搜索排名
- **可配置的索引控制**：通过环境变量控制搜索引擎索引

## 文件位置汇总

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 主服务器入口 | `server/index.ts` | 全文 |
| Trust Proxy | `server/index.ts` | 29 |
| HTTPS 重定向 | `server/index.ts` | 31-42 |
| URL 尾部斜杠 | `server/index.ts` | 46-54 |
| 压缩 | `server/index.ts` | 56 |
| 安全头 | `server/index.ts` | 58-65 |
| 图片 404 | `server/index.ts` | 67-71 |
| 日志 | `server/index.ts` | 73-87 |
| 限流 | `server/index.ts` | 89-149 |
| 搜索引擎控制 | `server/index.ts` | 151-156 |
| Dev/Prod 分叉 | `server/index.ts` | 158-195 |
| React Router 处理 | `server/app.ts` | 1-23 |
