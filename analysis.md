# Epic Stack 资源路由与缓存系统分析

## 1. 资源路由处理机制

### 1.1 图片处理路由 (images.tsx)

**核心功能：

```
loader() - 图片资源路由
├── 参数解析：objectKey 或 src 参数
├── 缓存策略：public, max-age=31536000, immutable
├── 来源处理：
│   ├── objectKey → S3 签名 URL (getSignedGetRequestInfo)
│   ├── 外部 URL → 白名单验证后 fetch
│   ├── /assets → Vite 管理的文件系统
│   └── 其他 → public 目录
└── 本地缓存：getCacheDir()
    ├── 开发环境：./tests/fixtures/openimg
    └── 生产环境：/data/images (可写时)
```

**关键特性：**

- 使用 `openimg/node` 库的 `getImgResponse` 处理图片优化和缓存
- 支持 S3 存储的对象通过签名 URL 访问
- 本地缓存目录根据环境动态选择
- 图片来源白名单验证防止 SSRF 攻击

### 1.2 主题切换路由 (theme-switch.tsx)

**核心功能：**

```
action() - POST 请求处理
├── 表单验证：ThemeFormSchema (zod)
│   ├── theme: 'system' | 'light' | 'dark'
│   └── redirectTo: 可选重定向目标
├── Cookie 设置：setTheme()
│   ├── 'system' → 删除 Cookie (maxAge: -1)
│   └── 'light'/'dark' → 设置 Cookie (maxAge: 31536000)
├── 响应处理：
│   ├── 有 redirectTo → 3xx 重定向
│   └── 无 redirectTo → 返回 JSON 数据
└── 乐观更新：useOptimisticThemeMode()
    ├── useFetchers() 监听所有 fetcher
    ├── 解析 formData 中的 theme 参数
    └── 立即应用主题切换视觉
```

**关键特性：**

- 支持渐进式增强 (progressive enhancement)
- 使用 `conform-to` 进行表单验证
- `useFetcher` 实现无刷新提交
- 乐观 UI 更新提升用户体验

### 1.3 用户数据下载路由 (download-user-data.tsx)

**核心功能：**

```
loader() - GET 请求处理
├── 权限验证：requireUserId()
├── 数据查询：prisma.user.findUniqueOrThrow()
│   ├── include:
│   │   ├── image: objectKey 等元数据 (不包含 blob)
│   │   ├── notes: 包含 notes 及 images
│   │   ├── password: false (明确排除)
│   │   ├── sessions: true
│   │   └── roles: true
├── URL 构建：
│   ├── getUserImgSrc() 生成图片访问路径
│   └── getNoteImgSrc() 生成笔记图片访问路径
└── 响应格式：JSON
```

**关键特性：**

- 明确排除敏感字段 (password)
- 图片不直接返回 blob，返回可访问的 URL
- 使用 `getDomainUrl()` 构建完整 URL

## 2. 缓存与存储协作方式

### 2.1 缓存架构

#### 2.1.1 缓存层级

```
缓存系统 (cache.server.ts)
├── LRU 内存缓存 (lruCache)
│   ├── 容量：5000 条目
│   ├── 特性：TTL 过期、start 时间
│   └── 适用：热点数据、频繁访问
└── SQLite 持久化缓存 (cache)
│   ├── 表结构：
│   │   ├── key: TEXT PRIMARY KEY
│   │   ├── metadata: TEXT (JSON)
│   │   └── value: TEXT (JSON)
│   ├── Buffer 序列化：
│   │   ├── 序列化：base64 编码
│   │   └── 反序列化：base64 解码
│   └── 实例同步：
│   │   ├── Primary 实例 → 直接操作
│   │   └── Replica 实例 → 通过 updatePrimaryCacheValue()
│   │       └── 调用 Primary 的 /admin/cache/sqlite
│   │       └── Authorization: Bearer ${INTERNAL_COMMAND_TOKEN
│   └── 操作接口：
│       ├── get(key)
│       ├── set(key, entry)
│       └── delete(key)
└── cachified() 包装函数
    ├── 合并 reporter：verboseReporter + cachifiedTimingReporter
    └── 支持 timings 参数
```

#### 2.1.2 缓存操作流程

```
缓存写入流程：
├── 检查是否为 Primary 实例
│   ├── 是 → 直接写入
│   └── 否 → 异步调用 Primary 的 /admin/cache/sqlite
│       └── 携带 INTERNAL_COMMAND_TOKEN 鉴权

缓存读取流程：
├── LRU 内存缓存 → SQLite 持久化缓存
└── 支持 TTL 和 SWR (stale-while-revalidate)
```

### 2.2 存储系统

#### 2.2.1 S3 兼容存储 (storage.server.ts)

```
存储系统架构：
├── 配置：
│   ├── AWS_ENDPOINT_URL_S3
│   ├── BUCKET_NAME
│   ├── AWS_ACCESS_KEY_ID
│   ├── AWS_SECRET_ACCESS_KEY
│   └── AWS_REGION
├── 上传功能：
│   ├── uploadProfileImage(userId, file)
│   │   └── Key: users/${userId}/profile-images/${timestamp}-${fileId}.${ext}
│   ├── uploadNoteImage(userId, noteId, file)
│   │   └── Key: users/${userId}/notes/${noteId}/images/${timestamp}-${fileId}.${ext}
│   └── uploadToStorage(file, key)
│       ├── 生成签名 PUT 请求
│       └── fetch 上传
├── 签名机制：
│   ├── getSignedPutRequestInfo()
│   │   └── AWS4-HMAC-SHA256
│   │   └── 包含 Content-Type, X-Amz-Meta-Upload-Date
│   └── getSignedGetRequestInfo()
│       └── AWS4-HMAC-SHA256
│       └── 不包含额外元数据
└── 签名算法：
    ├── hmacSha256()
    ├── sha256()
    └── getSignatureKey()
```

#### 2.2.2 缓存与存储协作

```
图片访问流程：
1. 用户请求 /resources/images
2. images.tsx loader 处理
3. 检查 objectKey 参数
   ├── 是 → getSignedGetRequestInfo() 获取 S3 签名 URL
   └── 否 → 从文件系统读取
4. openimg 库处理：
   ├── 检查本地缓存目录
   ├── 本地有缓存 → 直接返回
   └── 本地无缓存 → 从源获取并缓存
5. 设置 HTTP 缓存头
   └── Cache-Control: public, max-age=31536000, immutable
```

## 3. 管理员缓存页面穿透到后端实现

### 3.0 深度分析：副本实例异步转发失败路径与一致性

#### 3.0.1 副本实例写入 SQLite 缓存的完整链路

```
副本实例写入流程 (cache.server.ts:158-197):

1. cache.set(key, entry) 或 cache.delete(key) 被调用
   ↓
2. getInstanceInfo() 获取实例角色
   ├── currentIsPrimary === true → 直接操作本地 SQLite
   │   └── 正常路径：setStatement.run() / deleteStatement.run()
   └── currentIsPrimary === false → 进入异步转发路径
       └── void updatePrimaryCacheValue({ key, cacheValue })
           ├── 这是一个 fire-and-forget (即发即忘) 调用
           ├── 不等待响应，不阻塞主流程
           └── 没有返回值，没有 Promise rejection 处理
```

#### 3.0.2 异步转发的实现细节 (sqlite.server.ts:10-33)

```typescript
export async function updatePrimaryCacheValue({
  key,
  cacheValue,
}: {
  key: string
  cacheValue: any
}) {
  // 防御性检查：确保不在 Primary 实例上调用
  const { currentIsPrimary, primaryInstance } = await getInstanceInfo()
  if (currentIsPrimary) {
    throw new Error(
      `updatePrimaryCacheValue should not be called on the primary instance (${primaryInstance})}`,
    )
  }
  
  // 构建内部请求
  const domain = getInternalInstanceDomain(primaryInstance)
  const token = process.env.INTERNAL_COMMAND_TOKEN
  
  // 发起 fetch 请求，返回 Promise
  return fetch(`${domain}/admin/cache/sqlite`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ key, cacheValue }),
  })
}
```

**调用者处的错误处理 (cache.server.ts:166-176):**

```typescript
void updatePrimaryCacheValue({
  key,
  cacheValue: entry,
}).then((response) => {
  if (!response.ok) {
    console.error(
      `Error updating cache value for key "${key}" on primary instance (${primaryInstance}): ${response.status} ${response.statusText}`,
      { entry },
    )
  }
})
// ⚠️ 注意：这里没有 .catch() 处理 fetch 本身的网络错误！
```

#### 3.0.3 失败路径分类分析

**失败类型 1：fetch 网络错误 (Network Error)**
```
可能原因：
├── 主实例不可达 (网络分区)
├── DNS 解析失败
├── TLS 握手失败
└── 连接超时

结果：
├── fetch Promise 被 reject
├── 但调用者没有 .catch()
├── 产生 Unhandled Promise Rejection
├── Node.js 默认行为：警告日志，高版本可能退出进程
└── 缓存写入完全丢失
```

**失败类型 2：HTTP 非 2xx 响应**
```
可能原因：
├── 内部 Token 验证失败
├── 主实例不是当前实例 (防御性检查)
├── 主实例过载返回 503
└── 请求体解析失败 (Zod 验证失败)

结果：
├── fetch Promise 正常 resolve (response.ok === false)
├── console.error 记录日志
├── 日志内容：key, primaryInstance, status, statusText, entry
└── 缓存写入丢失
```

**失败类型 3：主实例成功写入但 LiteFS 同步延迟**
```
可能原因：
├── 主实例成功写入
├── 副本实例返回前未收到 LiteFS 同步
└── 网络延迟或 LiteFS 复制滞后

结果：
├── fetch 返回 200 OK
├── 主实例已有新缓存值
├── 但副本实例本地 SQLite 仍是旧值
└── 产生短暂的缓存不一致窗口
```

**失败类型 4：主实例写入后进程崩溃**
```
可能原因：
├── 主实例成功写入 SQLite
├── 但返回响应前崩溃
└── LiteFS 可能尚未同步

结果：
├── fetch 可能超时或连接断开
├── 被当作网络错误处理
└── 实际缓存可能已写入，取决于崩溃时机
```

#### 3.0.4 未授权分支行为深度核对 (sqlite.server.ts:35-58)

```typescript
export async function action({ request }: Route.ActionArgs) {
  // 防御性检查 1：必须在 Primary 实例
  const { currentIsPrimary, primaryInstance } = await getInstanceInfo()
  if (!currentIsPrimary) {
    // 未授权路径 A：请求到了 Replica 实例
    // 结果：抛出 Error，返回 500 Internal Server Error
    throw new Error(
      `${request.url} should only be called on the primary instance (${primaryInstance})}`,
    )
  }
  
  // 防御性检查 2：Token 验证
  const token = process.env.INTERNAL_COMMAND_TOKEN
  const isAuthorized =
    request.headers.get('Authorization') === `Bearer ${token}`
  
  if (!isAuthorized) {
    // 未授权路径 B：Token 不匹配或缺失
    // 结果：3xx 重定向到经典 Rickroll
    // 这是一个防御性迷惑措施，不暴露存在该 API
    return redirect('https://www.youtube.com/watch?v=dQw4w9WgXcQ')
  }
  
  // 防御性检查 3：请求体验证
  const { key, cacheValue } = z
    .object({ key: z.string(), cacheValue: z.unknown().optional() })
    .parse(await request.json())
  // 未授权路径 C：请求体格式错误
  // 结果：z.parse() 抛出 ZodError，返回 400/500
  
  // 正常路径：执行缓存操作
  if (cacheValue === undefined) {
    await cache.delete(key)
  } else {
    // @ts-expect-error - we don't reliably know the type of cacheValue
    await cache.set(key, cacheValue)
  }
  return { success: true }
}
```

**未授权路径总结表：**

| 路径 | 触发条件 | HTTP 状态 | 响应内容 | 可观测性 |
|------|---------|-----------|---------|----------|
| 路径 A | 请求到了 Replica | 500 | Error 消息 | 无专门日志 |
| 路径 B | Token 不匹配 | 3xx | Rickroll 重定向 | 无日志 |
| 路径 C | 请求体格式错误 | 400/500 | Zod 错误 | 无专门日志 |

#### 3.0.5 一致性影响分析

**缓存一致性模型：最终一致 (Eventual Consistency)**

```
正常流程：
Replica 写入请求
    ↓
updatePrimaryCacheValue()
    ↓
Primary 本地写入 SQLite
    ↓
LiteFS 异步复制到所有 Replica
    ↓
所有 Replica 读取到新值

延迟窗口：通常毫秒级，取决于网络和 LiteFS 配置
```

**失败场景下的一致性影响：**

| 失败类型 | 写入结果 | 读取一致性 | 影响范围 |
|---------|---------|-----------|---------|
| 网络错误 (Type 1) | 完全丢失 | 所有实例都是旧值 | 全局不一致 |
| HTTP 错误 (Type 2) | 完全丢失 | 所有实例都是旧值 | 全局不一致 |
| 同步延迟 (Type 3) | Primary 成功 | 短暂窗口不一致 | 局部（请求副本的用户）|
| 进程崩溃 (Type 4) | 不确定 | 取决于崩溃时机 | 不确定 |

**cachified 的缓解机制：**

```typescript
// github.server.ts:106-127 中的 TTL/SWR 配置
await cachified({
  key: `connection-data:github:${providerId}`,
  cache,
  ttl: 1000 * 60,           // 60 秒后视为过期
  swr: 1000 * 60 * 60 * 24 * 7,  // 7 天内可返回过期值
  async getFreshValue(context) {
    // ... 重新获取
  },
})
```

**cachified 如何缓解缓存写入失败：**

1. **过期后自动刷新**：TTL 到期后，下一次访问会重新获取
2. **stale-while-revalidate**：过期后可以返回旧值，后台刷新
3. **多副本独立缓存**：每个实例有自己的 SQLite，可独立刷新
4. **L2 缓存**：LRU 内存缓存 + SQLite 持久化缓存，两层都可能需要刷新

**注意：cachified 不能解决的问题**

- 不能"修复"已丢失的缓存写入
- 只能在下次访问时重新计算
- 如果缓存失效成本很高（外部 API 限流、昂贵查询），可能导致性能下降
- 如果大量缓存写入失败，可能引发"缓存雪崩"

#### 3.0.6 可观测性分析

**现有的可观测性措施：**

1. **console.error 日志 (cache.server.ts:171-174)**
   ```
   记录内容：
   ├── cache key
   ├── primary instance 地址
   ├── HTTP status code
   ├── status text
   └── entry (缓存条目内容)
   
   缺失：
   ├── 网络错误 (Unhandled Rejection)
   ├── 成功写入的日志
   ├── 延迟/性能指标
   └── 结构化日志字段
   ```

2. **Sentry 集成 (server/utils/monitoring.ts)**
   ```
   已配置：
   ├── Prisma 数据库查询追踪
   ├── HTTP 请求追踪
   ├── Profiling
   └── 错误捕获
   
   未专门追踪：
   ├── 内部实例间通信 (updatePrimaryCacheValue)
   ├── 缓存写入失败率
   ├── 缓存命中率
   └── LiteFS 同步延迟
   ```

3. **Server Timing 集成 (timing.server.ts:93-121)**
   ```
   cachifiedTimingReporter 追踪：
   ├── cache:${key} - 缓存检索总时间
   └── getFreshValue:${key} - 重新获取数据时间
   
   不追踪：
   ├── 内部实例通信延迟
   ├── cache.set 操作时间
   └── 跨实例转发时间
   ```

**可观测性缺失点：**

| 缺失项 | 影响 | 建议措施 |
|-------|------|---------|
| Unhandled Rejection 捕获 | 网络错误可能丢失 | 添加 .catch() 并上报 Sentry |
| 内部请求指标 | 无法监控实例间通信 | 添加 fetch 超时和重试指标 |
| 缓存写入成功率 | 无法发现系统问题 | 添加 metrics 计数器 |
| 结构化日志 | 难以聚合分析 | 使用结构化日志库 |
| 重试机制 | 临时失败也会丢失 | 添加指数退避重试 |
| 超时控制 | 请求可能无限等待 | 设置 fetch timeout |

#### 3.0.7 完整失败场景时序图

```
场景：副本实例 cache.set() 失败，主实例不可达

时间线：
T0  User 请求到达 Replica 实例
    │
    ├── 业务逻辑执行
    │
    ├── cachified() 生成新缓存值
    │       ↓
    ├── cache.set(key, entry) 被调用
    │       ↓
    ├── getInstanceInfo() → currentIsPrimary = false
    │       ↓
    ├── void updatePrimaryCacheValue(...) 发起异步调用
    │       │
    │       ├── fetch() 发起到 Primary
    │       │       ↓
    │       ├── 网络分区 → fetch Promise reject
    │       │       ↓
    │       ├── 无 .catch() → Unhandled Promise Rejection
    │       │       ↓
    │       ├── Node.js 警告 (可能崩溃)
    │       │
    │       └── ( fire-and-forget, 主流程不等待 )
    │
    ├── 主流程继续，返回响应给用户
    │       ↓
    └── 用户以为操作成功

T1 (后续)
    │
    ├── 用户再次访问
    │       ↓
    ├── cachified 检查缓存
    │       ↓
    ├── 旧值存在但可能已过期
    │       ↓
    ├── TTL 到期 → getFreshValue 重新获取
    │       ↓
    └── 缓存自动恢复 (取决于具体配置)

一致性状态：
├── 所有实例都没有新缓存值
├── 下次访问时可能重新计算
└── 如果外部 API 限流，可能影响体验
```

#### 3.0.8 改进建议

**1. 添加网络错误捕获**
```typescript
// 当前 (cache.server.ts:166-176)
void updatePrimaryCacheValue({...}).then((response) => {
  if (!response.ok) console.error(...)
})
// ❌ 没有 .catch()

// 建议
void updatePrimaryCacheValue({...})
  .then((response) => {
    if (!response.ok) console.error(...)
  })
  .catch((error) => {
    console.error('Network error updating cache:', error)
    // 上报到 Sentry
    Sentry.captureException(error)
  })
```

**2. 添加重试机制**
```typescript
async function updatePrimaryCacheValueWithRetry({ key, cacheValue }) {
  const maxRetries = 3
  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await updatePrimaryCacheValue({ key, cacheValue })
      if (response.ok) return response
      // 5xx 错误重试
      if (response.status >= 500 && i < maxRetries - 1) {
        await delay(1000 * (i + 1)) // 指数退避
        continue
      }
      return response
    } catch (error) {
      if (i === maxRetries - 1) throw error
      await delay(1000 * (i + 1))
    }
  }
}
```

**3. 添加超时控制**
```typescript
// 当前：fetch 无超时
return fetch(url, {...})

// 建议：添加超时
const controller = new AbortController()
const timeoutId = setTimeout(() => controller.abort(), 5000)
try {
  return fetch(url, {
    ...options,
    signal: controller.signal,
  })
} finally {
  clearTimeout(timeoutId)
}
```

**4. 添加结构化指标**
```typescript
// 建议添加
const metrics = {
  cacheWriteSuccess: new Counter(),
  cacheWriteFailure: new Counter(),
  cacheWriteLatency: new Histogram(),
}
```

**5. 考虑使用队列**
```
对于重要的缓存失效：
├── 使用消息队列 (如 BullMQ)
├── 持久化待处理的缓存写入
├── 后台 worker 处理
└── 失败自动重试
```

### 3.1 路由结构

```
/admin/cache/
├── index.tsx (主页面)
│   ├── loader()
│   │   ├── 权限验证：requireUserWithRole(request, 'admin')
│   │   ├── 实例选择：getAllInstances(), ensureInstance()
│   │   └── 缓存查询：
│   │       ├── searchCacheKeys(query, limit)
│   │       └── getAllCacheKeys(limit)
│   ├── action()
│   │   ├── 权限验证：requireUserWithRole(request, 'admin')
│   │   ├── 实例选择：ensureInstance()
│   │   └── 缓存删除：
│   │       ├── 'sqlite' → cache.delete(key)
│   │       └── 'lru' → lruCache.delete(key)
│   └── UI 组件：
│       ├── 搜索表单：query, limit
│       ├── 实例选择下拉框
│       ├── LRU Cache 列表
│       └── SQLite Cache 列表
├── sqlite.server.ts (内部 API)
│   ├── updatePrimaryCacheValue()
│   │   ├── 仅在 Replica 实例调用
│   │   ├── 生成内部请求：
│   │   │   └── POST ${primary}/admin/cache/sqlite
│   │   │   └── Authorization: Bearer ${INTERNAL_COMMAND_TOKEN
│   │   │   └── body: { key, cacheValue }
│   └── action()
│       ├── 仅在 Primary 实例处理
│       ├── 鉴权：INTERNAL_COMMAND_TOKEN
│       └── 操作：
│           ├── cacheValue === undefined → cache.delete(key)
│           └── 其他 → cache.set(key, cacheValue)
├── lru.$cacheKey.ts (LRU 详情)
│   └── loader()
│       ├── 权限验证：requireUserWithRole(request, 'admin')
│       ├── 实例选择：ensureInstance()
│       └── 读取：lruCache.get(cacheKey)
└── sqlite.$cacheKey.ts (SQLite 详情)
    └── loader()
        ├── 权限验证：requireUserWithRole(request, 'admin')
        ├── 实例选择：ensureInstance()
        └── 读取：cache.get(cacheKey)
```

### 3.2 实现边界

#### 3.2.1 权限边界

```
权限验证层级：
├── 管理员路由：
│   └── requireUserWithRole(request, 'admin')
│   └── 所有 admin 角色才能访问
├── 内部 API：
│   └── /admin/cache/sqlite
│   └── INTERNAL_COMMAND_TOKEN 鉴权
│   └── 仅 Primary 实例处理
└── 普通用户：
    └── 无法访问 /admin/* 路由
```

#### 3.2.2 实例边界

```
实例操作边界：
├── LRU 缓存：
│   ├── 每个实例独立的内存缓存
│   ├── 无法跨实例访问
│   └── 只能在目标实例上操作
├── SQLite 缓存：
│   ├── Primary 实例：
│   │   ├── 直接读写
│   │   └── LiteFS 同步到 Replica
│   └── Replica 实例：
│       ├── 只读（通过 LiteFS 同步）
│       └── 写入需要通过 updatePrimaryCacheValue() 转发到 Primary
└── 实例选择：
    ├── UI 层：用户选择目标实例
    └── 路由层：ensureInstance() 确保请求路由到正确实例
```

#### 3.2.3 缓存类型边界

```
缓存类型边界：
├── LRU 内存缓存：
│   ├── 内存中，进程重启丢失
│   ├── 容量限制：5000 条目
│   └── TTL 过期
│   └── 适合热点数据
└── SQLite 持久化缓存：
    ├── 磁盘持久化
    ├── LiteFS 跨实例同步
    └── 适合需要持久化的数据
    └── 支持跨实例共享
```

#### 3.2.4 操作边界

```
操作权限边界：
├── 管理员 UI 操作：
│   ├── 查看缓存键列表
│   ├── 搜索缓存键
│   ├── 查看缓存值详情
│   └── 删除缓存键
├── 内部 API 操作：
│   ├── 设置缓存值
│   └── 删除缓存值
│   └── 仅 INTERNAL_COMMAND_TOKEN 鉴权
└── 普通应用操作：
    ├── 通过 cachified() 包装
    └── 自动处理缓存失效
    └── 无需手动管理
```

## 4. 安全边界总结

| 边界类型 | 实现方式 | 保护机制 |
|---------|---------|--------|
| 权限边界 | requireUserWithRole, INTERNAL_COMMAND_TOKEN | 角色验证 + Token 鉴权 |
| 实例边界 | LiteFS, ensureInstance, updatePrimaryCacheValue | 主从复制 + 请求转发 |
| 缓存类型边界 | LRU 内存 + SQLite 持久化 | 独立存储 + 不同特性 |
| 操作边界 | 不同路由 + 不同权限 | 分层控制 |
| 图片来源边界 | allowlistedOrigins | SSRF 防护 |
| 敏感数据边界 | password: false | 明确排除敏感字段 |

## 5. 关键文件参考

- 资源路由：
  - `app/routes/resources/images.tsx` - 图片处理
  - `app/routes/resources/theme-switch.tsx` - 主题切换
  - `app/routes/resources/download-user-data.tsx` - 数据下载

- 缓存系统：
  - `app/utils/cache.server.ts` - 缓存实现
  - `app/utils/storage.server.ts` - 存储实现
  - `app/utils/theme.server.ts` - 主题 Cookie

- 管理员缓存：
  - `app/routes/admin/cache/index.tsx` - 缓存管理主页面
  - `app/routes/admin/cache/sqlite.server.ts` - 内部缓存 API
  - `app/routes/admin/cache/lru.$cacheKey.ts` - LRU 详情
  - `app/routes/admin/cache/sqlite.$cacheKey.ts` - SQLite 详情
  - `app/utils/litefs.server.ts` - LiteFS 集成
