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
