# Epic Stack 非页面性服务端路由与缓存策略分析报告

## 一、非页面性服务端路由概览

Epic Stack 中承担非页面性服务端职责的路由主要分为以下几类：

### 1. Resources 路由 (app/routes/resources/)

| 路由路径 | 文件位置 | 主要职责 |
|---------|---------|---------|
| `/resources/healthcheck` | `healthcheck.tsx` | 服务端健康检查 |
| `/resources/theme-switch` | `theme-switch.tsx` | 主题切换服务（设置 Cookie） |
| `/resources/images` | `images.tsx` | 图片优化与代理服务 |
| `/resources/download-user-data` | `download-user-data.tsx` | 用户数据导出下载 |

### 2. SEO 路由 (app/routes/_seo/)

| 路由路径 | 文件位置 | 主要职责 |
|---------|---------|---------|
| `/robots.txt` | `robots[.]txt.ts` | 生成 robots.txt 文件 |
| `/sitemap.xml` | `sitemap[.]xml.ts` | 生成 sitemap.xml 文件 |

### 3. 缓存管理路由 (app/routes/admin/cache/)

| 路由路径 | 文件位置 | 主要职责 |
|---------|---------|---------|
| `/admin/cache` | `index.tsx` | 缓存管理界面（需要 admin 权限） |
| `/admin/cache/sqlite` | `sqlite.server.ts` | SQLite 缓存同步端点（内部通信） |

---

## 二、缓存策略配置

### 1. HTTP 缓存头策略

Epic Stack 使用 `@tusbar/cache-control` 库来解析和格式化 Cache-Control 响应头，核心逻辑在 `app/utils/headers.server.ts` 中实现。

#### 2.1.1 保守缓存合并策略

`getConservativeCacheControl` 函数会合并多个 Cache-Control 头，取最保守的值：

- **布尔指令**：只要任意一个源设置了该指令，结果就包含
- **数字指令**：取所有源中的最小值

```typescript
// 核心实现 (headers.server.ts:75-113)
export function getConservativeCacheControl(
    ...cacheControlHeaders: Array<string | null>
): string {
    return format(
        cacheControlHeaders
            .filter(Boolean)
            .map((header) => parse(header))
            .reduce<CacheControlValue>((acc, current) => {
                for (const key in current) {
                    const directive = key as keyof Required<CacheControlValue>
                    const currentValue = current[directive]
                    
                    switch (typeof currentValue) {
                        case 'boolean': {
                            if (currentValue) {
                                acc[directive] = true as any
                            }
                            break
                        }
                        case 'number': {
                            const accValue = acc[directive] as number | undefined
                            if (accValue === undefined) {
                                acc[directive] = currentValue as any
                            } else {
                                // 取最小值
                                const result = Math.min(accValue, currentValue)
                                acc[directive] = result as any
                            }
                            break
                        }
                    }
                }
                return acc
            }, {}),
    )
}
```

#### 2.1.2 各路由的缓存配置

| 路由 | Cache-Control 配置 | 说明 |
|-----|-------------------|------|
| `/sitemap.xml` | `public, max-age=300` | 5 分钟缓存 |
| `/resources/images` | `public, max-age=31536000, immutable` | 1 年，不可变 |
| 其他路由 | 未显式设置 | 继承父路由或使用默认 |

**sitemap.xml 配置示例** (`sitemap[.]xml.ts:10-12`)：
```typescript
headers: {
    'Cache-Control': `public, max-age=${60 * 5}`, // 5分钟
}
```

**图片路由配置示例** (`images.tsx:33`)：
```typescript
headers.set('Cache-Control', 'public, max-age=31536000, immutable')
```

### 2. 服务端应用层缓存

Epic Stack 提供了两层应用级缓存，在 `app/utils/cache.server.ts` 中实现：

#### 2.2.1 SQLite 持久化缓存

- **特性**：独立于主数据库，通过 LiteFS 跨实例复制
- **适用场景**：长生命周期的缓存值
- **表结构**：
  ```sql
  CREATE TABLE IF NOT EXISTS cache (
      key TEXT PRIMARY KEY,
      metadata TEXT,
      value TEXT
  )
  ```

#### 2.2.2 LRU 内存缓存

- **特性**：基于 `lru-cache` 的内存缓存，最大 5000 项
- **适用场景**：短生命周期缓存、请求去重
- **限制**：不跨实例，应用重启后清空

#### 2.2.3 缓存使用模式

通过 `@epic-web/cachified` 库抽象缓存管理：

```typescript
// 典型使用模式 (来自 docs/caching.md)
import { cachified, cache } from '#app/utils/cache.server.ts'

const scheduledEvents = await cachified({
    key: 'tito:scheduled-events',
    cache,  // SQLite 缓存
    timings,
    getFreshValue: () => { /* 获取新鲜值 */ },
    checkValue: eventSchema.array(),
    ttl: 1000 * 60 * 60 * 24,  // 24小时有效
    staleWhileRevalidate: 1000 * 60 * 60 * 24 * 30,  // 30天内可 stale
})
```

---

## 三、响应头设置方式

响应头的设置在 Epic Stack 中分为三个层次：

### 1. 全局入口层 (entry.server.tsx)

在 `entry.server.tsx` 中设置 Fly.io 部署相关的实例信息头：

```typescript
// 文档请求处理 (entry.server.tsx:29-40)
export default async function handleRequest(...args: DocRequestArgs) {
    const [request, responseStatusCode, responseHeaders, reactRouterContext] = args
    const { currentInstance, primaryInstance } = await getInstanceInfo()
    
    // 设置 Fly.io 实例信息头
    responseHeaders.set('fly-region', process.env.FLY_REGION ?? 'unknown')
    responseHeaders.set('fly-app', process.env.FLY_APP_NAME ?? 'unknown')
    responseHeaders.set('fly-primary-instance', primaryInstance)
    responseHeaders.set('fly-instance', currentInstance)
    
    // Sentry 性能分析
    if (process.env.NODE_ENV === 'production' && process.env.SENTRY_DSN) {
        responseHeaders.append('Document-Policy', 'js-profiling')
    }
    // ...
}

// 数据请求处理 (entry.server.tsx:115-123)
export async function handleDataRequest(response: Response) {
    const { currentInstance, primaryInstance } = await getInstanceInfo()
    response.headers.set('fly-region', process.env.FLY_REGION ?? 'unknown')
    response.headers.set('fly-app', process.env.FLY_APP_NAME ?? 'unknown')
    response.headers.set('fly-primary-instance', primaryInstance)
    response.headers.set('fly-instance', currentInstance)
    
    return response
}
```

### 2. 路由层 (root.tsx)

根路由使用 `pipeHeaders` 函数作为 headers 导出：

```typescript
// root.tsx:135
export const headers: Route.HeadersFunction = pipeHeaders
```

### 3. Header 管道处理逻辑

`pipeHeaders` 函数负责处理路由头的合并、继承和回退：

```typescript
// headers.server.ts:12-70
export function pipeHeaders({
    parentHeaders,
    loaderHeaders,
    actionHeaders,
    errorHeaders,
}: HeadersArgs) {
    const headers = new Headers()
    
    // 1. 确定当前使用的 headers (error > loader > action)
    let currentHeaders: Headers
    if (errorHeaders !== undefined) {
        currentHeaders = errorHeaders
    } else if (loaderHeaders.entries().next().done) {
        currentHeaders = actionHeaders
    } else {
        currentHeaders = loaderHeaders
    }
    
    // 2. 转发关键 headers
    const forwardHeaders = ['Cache-Control', 'Vary', 'Server-Timing']
    for (const headerName of forwardHeaders) {
        const header = currentHeaders.get(headerName)
        if (header) {
            headers.set(headerName, header)
        }
    }
    
    // 3. 合并 Cache-Control（取最保守值）
    headers.set(
        'Cache-Control',
        getConservativeCacheControl(
            parentHeaders.get('Cache-Control'),
            headers.get('Cache-Control'),
        ),
    )
    
    // 4. 继承父路由的某些 headers (追加)
    const inheritHeaders = ['Vary', 'Server-Timing']
    for (const headerName of inheritHeaders) {
        const header = parentHeaders.get(headerName)
        if (header) {
            headers.append(headerName, header)
        }
    }
    
    // 5. 回退到父路由的 headers（如果当前没有）
    const fallbackHeaders = ['Cache-Control', 'Vary']
    for (const headerName of fallbackHeaders) {
        if (headers.has(headerName)) {
            continue
        }
        const fallbackHeader = parentHeaders.get(headerName)
        if (fallbackHeader) {
            headers.set(headerName, fallbackHeader)
        }
    }
    
    return headers
}
```

---

## 四、服务端健康检查实现

健康检查路由位于 `app/routes/resources/healthcheck.tsx`，用于 Fly.io 的 HTTP 健康检查（参考 `fly.io/docs/reference/configuration/#services-http_checks`）。

### 1. 实现逻辑

```typescript
// healthcheck.tsx:5-26
export async function loader({ request }: Route.LoaderArgs) {
    const host =
        request.headers.get('X-Forwarded-Host') ?? request.headers.get('host')

    try {
        // 并行执行两个检查
        await Promise.all([
            // 检查 1: 数据库连接
            prisma.user.count(),
            // 检查 2: 自身 HEAD 请求（避免无限循环）
            fetch(`${new URL(request.url).protocol}${host}`, {
                method: 'HEAD',
                headers: { 'X-Healthcheck': 'true' },
            }).then((r) => {
                if (!r.ok) return Promise.reject(r)
            }),
        ])
        return new Response('OK')
    } catch (error: unknown) {
        console.log('healthcheck ❌', { error })
        return new Response('ERROR', { status: 500 })
    }
}
```

### 2. 检查项说明

| 检查项 | 实现方式 | 失败处理 |
|-------|---------|---------|
| 数据库连接 | `prisma.user.count()` | 捕获异常，返回 500 |
| 应用自连通性 | 对自身发起 HEAD 请求（带 `X-Healthcheck: true` 头） | 响应非 2xx 则 reject |

### 3. 设计要点

- **并行检查**：使用 `Promise.all` 同时执行两项检查
- **防止无限循环**：通过 `X-Healthcheck: true` 头标识健康检查请求
- **简洁响应**：成功返回 `"OK"`，失败返回 `"ERROR"` 并设置 500 状态码

---

## 五、缓存边界在请求管线中的位置

### 1. 请求处理流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        客户端请求                                  │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│              entry.server.tsx (全局入口层)                        │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 1. 设置 Fly.io 实例信息头 (fly-region, fly-app 等)        │  │
│  │ 2. 设置 CSP 安全头 (通过 @nichtsam/helmet)                 │  │
│  │ 3. 设置 Server-Timing 头                                   │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                  React Router 路由匹配                            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 1. 匹配路由模块                                             │  │
│  │ 2. 执行 loader/action                                      │  │
│  │ 3. 调用 headers 函数 (pipeHeaders)                         │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                pipeHeaders (路由头处理层)                         │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 优先级: errorHeaders > loaderHeaders > actionHeaders       │  │
│  │                                                             │  │
│  │ 转发 Headers:  Cache-Control, Vary, Server-Timing         │  │
│  │ 继承 Headers:  Vary, Server-Timing (追加)                  │  │
│  │ 回退 Headers:  Cache-Control, Vary (父路由)                │  │
│  │                                                             │  │
│  │ Cache-Control 合并策略: 取所有源的最保守值                  │  │
│  │  - 布尔指令: OR 逻辑 (任一设置则保留)                       │  │
│  │  - 数字指令: MIN 逻辑 (取最小值)                            │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                      响应返回                                      │
│  - 文档请求: HTML 流 + 合并后的 headers                           │
│  - 数据请求: JSON + 合并后的 headers                              │
└─────────────────────────────────────────────────────────────────┘
```

### 2. 缓存边界的关键位置

#### 2.5.1 HTTP 缓存头边界

**设置位置**：
1. **路由 loader/action**：在 `data()` 或直接返回 Response 时设置
2. **headers 函数**：通过 `pipeHeaders` 合并处理
3. **entry.server.tsx**：全局头设置（但不处理 Cache-Control）

**处理优先级**：
```
子路由 loader/action 设置的 Cache-Control
    ↓ 合并（取保守值）
父路由设置的 Cache-Control
    ↓
最终响应的 Cache-Control
```

#### 2.5.2 应用层缓存边界

**调用位置**：在 loader/action 内部调用 `cachified()`

```
请求到达
    ↓
loader/action 执行
    ↓
cachified() 检查缓存
    ├── 命中缓存 → 直接返回缓存值
    └── 未命中 → 执行 getFreshValue() → 存入缓存 → 返回
    ↓
设置响应头 (如果有)
    ↓
headers 函数处理
    ↓
返回响应
```

### 3. 关键代码位置汇总

| 功能 | 文件位置 | 关键函数/导出 |
|-----|---------|--------------|
| Header 管道处理 | `app/utils/headers.server.ts` | `pipeHeaders()`, `getConservativeCacheControl()` |
| 根路由 Header | `app/root.tsx` | `export const headers = pipeHeaders` |
| 全局入口 Header | `app/entry.server.tsx` | `handleRequest()`, `handleDataRequest()` |
| 应用层缓存 | `app/utils/cache.server.ts` | `cache`, `lruCache`, `cachified()` |
| 健康检查 | `app/routes/resources/healthcheck.tsx` | `loader()` |
| Sitemap 缓存 | `app/routes/_seo/sitemap[.]xml.ts` | `headers: { 'Cache-Control': ... }` |
| 图片缓存 | `app/routes/resources/images.tsx` | `headers.set('Cache-Control', ...)` |

---

## 六、总结

### 1. 设计亮点

1. **分层缓存策略**：HTTP 缓存头 + 应用层缓存（SQLite + LRU）的多层次设计
2. **保守缓存合并**：`getConservativeCacheControl` 确保不会意外放松缓存策略
3. **灵活的 Header 管道**：`pipeHeaders` 支持转发、继承、回退三种模式
4. **健康检查完整性**：同时验证数据库和应用连通性

### 2. 缓存策略决策树

```
是否为静态资源？
├── 是 (图片等)
│   └── 使用: public, max-age=31536000, immutable
└── 否
    ├── 是否为 SEO 资源？
    │   ├── 是 (sitemap)
    │   │   └── 使用: public, max-age=300
    │   └── 否
    │       └── 是否需要缓存？
    │           ├── 是 → 使用应用层 cachified
    │           └── 否 → 不设置或使用默认
    └── 是否为动态数据？
        ├── 是 → 通常不设置 Cache-Control (no-store)
        └── 否 → 视情况而定
```

### 3. 注意事项

1. **HTTP 缓存 vs 应用缓存**：HTTP 缓存控制浏览器/CDN 行为，应用缓存控制服务器内部行为
2. **Cache-Control 优先级**：子路由设置不会覆盖父路由，而是取更保守的值
3. **LRU 缓存局限性**：不跨实例，适合短期缓存
4. **SQLite 缓存同步**：非主实例通过调用主实例的 `/admin/cache/sqlite` 端点更新缓存
