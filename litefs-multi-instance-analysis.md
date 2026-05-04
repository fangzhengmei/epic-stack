# LiteFS 多实例架构分析

## 1. 应用如何读取当前实例和 Primary 信息

### 核心机制

应用通过 `litefs-js` 库提供的 API 来获取实例信息。相关代码位于 `app/utils/litefs.server.ts`。

### 导出的 API

```typescript
// 从 litefs-js 导出
export {
	getInstanceInfo,        // 异步获取实例信息
	getInstanceInfoSync,    // 同步获取实例信息
	getInternalInstanceDomain,  // 获取实例内部域名
	getAllInstances,        // 获取所有实例列表
	// ... 其他事务一致性相关 API
} from 'litefs-js'

// 从 litefs-js/remix 导出
export {
	ensurePrimary,      // 确保在主实例执行
	ensureInstance,     // 确保在指定实例执行
	// ... 其他 Remix 集成 API
} from 'litefs-js/remix'
```

### 返回的实例信息结构

调用 `getInstanceInfo()` 或 `getInstanceInfoSync()` 会返回包含以下关键信息的对象：

- **`currentIsPrimary`**: `boolean` - 当前实例是否是 Primary（主实例）
- **`currentInstance`**: `string` - 当前实例的主机名/标识
- **`primaryInstance`**: `string` - Primary 实例的主机名/标识

### 使用示例

```typescript
// 同步获取（在数据库初始化时使用）
const { currentIsPrimary } = getInstanceInfoSync()

// 异步获取（在请求处理时使用）
const { currentIsPrimary, primaryInstance } = await getInstanceInfo()
```

### 实际使用场景

1. **数据库初始化** (`cache.server.ts:36`):
   ```typescript
   const { currentIsPrimary } = getInstanceInfoSync()
   if (!currentIsPrimary) return db  // 非主实例不执行表创建
   ```

2. **缓存操作判断** (`cache.server.ts:159`, `cache.server.ts:180`):
   ```typescript
   const { currentIsPrimary, primaryInstance } = await getInstanceInfo()
   if (currentIsPrimary) {
       // 直接执行数据库操作
   } else {
       // 转发到主实例
   }
   ```

3. **管理路由实例切换** (`sqlite.$cacheKey.ts:14-18`):
   ```typescript
   const currentInstanceInfo = await getInstanceInfo()
   const allInstances = await getAllInstances()
   const instance = searchParams.get('instance') ?? currentInstanceInfo.currentInstance
   await ensureInstance(instance)
   ```

---

## 2. OAuth 回调如何确保跑在主实例

### 问题背景

在 LiteFS 多实例架构中：
- **Primary 实例**: 可写，负责处理所有写入操作
- **Replica 实例**: 只读，只能处理读取操作
- SQLite 数据库通过 LiteFS 自动同步，但写入必须在 Primary 实例执行

OAuth 回调流程涉及：
1. 数据库查询（读取现有连接）
2. 创建新连接（写入）
3. 创建新会话（写入）
4. 关联用户账户（写入）

这些写入操作**必须**在 Primary 实例执行，否则会失败。

### 解决方案：`ensurePrimary()`

OAuth 回调路由 `app/routes/_auth/auth.$provider/callback.ts` 在 loader 函数的**第一行**调用 `ensurePrimary()`：

```typescript
export async function loader({ request, params }: Route.LoaderArgs) {
	// this loader performs mutations, so we need to make sure we're on the
	// primary instance to avoid writing to a read-only replica
	await ensurePrimary()
	// ... 后续的 OAuth 处理逻辑
}
```

### `ensurePrimary()` 的工作原理

`ensurePrimary()` 是 `litefs-js/remix` 提供的函数，其行为如下：

1. **检查当前实例状态**：调用 `getInstanceInfo()` 获取 `currentIsPrimary`
2. **如果已是 Primary**：什么都不做，继续执行后续代码
3. **如果不是 Primary**：
   - 获取 `primaryInstance` 的主机名
   - 通过 `getInternalInstanceDomain(primaryInstance)` 获取主实例的内部域名
   - 返回一个 **Replay Response**（重放响应）
   - Fly.io 的代理会识别这个响应，并将原始请求**透明地转发**到 Primary 实例执行

### 透明转发机制

这个机制的关键在于：
- 对 OAuth 提供商（如 GitHub、Google）来说，回调 URL 是固定的
- 用户/浏览器不会感知到任何重定向（没有 302 状态码）
- Fly.io 内部代理处理实例间的请求转发
- 最终响应由 Primary 实例生成并返回

### 为什么必须在 Primary 执行

从回调代码可以看到大量的写入操作：

```typescript
// 1. 创建新连接
await prisma.connection.create({
	data: {
		providerName,
		providerId: String(profile.id),
		userId,
	},
})

// 2. 创建新会话
const session = await prisma.session.create({
	select: { id: true, expirationDate: true, userId: true },
	data: {
		expirationDate: getSessionExpirationDate(),
		userId,
	},
})
```

这些操作如果在 Replica 实例执行会导致：
- SQLite 写入失败（数据库处于只读模式）
- 或数据不一致（写入不会同步到其他实例）

---

## 3. SQLite Cache 如何把更新转发到 Primary

### 问题背景

应用使用了两层缓存：
1. **LRU Cache**: 内存缓存，每个实例独立
2. **SQLite Cache**: 持久化缓存，通过 LiteFS 同步

SQLite Cache 的写入问题：
- 任意实例都可能触发缓存更新（`cache.set()` 或 `cache.delete()`）
- 但只有 Primary 实例可以写入 SQLite 数据库
- Replica 实例的写入需要**转发**到 Primary 实例执行

### 核心实现：`cache.server.ts`

缓存的 `set` 和 `delete` 方法实现了自动转发逻辑：

#### `cache.set()` 方法

```typescript
async set(key, entry) {
	const { currentIsPrimary, primaryInstance } = await getInstanceInfo()

	if (currentIsPrimary) {
		// 主实例：直接写入数据库
		const value = JSON.stringify(entry.value, bufferReplacer)
		setStatement.run(key, value, JSON.stringify(entry.metadata))
	} else {
		// 副本实例：转发到主实例（fire-and-forget）
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
	}
}
```

#### `cache.delete()` 方法

```typescript
async delete(key) {
	const { currentIsPrimary, primaryInstance } = await getInstanceInfo()

	if (currentIsPrimary) {
		// 主实例：直接删除
		deleteStatement.run(key)
	} else {
		// 副本实例：转发到主实例
		void updatePrimaryCacheValue({
			key,
			cacheValue: undefined,  // undefined 表示删除
		}).then((response) => {
			if (!response.ok) {
				console.error(
					`Error deleting cache value for key "${key}" on primary instance (${primaryInstance}): ${response.status} ${response.statusText}`,
				)
			}
		})
	}
}
```

### 转发机制：`updatePrimaryCacheValue()`

转发逻辑实现在 `app/routes/admin/cache/sqlite.server.ts`：

```typescript
export async function updatePrimaryCacheValue({
	key,
	cacheValue,
}: {
	key: string
	cacheValue: any
}) {
	const { currentIsPrimary, primaryInstance } = await getInstanceInfo()
	
	// 安全检查：不应该在 Primary 实例调用此函数
	if (currentIsPrimary) {
		throw new Error(
			`updatePrimaryCacheValue should not be called on the primary instance (${primaryInstance})}`,
		)
	}
	
	// 构建 Primary 实例的内部域名
	const domain = getInternalInstanceDomain(primaryInstance)
	const token = process.env.INTERNAL_COMMAND_TOKEN
	
	// 发送 HTTP POST 请求到 Primary 实例的管理端点
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

### Primary 实例的接收端点

Primary 实例通过 `action` 函数接收并处理转发的请求：

```typescript
export async function action({ request }: Route.ActionArgs) {
	const { currentIsPrimary, primaryInstance } = await getInstanceInfo()
	
	// 安全检查：只允许在 Primary 实例调用
	if (!currentIsPrimary) {
		throw new Error(
			`${request.url} should only be called on the primary instance (${primaryInstance})}`,
		)
	}
	
	// 认证验证：使用内部命令 token
	const token = process.env.INTERNAL_COMMAND_TOKEN
	const isAuthorized =
		request.headers.get('Authorization') === `Bearer ${token}`
	if (!isAuthorized) {
		// 未授权请求重定向到搞笑视频
		return redirect('https://www.youtube.com/watch?v=dQw4w9WgXcQ')
	}
	
	// 解析请求体
	const { key, cacheValue } = z
		.object({ key: z.string(), cacheValue: z.unknown().optional() })
		.parse(await request.json())
	
	// 执行缓存操作
	if (cacheValue === undefined) {
		await cache.delete(key)  // 删除
	} else {
		await cache.set(key, cacheValue)  // 设置
	}
	
	return { success: true }
}
```

### 数据流图

```
┌─────────────────────────────────────────────────────────────────┐
│                    Replica 实例（任意副本）                        │
│                                                                   │
│   用户请求 ──► cache.set(key, value)                             │
│                         │                                         │
│                         ▼                                         │
│              getInstanceInfo() ──► currentIsPrimary = false      │
│                         │                                         │
│                         ▼                                         │
│              updatePrimaryCacheValue(key, value)                  │
│                         │                                         │
│                         ▼                                         │
│              fetch(primary-domain/admin/cache/sqlite)            │
│                    │                                              │
└────────────────────┼──────────────────────────────────────────────┘
                     │ HTTP POST (内部网络)
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Primary 实例（主实例）                          │
│                                                                   │
│   POST /admin/cache/sqlite                                       │
│           │                                                       │
│           ▼                                                       │
│   action() 函数                                                   │
│           │                                                       │
│           ├──► 验证 Authorization Bearer token                   │
│           │                                                       │
│           ├──► 验证 currentIsPrimary = true                      │
│           │                                                       │
│           └──► cache.set(key, value)                             │
│                    │                                              │
│                    ▼                                              │
│           SQLite 数据库写入（可写）                                 │
│                    │                                              │
│                    ▼                                              │
│           LiteFS 自动同步到所有 Replica 实例                       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 关键设计要点

1. **Fire-and-Forget 模式**：
   - 使用 `void updatePrimaryCacheValue(...).then(...)`
   - 不等待响应，避免阻塞请求
   - 失败只记录日志，不影响主流程

2. **安全性**：
   - `INTERNAL_COMMAND_TOKEN` 环境变量用于认证
   - 只有持有相同 token 的实例才能调用管理端点
   - 未授权请求会被重定向（幽默的安全措施）

3. **数据同步**：
   - Primary 实例写入后，LiteFS 会自动同步到所有 Replica
   - 各实例的 LRU Cache 是独立的，不会同步
   - 下次读取时如果 SQLite 已更新，LRU Cache 会被刷新

4. **错误处理**：
   - 转发失败只记录错误日志
   - 不会抛出异常，不影响用户请求
   - 容忍短暂的缓存不一致（最终一致性）

---

## 总结

### 架构设计原则

| 问题 | 解决方案 | 实现位置 |
|------|----------|----------|
| 如何识别 Primary？ | `getInstanceInfo()` 返回 `currentIsPrimary` | `litefs.server.ts` |
| 如何确保写入在 Primary？ | `ensurePrimary()` 透明转发请求 | `callback.ts` (OAuth) |
| 缓存更新如何转发？ | HTTP POST 到 Primary 的管理端点 | `cache.server.ts` + `sqlite.server.ts` |
| 如何认证内部调用？ | `INTERNAL_COMMAND_TOKEN` Bearer Token | `sqlite.server.ts` |

### 一致性模型

- **强一致性**：通过 `ensurePrimary()` 确保的写入操作（如 OAuth 回调）
- **最终一致性**：缓存更新（fire-and-forget，允许短暂不一致）
- **自动同步**：LiteFS 处理 SQLite 数据库的跨实例同步

### 环境变量依赖

| 变量名 | 用途 |
|--------|------|
| `CACHE_DATABASE_PATH` | SQLite 缓存数据库路径 |
| `INTERNAL_COMMAND_TOKEN` | 实例间内部调用的认证 Token |
| `FLY_REGION` / `FLY_ALLOC_ID` 等 | Fly.io 平台提供的实例信息（由 litefs-js 使用） |
