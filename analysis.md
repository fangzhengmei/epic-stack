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

#### 3.0.5 Token 缺失 vs Token 不匹配：两种场景的关键区别

##### 3.0.5.1 场景 A：INTERNAL_COMMAND_TOKEN 缺失（启动期失败）

**前置条件：**
- 某个实例的环境变量 `INTERNAL_COMMAND_TOKEN` 未设置或为空
- 该实例可能是 Primary 或 Replica

**失败路径：**

```
启动期检查 (entry.server.tsx:22):
├── init() 被调用
│       ↓
├── env.server.ts:38-47 执行 Zod 验证
│       ├── schema.safeParse(process.env)
│       └── INTERNAL_COMMAND_TOKEN: z.string() (非 optional)
│       ↓
├── parsed.success === false
│       ↓
├── console.error 输出错误：
│   │
│   └── ❌ Invalid environment variables:
│       └── INTERNAL_COMMAND_TOKEN: [ 'Required' ]
│       ↓
└── throw new Error('Invalid environment variables')
    │
    └── 进程崩溃退出，应用无法启动
```

**关键特点：**
- **快速失败 (Fail Fast)**：启动时立即检测到问题
- **高可观测性**：明确的错误日志指向问题
- **影响范围**：单个实例无法启动，其他实例可能正常
- **故障模式**：实例级别的停机，而非功能级别的静默失效

**观测信号（调用侧/运维侧）：**

| 观测维度 | 信号 | 可操作性 |
|---------|------|---------|
| 进程状态 | 实例进程不存在或不断重启 | ✅ 立即可见 |
| 容器日志 | 明确的 `Invalid environment variables` | ✅ 立即可定位 |
| 健康检查 | `/resources/healthcheck` 无响应 | ✅ 告警触发 |
| 部署流水线 | 新实例无法通过健康检查 | ✅ 回滚触发 |

##### 3.0.5.2 场景 B：INTERNAL_COMMAND_TOKEN 不匹配（运行期失败）

**前置条件：**
- 所有实例的 `INTERNAL_COMMAND_TOKEN` 都已设置（启动期验证通过）
- 但至少两个实例的值不同
- 例如：Replica A = "token-a"，Primary = "token-b"

**失败路径（运行时）：**

```
时间线：
T0 所有实例成功启动（Token 格式正确，但值不同）
    │
T1 Replica 发起 cache.set(key, entry) 调用
    │
    ├── getInstanceInfo() → currentIsPrimary = false
    │       ↓
    ├── void updatePrimaryCacheValue({ key, cacheValue: entry })
    │       ↓
    ├── 构建请求头：
    │   └── Authorization: `Bearer ${Replica.Token}`  // "token-a"
    │       ↓
    ├── fetch 发起到 Primary 的 /admin/cache/sqlite
    │       ↓
    ├── Primary 接收请求，执行 Token 验证：
    │   │
    │   ├── const token = process.env.INTERNAL_COMMAND_TOKEN  // "token-b"
    │   ├── const isAuthorized = request.headers.get('Authorization') === `Bearer ${token}`
    │   └── isAuthorized = false  // "token-a" !== "token-b"
    │       ↓
    ├── Primary 返回 302 重定向：
    │   │
    │   └── return redirect('https://www.youtube.com/watch?v=dQw4w9WgXcQ')
    │       │
    │       ├── HTTP/1.1 302 Found
    │       └── Location: https://www.youtube.com/watch?v=dQw4w9WgXcQ
    │       ↓
    ├── Replica 侧 fetch 默认行为：
    │   │
    │   ├── redirect: 'follow'（默认配置）
    │   ├── 自动跟随重定向
    │   └── 发起新请求到 youtube.com
    │       ↓
    ├── YouTube 返回 200 OK
    │       ↓
    └── fetch 返回 Response：
        ├── response.ok = true ✅
        ├── response.status = 200
        ├── response.statusText = 'OK'
        └── response.body = YouTube HTML

调用侧判定：
.then((response) => {
  if (!response.ok) {  // response.ok === true，不进入此分支
    console.error(...)  // ❌ 不会执行！
  }
})
// 🔥 结果：缓存写入失败，但没有任何日志！完全静默！
```

**关键特点：**
- **延迟失败 (Silent Failure)**：启动期正常，运行时静默失效
- **低可观测性**：没有任何错误迹象
- **影响范围**：功能级别的失效，实例仍在运行
- **故障模式**：缓存逐渐过期，性能逐渐下降

##### 3.0.5.3 fetch 重定向行为的深度分析

**Node.js fetch 默认配置：**

```
fetch(url, options) 默认行为：
├── redirect: 'follow'
│   ├── 自动跟随所有 3xx 响应
│   ├── 最多跟随 20 次重定向（浏览器兼容）
│   └── 返回最终响应的 Response 对象
│
├── response.ok 的定义：
│   └── response.status 在 200-299 范围内
│
└── 3xx 状态码在跟随重定向后：
    ├── 原始 3xx 响应被"消费"
    ├── 最终响应的状态码决定 response.ok
    └── 无法通过 response.ok 检测是否发生过重定向
```

**Token 不匹配时的状态码链：**

```
请求链：
├── 第 1 次请求：POST /admin/cache/sqlite
│   └── 响应：302 Found (重定向到 YouTube)
│
├── 第 2 次请求：GET youtube.com/watch?v=...
│   └── 响应：可能是 302/303 (Google 登录重定向)
│
├── 第 N 次请求：最终页面
│   └── 响应：200 OK
│
└── fetch 返回的 Response：
    ├── response.ok = true (200 在 2xx 范围内)
    ├── response.status = 200
    ├── response.url = 最终 URL (可能是 accounts.google.com 或 youtube.com)
    └── 原始 302 响应的信息完全丢失！
```

##### 3.0.5.4 React Router redirect() 的行为细节

```
React Router redirect() 函数：
├── 返回值类型：TypedResponse
├── 默认状态码：302 Found
├── 响应头：
│   ├── Location: <目标 URL>
│   └── 其他标准响应头
└── 在 Remix/React Router 路由中：
    ├── loader/action 返回 redirect
    ├── 框架处理为 HTTP 3xx 响应
    └── 不执行后续代码
```

**代码位置**：`sqlite.server.ts:47`
```typescript
return redirect('https://www.youtube.com/watch?v=dQw4w9WgXcQ')
// 等价于：
// return new Response(null, {
//   status: 302,
//   headers: { Location: 'https://www.youtube.com/watch?v=dQw4w9WgXcQ' }
// })
```

##### 3.0.5.5 统一口径后的失败场景矩阵

**统一口径定义：**
- **Token 缺失** = `INTERNAL_COMMAND_TOKEN` 环境变量未设置或为空字符串 → 启动期失败（快速失败）
- **Token 不匹配** = `INTERNAL_COMMAND_TOKEN` 环境变量存在，但至少两个实例的值不同 → 运行期静默失败

| 场景编号 | 前置条件 | 失败阶段 | 失败类型 | fetch 行为 | response.ok | 日志记录 | 可观测性 | 实际结果 |
|---------|---------|---------|---------|-----------|------------|---------|---------|---------|
| A1 | Token 未设置 | 启动期 | 环境变量缺失 | N/A (未启动) | N/A | ✅ 明确错误日志 | ✅ 高（进程退出） | **快速失败** |
| A2 | Token 为空字符串 | 启动期 | 环境变量验证失败 | N/A (未启动) | N/A | ✅ 明确错误日志 | ✅ 高（进程退出） | **快速失败** |
| B1 | Token 值不匹配 | 运行期 | Token 字符串比较失败 | 跟随重定向 → YouTube 200 | ✅ true | ❌ 无日志 | ❌ 低（完全静默） | **静默失败** 🔥 |
| B2 | 请求到 Replica 实例 | 运行期 | 实例角色检查失败 | 不跟随重定向 (500) | ❌ false | ✅ console.error | ✅ 中（有日志） | **有日志的失败** |
| B3 | 请求体格式错误 | 运行期 | Zod 验证失败 | 不跟随重定向 (400/500) | ❌ false | ✅ console.error | ✅ 中（有日志） | **有日志的失败** |
| B4 | 网络分区/超时 | 运行期 | 网络错误 | Promise reject | N/A | ❌ 无 .catch() | ⚠️ 中 (Unhandled Rejection) | **静默失败** 🔥 |
| B5 | DNS 解析失败 | 运行期 | 网络错误 | Promise reject | N/A | ❌ 无 .catch() | ⚠️ 中 (Unhandled Rejection) | **静默失败** 🔥 |

**注：** "Token 格式错误（缺少 'Bearer '）"属于代码 bug 而非 Token 配置问题。当前代码中 `Authorization: \`Bearer ${token}\`` 是硬编码构建的，此场景不会发生。

##### 3.0.5.6 调用侧如何判别两类问题的观测信号

**场景 A（启动期失败）的观测信号：**

```
可观测性：高
├── 直接信号：
│   ├── 容器/进程状态：CrashLoopBackOff 或 ExitCode != 0
│   ├── 启动日志：明确的 "Invalid environment variables"
│   ├── 环境变量验证失败的字段列表
│   └── 堆栈跟踪指向 env.server.ts:init()
│
├── 间接信号：
│   ├── 健康检查端点无响应
│   ├── 负载均衡器持续剔除该实例
│   └── 部署失败/回滚触发
│
└── 定位难度：低
    └── 日志直接指出缺失的环境变量名
```

**场景 B（运行期失败）的观测信号：**

```
可观测性：极低（几乎不可见）
├── 直接信号：
│   ├── ❌ 无错误日志
│   ├── ❌ 无异常抛出
│   ├── ❌ response.ok = true（误导）
│   └── ⚠️ 可能通过 response.url 检测（包含 youtube.com）
│
├── 间接信号（需要推断）：
│   ├── 缓存命中率逐渐下降
│   ├── 外部 API 调用率异常升高
│   ├── 响应时间逐渐变长
│   ├── 特定实例的负载高于其他实例
│   └── 最终可能触发限流或费用告警
│
└── 定位难度：极高
    ├── 需要通过"症状"反推"病因"
    ├── 容易误判为"性能问题"或"缓存策略问题"
```

**关键区别总结表：**

| 维度 | 场景 A：Token 缺失 | 场景 B：Token 不匹配 |
|-----|-----------------|-------------------|
| **失败阶段** | 启动期 | 运行期 |
| **应用状态** | 进程崩溃退出 | 进程正常运行 |
| **可观测性** | 高（明确错误日志） | 极低（完全静默） |
| **response.ok** | N/A | ✅ true（误导） |
| **日志记录** | 启动失败日志 | ❌ 无日志 |
| **定位难度** | 低（日志直接指出问题） | 极高（需要通过症状推断） |
| **影响范围** | 实例级停机 | 功能级失效 |
| **发现时机** | 立即（启动时） | 延迟（缓存过期后） |
| **故障模式** | Fail Fast（快速失败） | Silent Failure（静默失败） |

##### 3.0.5.7 如何"修复"重定向检测

**方案 1：设置 redirect: 'manual' 或 'error'**

```typescript
// 不跟随重定向，直接返回 3xx 响应
return fetch(`${domain}/admin/cache/sqlite`, {
  method: 'POST',
  redirect: 'manual',  // 或 'error'
  headers: {
    Authorization: `Bearer ${token}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ key, cacheValue }),
})

// 此时：
// Token 失败 → response.status = 302
// response.ok = false (302 不在 200-299)
// if (!response.ok) 分支会执行 ✅
```

**方案 2：检查响应内容类型或 URL**

```typescript
// 跟随重定向后检查最终 URL
.then((response) => {
  const finalUrl = response.url
  if (finalUrl.includes('youtube.com')) {
    console.error('Token validation failed, redirected to YouTube')
  }
  if (!response.ok) {
    console.error(...)
  }
})
```

**方案 3：服务器端返回 401 而非重定向**

```typescript
// 当前 (sqlite.server.ts:47)
return redirect('https://www.youtube.com/watch?v=dQw4w9WgXcQ')

// 建议：内部 API 应该返回 401 Unauthorized
if (!isAuthorized) {
  return new Response(JSON.stringify({ error: 'Unauthorized' }), {
    status: 401,
    headers: { 'Content-Type': 'application/json' },
  })
}
// 此时 response.status = 401, response.ok = false
// 日志会正确记录 ✅
```

#### 3.0.6 无 .catch 场景下的错误可观测性缺口

##### 3.0.6.1 void Promise 的风险

```typescript
// 当前代码 (cache.server.ts:166)
void updatePrimaryCacheValue({...})
  .then((response) => {
    if (!response.ok) console.error(...)
  })
// ❌ 没有 .catch()
```

**void 关键字的作用：**
- 显式表示"忽略 Promise 返回值"
- 不等待 Promise 完成
- 不阻塞主流程

**风险：**

| Promise 状态 | 结果 | 可观测性 |
|------------|------|---------|
| resolve (response.ok = true) | 正常或静默失败 | ❌ 无日志 |
| resolve (response.ok = false) | 记录 console.error | ✅ 有日志 |
| reject (网络错误等) | Unhandled Promise Rejection | ⚠️ 取决于 Node.js 配置 |

##### 3.0.6.2 Unhandled Promise Rejection 的实际行为

**Node.js 默认行为 (版本 22.x)：**

```
Unhandled Promise Rejection 触发时：
├── 输出警告到 stderr
├── 格式：
│   ├── (node:12345) UnhandledPromiseRejectionWarning
│   ├── (node:12345) [DEP0018] DeprecationWarning
│   └── Stack trace
├── 进程默认行为：
│   ├── Node.js 15+：进程退出 (exit code 1)
│   └── 旧版本：继续运行，但有警告
└── 问题：
    ├── 日志格式不统一
    ├── 没有上下文信息 (cache key, primary instance)
    └── Sentry 可能不会捕获 (取决于集成)
```

##### 3.0.6.3 可观测性缺口矩阵

| 错误类型 | 触发场景 | console.error | Sentry 捕获 | 进程影响 |
|---------|---------|--------------|------------|---------|
| Token 不匹配 | env 变量不一致 | ❌ 无 | ❌ 无 | 无 |
| Token 缺失 | INTERNAL_COMMAND_TOKEN 未设置 | ❌ 无 | ❌ 无 | 无 |
| 网络分区 | Primary 不可达 | ❌ 无 | ⚠️ 可能 | 可能退出 |
| DNS 失败 | 内部域名解析失败 | ❌ 无 | ⚠️ 可能 | 可能退出 |
| TLS 握手失败 | 证书问题 | ❌ 无 | ⚠️ 可能 | 可能退出 |
| 连接超时 | 慢网络或负载高 | ❌ 无 | ⚠️ 可能 | 可能退出 |
| HTTP 4xx/5xx | 服务器返回错误 | ✅ 有 | ❌ 无 | 无 |

##### 3.0.6.4 最危险的场景：Token 不一致

**场景描述：**

```
假设：
├── Replica A: INTERNAL_COMMAND_TOKEN = "token-a"
├── Replica B: INTERNAL_COMMAND_TOKEN = "token-b"  ⚠️ 错误！
└── Primary:   INTERNAL_COMMAND_TOKEN = "token-a"

结果：
├── Replica A 写入 → Token 匹配 → 正常工作
├── Replica B 写入 → Token 不匹配 → 重定向到 YouTube
│       ├── fetch 跟随重定向
│       ├── response.ok = true
│       ├── 无日志
│       └── 缓存写入丢失
└── 影响范围：
    ├── Replica B 的所有 cache.set/delete 都失败
    ├── 没有任何错误迹象
    ├── 缓存逐渐过期
    └── 最后导致缓存雪崩
```

**检测难度：**

```
如何发现这个问题？
├── 用户侧：可能只是觉得"应用变慢了"
├── 监控侧：
│   ├── 没有专门的缓存写入成功率指标
│   ├── 没有 Token 验证失败日志
│   └── 可能在很久后才发现外部 API 调用率异常
└── 日志侧：
    ├── 没有 console.error
    ├── 可能有 Unhandled Rejection（如果是网络问题）
    └── 但 Token 问题完全静默
```

#### 3.0.7 边界结论

##### 3.0.7.1 静默失败边界

| 静默失败类型 | 触发条件 | 可见性 | 风险等级 |
|-------------|---------|--------|---------|
| Token 不匹配 | env 变量不一致 | ❌ 完全不可见 | 🔴 高 |
| Token 缺失 | INTERNAL_COMMAND_TOKEN 未设置 | ❌ 完全不可见 | 🔴 高 |
| 网络分区 | Primary 不可达 | ⚠️ 部分可见 (Unhandled Rejection) | 🟠 中 |
| 同步延迟 | LiteFS 复制滞后 | ❌ 不可检测 | 🟡 低 |

##### 3.0.7.2 可观测性边界

**当前可观测性覆盖：**

```
✅ 已覆盖：
├── HTTP 4xx/5xx 响应 → console.error
└── Unhandled Rejection → Node.js 警告 (可能退出)

❌ 未覆盖：
├── Token 验证失败 (重定向后 response.ok = true)
├── 成功写入 (可选采样日志)
├── 缓存写入延迟/耗时
├── 缓存写入成功率指标
├── 结构化日志字段
└── Sentry 事件捕获
```

##### 3.0.7.3 设计意图 vs 实际行为

| 设计意图 | 实际行为 | 偏差 |
|---------|---------|------|
| Token 失败时迷惑攻击者 (Rickroll) | 同时迷惑了自己的监控 | ⚠️ 安全/可观测性权衡 |
| fire-and-forget 不阻塞主流程 | 同时丢失了错误检测 | ⚠️ 性能/可靠性权衡 |
| redirect() 是防御性迷惑 | fetch 跟随重定向破坏了检测 | 🔴 严重缺陷 |

##### 3.0.7.4 修复优先级建议

| 优先级 | 问题 | 修复方案 | 影响范围 |
|-------|------|---------|---------|
| 🔴 P0 | Token 失败静默 | 内部 API 返回 401 而非重定向 | 所有副本写入 |
| 🔴 P0 | 无 .catch() | 添加 .catch() + Sentry | 网络错误 |
| 🟠 P1 | 无超时 | 添加 fetch timeout | 所有请求 |
| 🟠 P1 | 无重试 | 添加指数退避重试 | 临时故障 |
| 🟡 P2 | 无指标 | 添加计数器/直方图 | 监控告警 |
| 🟡 P2 | 无结构化日志 | 使用 pino/winston | 日志聚合 |

#### 3.0.8 一致性影响分析

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

## 5. 深度分析总结

### 5.1 副本实例写入失败路径总结

| 失败类型 | 触发条件 | 写入结果 | 可观测性 | 缓解建议 |
|---------|---------|---------|---------|---------|
| 网络错误 | 主实例不可达 | 完全丢失 | ❌ 无捕获 | 添加 .catch() + Sentry |
| HTTP 错误 | 4xx/5xx 响应 | 完全丢失 | ✅ console.error | 重试机制 |
| 同步延迟 | LiteFS 复制滞后 | Primary 成功 | ❌ 无法检测 | 可接受最终一致 |
| 进程崩溃 | 主实例崩溃 | 不确定 | ❌ 难以追踪 | 分布式事务/队列 |

### 5.2 内部 API 未授权分支行为

| 路径 | 防御检查 | HTTP 状态 | 响应内容 | 设计意图 |
|------|---------|-----------|---------|---------|
| 路径 A | 实例角色检查 | 500 | Error 消息 | 防止错误路由 |
| 路径 B | Token 验证 | 3xx | Rickroll 重定向 | 迷惑攻击者 |
| 路径 C | Zod 验证 | 400/500 | Zod 错误 | 请求体验证 |

### 5.3 一致性模型分析

```
一致性级别：最终一致 (Eventual Consistency)
├── 正常延迟：毫秒级 (LiteFS 复制)
├── 失败恢复：TTL/SWR 自动刷新
├── 风险点：
│   ├── 大量写入失败 → 缓存雪崩
│   ├── 外部 API 限流 → 刷新失败
│   └── 长 TTL + 无重试 → 长期不一致
└── 缓解措施：
    ├── 合理设置 TTL (如 60s)
    ├── 使用 SWR 容忍过期值
    └── 缓存写入失败重试机制
```

### 5.4 可观测性矩阵

| 监控维度 | 现有状态 | 缺失项 | 优先级 |
|---------|---------|--------|-------|
| Token 验证失败 | ❌ 完全静默 | 返回 401 + 日志 | 🔴 高 |
| HTTP 4xx/5xx 失败 | ✅ console.error | 结构化字段 | 中 |
| 网络失败 | ❌ 未捕获 | .catch() + Sentry | 🔴 高 |
| 成功写入 | ❌ 无日志 | 可选采样日志 | 低 |
| 延迟指标 | ❌ 无监控 | Histogram | 中 |
| 失败率 | ❌ 无指标 | Counter + 告警 | 🔴 高 |
| LiteFS 同步 | ❌ 无监控 | 需要 LiteFS 指标 | 中 |

### 5.5 核心边界结论

#### 5.5.1 最严重的静默失败点

```
🔴 致命缺陷：Token 验证失败导致完全静默

根本原因：
1. 服务端返回 redirect('youtube.com') → 302 Found
2. Node.js fetch 默认 redirect: 'follow' → 自动跟随重定向
3. YouTube 返回 200 OK → response.ok = true
4. 调用侧只有 if (!response.ok) 才记录日志
5. 结果：缓存写入失败，但没有任何迹象！

影响链：
env 变量不一致
    ↓
Token 验证失败
    ↓
重定向 → YouTube 200
    ↓
response.ok = true
    ↓
无日志记录
    ↓
缓存写入丢失
    ↓
缓存逐渐过期
    ↓
外部 API 调用激增
    ↓
可能触发限流或费用超支
```

#### 5.5.2 fetch 默认行为的反直觉陷阱

| fetch 配置 | 3xx 响应行为 | response.ok | 调用侧检测 |
|-----------|-------------|------------|----------|
| `redirect: 'follow'` (默认) | 跟随到最终 URL | 取决于最终响应 | ❌ 无法检测重定向 |
| `redirect: 'manual'` | 返回原始 3xx 响应 | false (3xx 非 2xx) | ✅ 可以检测 |
| `redirect: 'error'` | Promise reject | N/A | ✅ 通过异常检测 |

**当前代码使用默认值 `'follow'`，这是静默失败的根源。**

#### 5.5.3 可观测性的三层边界

```
可观测性层级：
├── 第一层：显式日志
│   ├── HTTP 4xx/5xx → console.error ✅
│   └── Token 失败 → 无日志 ❌
│
├── 第二层：Unhandled Rejection
│   ├── 网络错误 → Node.js 警告 ⚠️
│   └── Token 失败 → Promise resolve，无异常 ❌
│
└── 第三层：外部监控
    ├── 缓存写入成功率 → 无指标 ❌
    ├── Token 验证失败率 → 无指标 ❌
    └── 只能通过"症状"间接推断：
        ├── 外部 API 调用率异常
        ├── 响应时间变长
        └── 缓存命中率下降
```

#### 5.5.4 设计权衡的边界

| 设计决策 | 优点 | 缺点 | 边界 |
|---------|------|------|------|
| `redirect(youtube)` | 迷惑外部攻击者 | 内部调用无法检测错误 | 🔴 内部 API 不应使用重定向 |
| `void` 即发即忘 | 不阻塞主流程 | 丢失错误上下文 | ⚠️ 需要 .catch() |
| `redirect: 'follow'` | 浏览器友好 | 隐藏了 3xx 状态 | ⚠️ 内部 API 调用应使用 'error' 或 'manual' |
| `response.ok` 判定 | 简单直接 | 无法区分成功与"跟随重定向后的成功" | 🔴 需要额外检查 |

#### 5.6.5 最小修复建议

**不改变设计意图的最小修复：**

```typescript
// 方案 A：修改 fetch 配置（最小改动）
return fetch(`${domain}/admin/cache/sqlite`, {
  method: 'POST',
  redirect: 'error',  // 3xx 响应直接 reject
  headers: {
    Authorization: `Bearer ${token}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ key, cacheValue }),
})

// 同时添加 .catch()
void updatePrimaryCacheValue({...})
  .then((response) => {
    if (!response.ok) {
      console.error(...)
    }
  })
  .catch((error) => {
    console.error('Cache update failed:', error)
    // Sentry.captureException(error)
  })
```

**更彻底的修复（推荐）：**

```typescript
// 服务端：内部 API 返回 401 而非重定向
if (!isAuthorized) {
  return new Response(JSON.stringify({ error: 'Unauthorized' }), {
    status: 401,
    headers: { 'Content-Type': 'application/json' },
  })
}
// 如果需要迷惑外部攻击者，可以保留重定向但添加 header 检查：
// if (request.headers.get('X-Internal-Call') !== 'true') {
//   return redirect(...)
// }
```

## 6. 关键文件参考

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
