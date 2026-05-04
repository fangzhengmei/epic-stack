# LiteFS 多实例架构分析

## 文档说明

本文档区分两类信息：
- **代码可证实事实**：可以通过阅读代码直接验证的实现细节
- **平台推断**：基于 Fly.io/LiteFS 平台特性的合理推断，需要参考平台文档确认

---

## 1. 应用如何读取当前实例和 Primary 信息

### 核心机制

应用通过 `litefs-js` 库提供的 API 来获取实例信息。相关代码位于 `app/utils/litefs.server.ts`。

### 导出的 API（代码可证实事实）

```typescript
// 从 litefs-js 导出
export {
	getInstanceInfo,        // 异步获取实例信息
	getInstanceInfoSync,    // 同步获取实例信息
	getInternalInstanceDomain,  // 获取实例内部域名
	getAllInstances,        // 获取所有实例列表
	TXID_NUM_COOKIE_NAME,   // 事务 ID Cookie 名称
	waitForUpToDateTxNumber, // 等待事务号同步
	getTxNumber,            // 获取当前事务号
	getTxSetCookieHeader,   // 获取设置 Cookie 的 Header
	checkCookieForTransactionalConsistency, // 检查 Cookie 确保事务一致性
} from 'litefs-js'

// 从 litefs-js/remix 导出
export {
	ensurePrimary,      // 确保在主实例执行
	ensureInstance,     // 确保在指定实例执行
	getReplayResponse,  // 获取重放响应
	handleTransactionalConsistency, // 处理事务一致性
	appendTxNumberCookie, // 追加事务号 Cookie
} from 'litefs-js/remix'
```

### 返回的实例信息结构

调用 `getInstanceInfo()` 或 `getInstanceInfoSync()` 会返回包含以下关键信息的对象：

- **`currentIsPrimary`**: `boolean` - 当前实例是否是 Primary（主实例）
- **`currentInstance`**: `string` - 当前实例的主机名/标识
- **`primaryInstance`**: `string` - Primary 实例的主机名/标识

### 使用示例（代码可证实事实）

```typescript
// 同步获取（在数据库初始化时使用）
const { currentIsPrimary } = getInstanceInfoSync()

// 异步获取（在请求处理时使用）
const { currentIsPrimary, primaryInstance } = await getInstanceInfo()
```

### 实际使用场景

1. **数据库初始化** (`cache.server.ts:36`)：
   ```typescript
   const { currentIsPrimary } = getInstanceInfoSync()
   if (!currentIsPrimary) return db  // 非主实例不执行表创建
   ```

2. **缓存操作判断** (`cache.server.ts:159`, `cache.server.ts:180`)：
   ```typescript
   const { currentIsPrimary, primaryInstance } = await getInstanceInfo()
   if (currentIsPrimary) {
       // 直接执行数据库操作
   } else {
       // 转发到主实例
   }
   ```

3. **管理路由实例切换** (`sqlite.$cacheKey.ts:14-18`)：
   ```typescript
   const currentInstanceInfo = await getInstanceInfo()
   const allInstances = await getAllInstances()
   const instance = searchParams.get('instance') ?? currentInstanceInfo.currentInstance
   await ensureInstance(instance)
   ```

### litefs-js 获取实例信息的来源（平台推断）

虽然代码中直接调用了 `getInstanceInfo()`，但其内部实现依赖于 Fly.io 平台环境：

1. **环境变量来源**：
   - `FLY_REGION` - 当前实例所在区域
   - `FLY_ALLOC_ID` - 当前实例的唯一标识
   - `FLY_APP_NAME` - 应用名称
   - `PRIMARY_REGION` - 主区域配置
   - `FLY_CONSUL_URL` - Consul 服务地址（用于 Primary 选举）

2. **LiteFS 系统文件**：
   - LiteFS 在 `/data/litefs` 目录下维护集群状态
   - 包含当前实例角色（Primary/Replica）、已知实例列表等信息

3. **Consul 协调**：
   - 从 `litefs.yml` 配置可见，使用 `consul` 类型的 lease 进行 Primary 选举
   - 只有 `FLY_REGION == PRIMARY_REGION` 的实例才会成为 candidate

---

## 2. 实例信息在请求与响应中的消费链路

### 2.1 响应头中的实例信息注入（代码可证实事实）

在 `app/entry.server.tsx` 中，所有响应都会被注入实例信息的 HTTP 头：

```typescript
// 处理文档请求（HTML 页面）
export default async function handleRequest(...args: DocRequestArgs) {
	const [request, responseStatusCode, responseHeaders, reactRouterContext] = args
	const { currentInstance, primaryInstance } = await getInstanceInfo()
	
	// 注入实例信息到响应头
	responseHeaders.set('fly-region', process.env.FLY_REGION ?? 'unknown')
	responseHeaders.set('fly-app', process.env.FLY_APP_NAME ?? 'unknown')
	responseHeaders.set('fly-primary-instance', primaryInstance)
	responseHeaders.set('fly-instance', currentInstance)
	// ... 后续渲染逻辑
}

// 处理数据请求（API/JSON）
export async function handleDataRequest(response: Response) {
	const { currentInstance, primaryInstance } = await getInstanceInfo()
	response.headers.set('fly-region', process.env.FLY_REGION ?? 'unknown')
	response.headers.set('fly-app', process.env.FLY_APP_NAME ?? 'unknown')
	response.headers.set('fly-primary-instance', primaryInstance)
	response.headers.set('fly-instance', currentInstance)
	
	return response
}
```

### 2.2 注入的响应头详解

| Header 名称 | 来源 | 用途 |
|-------------|------|------|
| `fly-region` | `process.env.FLY_REGION` | 当前实例所在的地理区域 |
| `fly-app` | `process.env.FLY_APP_NAME` | Fly.io 应用名称 |
| `fly-primary-instance` | `getInstanceInfo().primaryInstance` | 当前 Primary 实例的主机名 |
| `fly-instance` | `getInstanceInfo().currentInstance` | 处理该请求的实例主机名 |

### 2.3 请求-响应链路完整视图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户请求流                                        │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐     ┌──────────────────┐     ┌─────────────────────────────┐
│   浏览器      │────▶│  Fly.io 负载均衡  │────▶│    任意实例 (Replica/Primary) │
└──────────────┘     └──────────────────┘     └─────────────────────────────┘
                                                                │
                                                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         实例内部处理流程                                       │
└─────────────────────────────────────────────────────────────────────────────┘

  1. 请求到达 Remix 路由处理
     │
     ▼
  2. 路由检查（如需要写入操作）
     ├── OAuth 回调 → 调用 `ensurePrimary()`
     ├── 缓存管理页面 → 调用 `ensureInstance(target)`
     └── 普通读取 → 直接处理
     │
     ▼
  3. 如果当前不是目标实例
     │
     ├──▶ `ensurePrimary()` / `ensureInstance()`
     │       │
     │       ▼
     │    返回 Replay Response（平台推断：Fly.io 代理识别并重放）
     │       │
     │       ▼
     │    Fly.io 代理将请求转发到目标实例
     │
     ▼
  4. 在目标实例重新处理（如果需要重放）
     │
     ▼
  5. 执行业务逻辑
     ├── 读取操作：直接查询本地 SQLite（已通过 LiteFS 同步）
     ├── 写入操作（Primary）：直接写入本地 SQLite
     └── 缓存更新（Replica）：HTTP POST 转发到 Primary
     │
     ▼
  6. `entry.server.tsx` 注入响应头
     │
     ├── `fly-region: sjc`
     ├── `fly-app: my-app`
     ├── `fly-primary-instance: instance-1`
     └── `fly-instance: instance-2`（实际处理请求的实例）
     │
     ▼
  7. 响应返回给用户

┌─────────────────────────────────────────────────────────────────────────────┐
│                           响应头信息的用途                                     │
└─────────────────────────────────────────────────────────────────────────────┘

  - **调试/监控**：开发者可以通过查看响应头了解请求由哪个实例处理
  - **问题排查**：当出现数据不一致时，可以追踪是哪个区域的实例
  - **A/B 测试**：理论上可以用于按区域路由（但当前未实现）
```

### 2.4 Replay Response 机制详解（平台推断）

当 `ensurePrimary()` 或 `ensureInstance()` 检测到当前实例不是目标实例时，会返回一个特殊的响应。这个响应的格式由 Fly.io 平台定义：

**Replay Response 特征**：
- HTTP 状态码：通常是 `409 Conflict` 或特殊的 `1xx` 状态码
- 包含 `fly-replay` Header，指定目标实例
- 响应体可能包含重放指令

**Fly.io 代理行为**：
1. 代理收到响应后检查 `fly-replay` Header
2. 如果存在，代理不将响应返回给用户
3. 代理将**原始请求**重新发送到目标实例
4. 目标实例的响应直接返回给用户
5. 整个过程对用户透明（没有浏览器重定向）

**代码中的间接证据**：
```typescript
// litefs.server.ts 导出了这些函数
export {
	getReplayResponse,  // 获取重放响应
	ensurePrimary,      // 内部可能调用 getReplayResponse
	ensureInstance,     // 内部可能调用 getReplayResponse
} from 'litefs-js/remix'
```

### 2.5 事务一致性的 Cookie 机制（代码可证实事实）

代码中导出了多个与事务 ID 相关的函数，这是 LiteFS 多实例架构中的另一个重要机制：

```typescript
export {
	TXID_NUM_COOKIE_NAME,           // Cookie 名称常量
	getTxNumber,                     // 获取当前事务号
	getTxSetCookieHeader,            // 生成设置 Cookie 的 Header
	checkCookieForTransactionalConsistency, // 检查 Cookie
	waitForUpToDateTxNumber,         // 等待事务同步
	handleTransactionalConsistency,  // Remix 集成的一致性处理
	appendTxNumberCookie,            // 追加 Cookie 到响应
} from 'litefs-js'
```

**使用场景推断**：
1. **写入后读取一致性**：
   - Primary 写入后生成事务号 `TXN`
   - 通过 `Set-Cookie: txnum=TXN` 返回给客户端
   - 客户端后续请求携带该 Cookie
   - Replica 收到请求后，检查本地数据库是否已同步到 `TXN`
   - 如果没有，等待同步后再处理请求

2. **防止读取旧数据**：
   - 用户刚刚在 Primary 创建了数据
   - 后续请求被路由到 Replica
   - 如果 Replica 还没同步到该事务，会等待或返回错误

**为什么当前代码中没有直接使用**：
- 这些函数被导出但在应用代码中未直接调用
- 可能由 `litefs-js/remix` 的中间件自动处理
- 或在特定场景（如表单提交后重定向）才需要显式使用

---

## 3. OAuth 回调如何确保跑在主实例

### 问题背景

在 LiteFS 多实例架构中：
- **Primary 实例**：可写，负责处理所有写入操作
- **Replica 实例**：只读，只能处理读取操作
- SQLite 数据库通过 LiteFS 自动同步，但写入必须在 Primary 实例执行

OAuth 回调流程涉及：
1. 数据库查询（读取现有连接）
2. 创建新连接（写入）
3. 创建新会话（写入）
4. 关联用户账户（写入）

这些写入操作**必须**在 Primary 实例执行，否则会失败。

### 解决方案：`ensurePrimary()`（代码可证实事实）

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

**代码可证实部分**：
- `ensurePrimary()` 从 `litefs-js/remix` 导入
- 在执行任何写入操作前必须调用
- 如果当前不是 Primary，会抛出或返回特殊响应

**平台推断部分**：
- 内部调用 `getInstanceInfo()` 检查 `currentIsPrimary`
- 如果不是 Primary，调用 `getReplayResponse()` 返回重放响应
- Fly.io 代理识别该响应，将原始请求透明转发到 Primary

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

## 4. SQLite Cache 如何把更新转发到 Primary

### 问题背景

应用使用了两层缓存：
1. **LRU Cache**：内存缓存，每个实例独立
2. **SQLite Cache**：持久化缓存，通过 LiteFS 同步

SQLite Cache 的写入问题：
- 任意实例都可能触发缓存更新（`cache.set()` 或 `cache.delete()`）
- 但只有 Primary 实例可以写入 SQLite 数据库
- Replica 实例的写入需要**转发**到 Primary 实例执行

### 核心实现：`cache.server.ts`（代码可证实事实）

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

### 转发机制：`updatePrimaryCacheValue()`（代码可证实事实）

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

### Primary 实例的接收端点（代码可证实事实）

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
                     │ Header: Authorization: Bearer ${INTERNAL_COMMAND_TOKEN}
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
│           （平台推断：LiteFS 后台异步同步）                         │
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

## 5. 缓存转发失败时的一致性影响与缓解建议

### 5.1 可能的失败场景（代码可证实事实 + 平台推断）

当前实现中，`updatePrimaryCacheValue()` 可能失败的场景：

| 失败场景 | 原因 | 代码证据 |
|----------|------|----------|
| **网络分区** | Replica 无法到达 Primary | `fetch()` 可能抛出网络错误 |
| **Primary 不可用** | Primary 实例崩溃或重启 | `fetch()` 连接失败 |
| **Token 不匹配** | 实例间 `INTERNAL_COMMAND_TOKEN` 不一致 | `action()` 中检查 `Authorization` Header |
| **请求超时** | 内部网络延迟或拥塞 | `fetch()` 超时 |
| **Primary 角色切换** | 转发过程中 Primary 发生故障转移 | `action()` 中检查 `currentIsPrimary` |

### 5.2 失败时的代码行为（代码可证实事实）

```typescript
// cache.server.ts:164-176
void updatePrimaryCacheValue({
	key,
	cacheValue: entry,
}).then((response) => {
	if (!response.ok) {
		// 只记录日志，不抛出异常
		console.error(
			`Error updating cache value for key "${key}" on primary instance (${primaryInstance}): ${response.status} ${response.statusText}`,
			{ entry },
		)
	}
})
// 注意：没有 .catch() 处理 fetch 抛出的异常！
```

**当前实现的问题**：
1. **Fire-and-Forget**：`void` 关键字表示不等待 Promise 完成
2. **只处理 HTTP 错误**：`.then()` 只检查 `response.ok`，但没有 `.catch()`
3. **可能的未处理 Promise 拒绝**：如果 `fetch()` 本身抛出（如网络错误），会产生未处理的拒绝
4. **无重试机制**：失败后不会自动重试
5. **无回退策略**：失败后没有本地缓存或其他补偿措施

### 5.3 一致性影响分析

#### 影响层级

```
┌─────────────────────────────────────────────────────────────────────┐
│                      一致性影响层级                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. 第一层：用户感知影响                                               │
│     ├── 不同用户看到不同数据                                          │
│     ├── 操作后立即读取显示旧数据                                      │
│     └── 缓存数据与数据库不一致                                        │
│                                                                       │
│  2. 第二层：系统内部影响                                               │
│     ├── 各实例 LRU Cache 不一致                                      │
│     ├── SQLite Cache 文件不一致（直到 LiteFS 同步）                  │
│     └── 日志污染（大量错误日志）                                      │
│                                                                       │
│  3. 第三层：业务逻辑影响                                               │
│     ├── 缓存穿透：大量请求绕过缓存直达数据库                          │
│     ├── 缓存击穿：热点 key 失效导致数据库压力                        │
│     └── 数据脏读：基于过期缓存做出错误决策                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

#### 具体场景分析

**场景 1：缓存更新转发失败**
- **发生了什么**：Replica A 尝试更新 `cache.set("user:123", newData)` 到 Primary
- **失败原因**：网络分区
- **后果**：
  - Primary 的 SQLite Cache 中仍是旧数据
  - Replica A 的 LRU Cache 中是新数据（注意：内存缓存没有同步！）
  - 其他 Replica 的 LRU Cache 和 SQLite Cache 都是旧数据
- **用户感知**：
  - 连接到 Replica A 的用户看到新数据
  - 连接到其他实例的用户看到旧数据
  - **数据不一致**

**场景 2：缓存删除转发失败**
- **发生了什么**：Replica B 尝试删除 `cache.delete("session:abc")`
- **失败原因**：Primary 临时不可用
- **后果**：
  - Primary 的 SQLite Cache 中该 key 仍然存在
  - LiteFS 同步后，所有 Replica 的 SQLite Cache 都有该 key
  - 但 Replica B 的 LRU Cache 已删除（如果有的话）
- **用户感知**：
  - 已失效的会话可能被重新激活
  - **安全隐患**

**场景 3：连续多次更新均失败**
- **发生了什么**：同一个 key 在短时间内被更新多次，每次转发都失败
- **后果**：
  - 中间状态全部丢失
  - 最终状态无法确定
  - **数据混乱**

### 5.4 当前架构的最终一致性保证

**什么是保证的**（平台推断）：
1. **LiteFS 数据库同步**：一旦 Primary 成功写入，LiteFS 会保证最终同步到所有 Replica
2. **LRU Cache 自然过期**：每个实例的 LRU Cache 有 TTL，过期后会重新从 SQLite 读取
3. **缓存击穿保护**：`@epic-web/cachified` 应该有防止缓存击穿的机制（需要确认）

**什么是不保证的**：
1. **转发成功**：`updatePrimaryCacheValue()` 不保证一定成功
2. **写入顺序**：如果多个 Replica 同时更新同一个 key，顺序无法保证
3. **立即一致性**：即使转发成功，LiteFS 同步也需要时间

### 5.5 缓解建议

#### 建议 1：完善错误处理（低风险，高收益）

**当前问题**：没有 `.catch()` 处理 `fetch()` 抛出的异常

**改进代码**：
```typescript
async set(key, entry) {
	const { currentIsPrimary, primaryInstance } = await getInstanceInfo()

	if (currentIsPrimary) {
		const value = JSON.stringify(entry.value, bufferReplacer)
		setStatement.run(key, value, JSON.stringify(entry.metadata))
	} else {
		// 不再使用 void，而是添加完整的错误处理
		updatePrimaryCacheValue({
			key,
			cacheValue: entry,
		})
		.then((response) => {
			if (!response.ok) {
				console.error(
					`Error updating cache value for key "${key}" on primary instance (${primaryInstance}): ${response.status} ${response.statusText}`,
					{ entry },
				)
			}
		})
		.catch((error) => {
			// 捕获网络错误等异常
			console.error(
				`Network error updating cache value for key "${key}" on primary instance (${primaryInstance})`,
				{ error },
			)
		})
	}
}
```

#### 建议 2：实现重试机制（中等风险，高收益）

**实现指数退避重试**：
```typescript
// 新增：带重试的转发函数
async function updatePrimaryCacheValueWithRetry({
	key,
	cacheValue,
	maxRetries = 3,
}: {
	key: string
	cacheValue: any
	maxRetries?: number
}): Promise<Response> {
	let lastError: Error | undefined
	
	for (let attempt = 0; attempt < maxRetries; attempt++) {
		try {
			const response = await updatePrimaryCacheValue({ key, cacheValue })
			if (response.ok) {
				return response
			}
			// 4xx 错误不需要重试（如 token 无效）
			if (response.status >= 400 && response.status < 500) {
				return response
			}
			// 5xx 错误可以重试
			lastError = new Error(`HTTP ${response.status}`)
		} catch (error) {
			lastError = error as Error
		}
		
		// 指数退避：100ms, 200ms, 400ms...
		const delay = Math.pow(2, attempt) * 100
		await new Promise(resolve => setTimeout(resolve, delay))
	}
	
	throw lastError ?? new Error('Max retries exceeded')
}

// 使用
async set(key, entry) {
	const { currentIsPrimary } = await getInstanceInfo()

	if (currentIsPrimary) {
		const value = JSON.stringify(entry.value, bufferReplacer)
		setStatement.run(key, value, JSON.stringify(entry.metadata))
	} else {
		// 带重试
		updatePrimaryCacheValueWithRetry({
			key,
			cacheValue: entry,
			maxRetries: 3,
		})
		.catch((error) => {
			console.error(`Failed to update cache after retries: ${key}`, { error })
			// TODO: 可以考虑写入本地队列，稍后重试
		})
	}
}
```

#### 建议 3：实现本地持久化队列（高风险，高收益）

**问题**：如果 Primary 长时间不可用，重试也会失败

**解决方案**：实现本地持久化队列，后台异步处理

```typescript
import { DatabaseSync } from 'node:sqlite'
import { remember } from '@epic-web/remember'

// 本地队列数据库（独立于主缓存）
const queueDb = remember('cacheUpdateQueueDb', () => {
	const db = new DatabaseSync('/data/cache-queue.db')
	db.exec(`
		CREATE TABLE IF NOT EXISTS pending_updates (
			id INTEGER PRIMARY KEY AUTOINCREMENT,
			key TEXT NOT NULL,
			cache_value TEXT,  -- JSON 序列化后的值
			operation TEXT NOT NULL,  -- 'set' | 'delete'
			created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
			attempts INTEGER DEFAULT 0,
			last_error TEXT
		)
	`)
	return db
})

const insertStatement = queueDb.prepare(`
	INSERT INTO pending_updates (key, cache_value, operation)
	VALUES (?, ?, ?)
`)

// 入队函数
function enqueueCacheUpdate({
	key,
	cacheValue,
	operation,
}: {
	key: string
	cacheValue: any
	operation: 'set' | 'delete'
}) {
	const serializedValue = cacheValue === undefined 
		? null 
		: JSON.stringify(cacheValue)
	insertStatement.run(key, serializedValue, operation)
}

// 后台处理队列
async function processCacheQueue() {
	const pending = queueDb.prepare(`
		SELECT * FROM pending_updates 
		WHERE attempts < 5 
		ORDER BY created_at ASC 
		LIMIT 10
	`).all() as Array<{
		id: number
		key: string
		cache_value: string | null
		operation: 'set' | 'delete'
		attempts: number
	}>

	for (const item of pending) {
		try {
			const cacheValue = item.cache_value === null 
				? undefined 
				: JSON.parse(item.cache_value)
			
			const response = await updatePrimaryCacheValue({
				key: item.key,
				cacheValue,
			})

			if (response.ok) {
				// 成功，删除队列项
				queueDb.prepare('DELETE FROM pending_updates WHERE id = ?').run(item.id)
			} else {
				// HTTP 错误，更新尝试次数
				queueDb.prepare(`
					UPDATE pending_updates 
					SET attempts = attempts + 1, last_error = ?
					WHERE id = ?
				`).run(`HTTP ${response.status}`, item.id)
			}
		} catch (error) {
			// 网络错误，更新尝试次数
			queueDb.prepare(`
				UPDATE pending_updates 
				SET attempts = attempts + 1, last_error = ?
				WHERE id = ?
			`).run(String(error), item.id)
		}
	}
}

// 启动后台处理（应用启动时）
let queueProcessor: ReturnType<typeof setInterval> | undefined
if (process.env.NODE_ENV === 'production') {
	// 每 5 秒处理一次队列
	queueProcessor = setInterval(processCacheQueue, 5000)
}

// 改进后的 cache.set
async set(key, entry) {
	const { currentIsPrimary } = await getInstanceInfo()

	if (currentIsPrimary) {
		const value = JSON.stringify(entry.value, bufferReplacer)
		setStatement.run(key, value, JSON.stringify(entry.metadata))
	} else {
		// 先尝试直接转发
		updatePrimaryCacheValue({ key, cacheValue: entry })
			.then((response) => {
				if (!response.ok) {
					// HTTP 失败，入队
					enqueueCacheUpdate({ key, cacheValue: entry, operation: 'set' })
				}
			})
			.catch(() => {
				// 网络异常，入队
				enqueueCacheUpdate({ key, cacheValue: entry, operation: 'set' })
			})
	}
}
```

#### 建议 4：使用 `ensurePrimary()` 处理关键写入（推荐用于缓存外场景）

对于**关键数据**（非缓存），应该使用 `ensurePrimary()` 而不是 HTTP 转发：

```typescript
// 错误示例：不要这样做
async function updateUserProfile(userId: string, data: UserData) {
	// ❌ 这不是缓存，不能接受最终不一致
	if (currentIsPrimary) {
		await prisma.user.update({ where: { id: userId }, data })
	} else {
		// ❌ 转发失败会导致数据丢失
		await forwardToPrimary({ userId, data })
	}
}

// 正确示例
async function loader({ request }: Route.LoaderArgs) {
	// ✅ 确保在 Primary 执行
	await ensurePrimary()
	
	// ✅ 直接操作，不担心转发失败
	await prisma.user.update({ where: { id: userId }, data })
}
```

**缓存 vs 关键数据的一致性要求**：

| 数据类型 | 一致性要求 | 推荐方案 |
|----------|------------|----------|
| **缓存数据** | 最终一致性可接受 | HTTP 转发 + 重试 + 队列 |
| **用户会话** | 强一致性 | `ensurePrimary()` |
| **OAuth 连接** | 强一致性 | `ensurePrimary()` |
| **用户资料** | 强一致性 | `ensurePrimary()` |
| **应用配置** | 最终一致性可接受 | HTTP 转发 |

#### 建议 5：监控与告警

**需要监控的指标**：
1. `updatePrimaryCacheValue` 失败率
2. 待处理队列长度（如果实现了队列）
3. 各实例缓存命中率差异
4. `fly-primary-instance` 变化频率（Primary 切换）

**告警策略**：
- 失败率 > 10% 持续 5 分钟 → P2 告警
- 失败率 > 50% 持续 1 分钟 → P1 告警
- 队列长度 > 100 持续 10 分钟 → P2 告警
- Primary 切换（非预期）→ P1 告警

#### 建议 6：考虑使用 LRU Cache 作为写入侧缓存（权衡方案）

**问题**：当前设计中，Replica 的 LRU Cache 可能与 SQLite Cache 不一致

**权衡方案**：在转发成功前，不更新本地 LRU Cache，或使用更短的 TTL

```typescript
async set(key, entry) {
	const { currentIsPrimary, primaryInstance } = await getInstanceInfo()

	if (currentIsPrimary) {
		// Primary：直接写入 SQLite，LRU 会在下次读取时刷新
		const value = JSON.stringify(entry.value, bufferReplacer)
		setStatement.run(key, value, JSON.stringify(entry.metadata))
	} else {
		// Replica：只转发，不立即更新 LRU
		// 这样可以避免 LRU 和 SQLite 不一致
		// 缺点：用户下次读取会缓存未命中
		void updatePrimaryCacheValue({
			key,
			cacheValue: entry,
		})
		// 注意：这里不更新 lruCache.set(key, entry)
		// 等待 LiteFS 同步后，下次读取从 SQLite 刷新
	}
}
```

**权衡分析**：
| 方案 | 优点 | 缺点 |
|------|------|------|
| **当前方案**（立即更新 LRU） | 写入后立即读取快 | 可能读到不一致数据 |
| **等待同步** | 数据一致性好 | 写入后首次读取慢 |
| **短 TTL** | 折中方案 | 实现复杂 |

---

## 6. 总结

### 6.1 架构设计原则

| 问题 | 解决方案 | 实现位置 | 证据类型 |
|------|----------|----------|----------|
| 如何识别 Primary？ | `getInstanceInfo()` 返回 `currentIsPrimary` | `litefs.server.ts` | 代码可证实 |
| 如何确保写入在 Primary？ | `ensurePrimary()` 透明转发请求 | `callback.ts` (OAuth) | 代码可证实 |
| Replay Response 如何工作？ | Fly.io 代理识别并重放请求 | 平台特性 | 平台推断 |
| 缓存更新如何转发？ | HTTP POST 到 Primary 的管理端点 | `cache.server.ts` + `sqlite.server.ts` | 代码可证实 |
| 如何认证内部调用？ | `INTERNAL_COMMAND_TOKEN` Bearer Token | `sqlite.server.ts` | 代码可证实 |
| LiteFS 如何同步？ | 后台异步复制数据库文件 | LiteFS 特性 | 平台推断 |

### 6.2 一致性模型

- **强一致性**：通过 `ensurePrimary()` 确保的写入操作（如 OAuth 回调）
- **最终一致性**：缓存更新（fire-and-forget，允许短暂不一致）
- **自动同步**：LiteFS 处理 SQLite 数据库的跨实例同步

### 6.3 响应头注入（代码可证实事实）

所有响应都会注入以下 Header（`entry.server.tsx`）：

```
fly-region: sjc
fly-app: my-app
fly-primary-instance: instance-123
fly-instance: instance-456
```

### 6.4 环境变量依赖

| 变量名 | 用途 | 证据类型 |
|--------|------|----------|
| `CACHE_DATABASE_PATH` | SQLite 缓存数据库路径 | 代码可证实 |
| `INTERNAL_COMMAND_TOKEN` | 实例间内部调用的认证 Token | 代码可证实 |
| `FLY_REGION` | 当前实例区域，注入响应头 | 代码可证实 |
| `FLY_APP_NAME` | 应用名称，注入响应头 | 代码可证实 |
| `FLY_CONSUL_URL` | Consul 地址，用于 Primary 选举 | `litefs.yml` 配置 |
| `PRIMARY_REGION` | 主区域配置 | `litefs.yml` 配置 |

### 6.5 缓存转发失败缓解建议优先级

1. **高优先级**：完善错误处理（添加 `.catch()`）
2. **高优先级**：实现指数退避重试
3. **中优先级**：实现本地持久化队列
4. **中优先级**：添加监控与告警
5. **低优先级**：考虑 LRU Cache 一致性策略

### 6.6 关键安全提示

1. **`INTERNAL_COMMAND_TOKEN` 必须保密**：
   - 该 Token 允许任意实例修改缓存
   - 如果泄露，攻击者可以篡改缓存数据
   - 必须使用 `fly secrets` 设置，不要硬编码

2. **管理端点的安全**：
   - `/admin/cache/sqlite` 端点验证 Token
   - 但没有速率限制
   - 理论上可能被暴力破解（如果 Token 强度不够）

3. **建议增强**：
   - 使用长随机 Token（32 字节以上）
   - 考虑添加来源 IP 验证（仅允许内部网络）
   - 考虑添加请求签名或时间戳防止重放

---

## 附录：代码引用速查

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| `ensurePrimary()` 调用 | `app/routes/_auth/auth.$provider/callback.ts` | 34 |
| 响应头注入 | `app/entry.server.tsx` | 32-36, 117-120 |
| 缓存 `set()` 转发 | `app/utils/cache.server.ts` | 158-178 |
| 缓存 `delete()` 转发 | `app/utils/cache.server.ts` | 179-198 |
| `updatePrimaryCacheValue()` | `app/routes/admin/cache/sqlite.server.ts` | 10-33 |
| Primary 接收端点 `action()` | `app/routes/admin/cache/sqlite.server.ts` | 35-59 |
| `ensureInstance()` 使用 | `app/routes/admin/cache/sqlite.$cacheKey.ts` | 18 |
| LiteFS 配置 | `other/litefs.yml` | 全文 |
