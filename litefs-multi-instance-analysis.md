# LiteFS 多实例架构分析（可核验版本）

> 本文档严格区分两类断言：
> - ✅ **仓内代码可证实**：结论可通过阅读仓库内代码直接验证
> - ⚠️ **平台待验证**：结论依赖 `litefs-js` 内部实现或 Fly.io 平台特性，需要参考外部文档确认

---

## 文档核验原则

所有标注为 **仓内代码可证实** 的结论必须满足：
1. 可在仓库内找到对应的代码实现
2. 不依赖对外部库（如 `litefs-js`）内部行为的推断
3. 不依赖对平台特性（如 Fly.io 代理行为）的假设

---

## 1. 应用如何读取当前实例和 Primary 信息

### 1.1 导出的 API（✅ 仓内代码可证实）

**文件位置**: `app/utils/litefs.server.ts`

```typescript
// 从 litefs-js 导出
export {
	getInstanceInfo,
	getInstanceInfoSync,
	getInternalInstanceDomain,
	getAllInstances,
	TXID_NUM_COOKIE_NAME,
	waitForUpToDateTxNumber,
	getTxNumber,
	getTxSetCookieHeader,
	checkCookieForTransactionalConsistency,
} from 'litefs-js'

// 从 litefs-js/remix 导出
export {
	ensurePrimary,
	ensureInstance,
	getReplayResponse,
	handleTransactionalConsistency,
	appendTxNumberCookie,
} from 'litefs-js/remix'
```

**核验方式**: 直接读取 `app/utils/litefs.server.ts` 文件。

---

### 1.2 API 的使用场景（✅ 仓内代码可证实）

#### 场景 1: 数据库初始化判断

**文件位置**: `app/utils/cache.server.ts:36-37`

```typescript
const { currentIsPrimary } = getInstanceInfoSync()
if (!currentIsPrimary) return db  // 非主实例不执行表创建
```

**核验结论**:
- `getInstanceInfoSync()` 返回的对象包含 `currentIsPrimary` 字段
- `currentIsPrimary` 是 `boolean` 类型
- 当 `currentIsPrimary` 为 `false` 时，代码提前返回，不执行 `CREATE TABLE`

**核验方式**: 读取 `app/utils/cache.server.ts`，分析 `createDatabase` 函数的控制流。

---

#### 场景 2: 缓存操作中的实例判断

**文件位置**: `app/utils/cache.server.ts:159-178` (set 方法)

```typescript
const { currentIsPrimary, primaryInstance } = await getInstanceInfo()

if (currentIsPrimary) {
	// 主实例分支：直接执行 setStatement.run()
} else {
	// 副本实例分支：调用 updatePrimaryCacheValue()
}
```

**核验结论**:
- `getInstanceInfo()` 是异步函数（使用 `await`）
- 返回值包含 `currentIsPrimary` 和 `primaryInstance` 两个字段
- `primaryInstance` 是字符串类型（被传入 `updatePrimaryCacheValue` 并在错误日志中使用）

**核验方式**: 读取 `app/utils/cache.server.ts` 中的 `cache.set` 和 `cache.delete` 方法。

---

#### 场景 3: 管理路由中的实例切换

**文件位置**: `app/routes/admin/cache/sqlite.$cacheKey.ts:14-18`

```typescript
const currentInstanceInfo = await getInstanceInfo()
const allInstances = await getAllInstances()
const instance =
	searchParams.get('instance') ?? currentInstanceInfo.currentInstance
await ensureInstance(instance)
```

**核验结论**:
- `getInstanceInfo()` 返回值包含 `currentInstance` 字段
- `getAllInstances()` 返回一个可迭代对象（用于显示所有实例列表）
- `ensureInstance()` 是异步函数，接收实例标识作为参数

**核验方式**: 读取 `app/routes/admin/cache/sqlite.$cacheKey.ts`、`lru.$cacheKey.ts`、`index.tsx`。

---

### 1.3 litefs-js 内部实现（⚠️ 平台待验证）

以下结论**无法**通过仓库内代码证实，需要参考 `litefs-js` 源码或文档：

| 待验证点 | 当前假设 | 验证方式 |
|----------|----------|----------|
| `getInstanceInfo()` 的数据来源 | 从环境变量（`FLY_*`）或 LiteFS 系统文件读取 | 查看 `litefs-js` 源码 |
| 返回值的完整结构 | 是否包含更多字段？ | 查看 `litefs-js` 类型定义 |
| `getAllInstances()` 的返回格式 | 对象？数组？Map？ | 查看 `litefs-js` 源码 |
| `getInternalInstanceDomain()` 的实现 | 如何构建内部域名？使用 `.internal` 后缀？ | 查看 `litefs-js` 源码 |

**对架构理解的影响**:
- 如果 `getInstanceInfo()` 的数据来源不可靠，整个多实例协调的基础就有问题
- 但代码已经在生产环境使用，说明这些假设在 Fly.io 平台上是成立的

---

## 2. 实例信息在请求与响应中的消费链路

### 2.1 响应头注入（✅ 仓内代码可证实）

**文件位置**: `app/entry.server.tsx:32-36, 117-120`

```typescript
// handleRequest - 文档请求
const { currentInstance, primaryInstance } = await getInstanceInfo()
responseHeaders.set('fly-region', process.env.FLY_REGION ?? 'unknown')
responseHeaders.set('fly-app', process.env.FLY_APP_NAME ?? 'unknown')
responseHeaders.set('fly-primary-instance', primaryInstance)
responseHeaders.set('fly-instance', currentInstance)

// handleDataRequest - 数据请求
const { currentInstance, primaryInstance } = await getInstanceInfo()
response.headers.set('fly-region', process.env.FLY_REGION ?? 'unknown')
response.headers.set('fly-app', process.env.FLY_APP_NAME ?? 'unknown')
response.headers.set('fly-primary-instance', primaryInstance)
response.headers.set('fly-instance', currentInstance)
```

**核验结论**:
1. 所有响应（包括 HTML 文档和 JSON 数据）都会被注入以下 Header：
   - `fly-region`: 值来自 `process.env.FLY_REGION`
   - `fly-app`: 值来自 `process.env.FLY_APP_NAME`
   - `fly-primary-instance`: 值来自 `getInstanceInfo().primaryInstance`
   - `fly-instance`: 值来自 `getInstanceInfo().currentInstance`

2. 注入发生在 `handleRequest`（文档请求）和 `handleDataRequest`（数据请求）两个函数中

**核验方式**:
- 读取 `app/entry.server.tsx`
- 确认 `handleRequest` 和 `handleDataRequest` 是 Remix 框架的入口函数
- 确认 `responseHeaders.set()` 调用的参数

---

### 2.2 响应头的消费情况（✅ 仓内代码可证实）

**核验结论**:
- 仓库内代码**没有**读取这些响应头的逻辑
- 这些 Header 的消费者是：
  - 开发者（通过浏览器 DevTools 查看）
  - 外部监控系统
  - Fly.io 平台（可能，但代码未证实）

**核验方式**:
- 在代码库中搜索 `fly-region`、`fly-instance`、`fly-primary-instance`、`fly-app`
- 确认只在 `entry.server.tsx` 中出现（写入），没有其他地方读取

---

### 2.3 Replay Response 机制（⚠️ 平台待验证）

**代码中可证实的部分**:
- `litefs.server.ts` 导出了 `ensurePrimary`、`ensureInstance`、`getReplayResponse` 函数
- 这些函数来自 `litefs-js/remix`

**无法证实的部分**（需要外部验证）:

| 待验证点 | 当前假设 | 验证方式 |
|----------|----------|----------|
| `ensurePrimary()` 的行为 | 检查当前实例，返回特殊响应或抛出 | 查看 `litefs-js/remix` 源码 |
| `getReplayResponse()` 的返回值 | 包含什么 Header？什么状态码？ | 查看 `litefs-js/remix` 源码 |
| Fly.io 代理如何识别 Replay Response | 检测特定 Header？状态码？ | Fly.io 官方文档 |
| 代理如何重放请求 | 保留原始请求的哪些部分？Body？Headers？ | Fly.io 官方文档 |

---

## 3. OAuth 回调如何确保跑在主实例

### 3.1 ensurePrimary() 的调用（✅ 仓内代码可证实）

**文件位置**: `app/routes/_auth/auth.$provider/callback.ts:31-34`

```typescript
export async function loader({ request, params }: Route.LoaderArgs) {
	// this loader performs mutations, so we need to make sure we're on the
	// primary instance to avoid writing to a read-only replica
	await ensurePrimary()
	// ... 后续的 OAuth 处理逻辑
}
```

**核验结论**:
1. `ensurePrimary()` 在 `loader` 函数的**第一行**被调用
2. 调用使用 `await`，说明是异步函数
3. 注释明确说明了目的："this loader performs mutations"
4. `ensurePrimary()` 之后的代码包含写入操作

**核验方式**:
- 读取 `app/routes/_auth/auth.$provider/callback.ts`
- 确认 `ensurePrimary()` 调用位置
- 确认后续代码包含 `prisma.connection.create()`、`prisma.session.create()` 等写入操作

---

### 3.2 后续写入操作（✅ 仓内代码可证实）

**文件位置**: `app/routes/_auth/auth.$provider/callback.ts` 中多处

```typescript
// 示例 1: 创建连接
await prisma.connection.create({
	data: {
		providerName,
		providerId: String(profile.id),
		userId,
	},
})

// 示例 2: 创建会话
const session = await prisma.session.create({
	select: { id: true, expirationDate: true, userId: true },
	data: {
		expirationDate: getSessionExpirationDate(),
		userId,
	},
})
```

**核验结论**:
- `ensurePrimary()` 之后有多个写入操作
- 如果 `ensurePrimary()` 不能确保在 Primary 执行，这些写入会失败

---

### 3.3 ensurePrimary() 的实际效果（⚠️ 平台待验证）

**关键待验证点**：

| 待验证点 | 对 OAuth 保障的影响 | 验证方式 |
|----------|---------------------|----------|
| `ensurePrimary()` 返回值类型 | 返回 Response？抛出异常？还是什么都不做？ | `litefs-js/remix` 源码 |
| Remix 如何处理这个返回值 | Remix loader 返回 Response 会直接使用？ | Remix 文档 |
| Fly.io 代理如何识别并转发 | 什么触发代理转发？ | Fly.io 文档 |
| 请求重放的完整性 | Body、Headers、Cookie 都被保留吗？ | Fly.io 文档 |
| 用户/客户端是否感知 | 浏览器会看到重定向吗？ | 实测或文档 |

**如果这些假设不成立的风险**:

```
假设: ensurePrimary() 返回一个 Replay Response，Fly.io 代理透明转发

风险场景 1: ensurePrimary() 只是检查并抛出异常
    → 后果: OAuth 回调在 Replica 上会直接 500 错误
    → 影响: 用户无法登录

风险场景 2: Fly.io 代理不识别或不处理 Replay Response
    → 后果: 特殊响应返回给浏览器，可能显示错误页面
    → 影响: 用户无法登录

风险场景 3: 请求重放时丢失了 OAuth 回调的关键参数
    → 后果: Primary 收到的请求不完整，OAuth 流程失败
    → 影响: 用户无法登录，或登录状态不一致
```

---

## 4. SQLite Cache 如何把更新转发到 Primary

### 4.1 cache.set() 的分支逻辑（✅ 仓内代码可证实）

**文件位置**: `app/utils/cache.server.ts:158-178`

```typescript
async set(key, entry) {
	const { currentIsPrimary, primaryInstance } = await getInstanceInfo()

	if (currentIsPrimary) {
		// 分支 A: 主实例
		const value = JSON.stringify(entry.value, bufferReplacer)
		setStatement.run(key, value, JSON.stringify(entry.metadata))
	} else {
		// 分支 B: 副本实例
		void updatePrimaryCacheValue({
			key,
			cacheValue: entry,
		}).then((response) => {
			if (!response.ok) {
				console.error(...)
			}
		})
	}
}
```

**核验结论**:
1. `cache.set()` 是异步函数
2. 根据 `currentIsPrimary` 决定执行路径
3. **Primary 路径**: 直接调用 `setStatement.run()` 写入 SQLite
4. **Replica 路径**: 调用 `updatePrimaryCacheValue()`，使用 `void` 关键字（fire-and-forget）
5. 只有 `.then()` 处理响应，**没有 `.catch()`** 处理 `fetch()` 可能抛出的异常

**核验方式**:
- 读取 `app/utils/cache.server.ts` 中的 `cache` 对象定义
- 确认 `set` 和 `delete` 方法的结构
- 确认 `void` 关键字和 `.then()` 的使用

---

### 4.2 updatePrimaryCacheValue() 的实现（✅ 仓内代码可证实）

**文件位置**: `app/routes/admin/cache/sqlite.server.ts:10-33`

```typescript
export async function updatePrimaryCacheValue({
	key,
	cacheValue,
}: {
	key: string
	cacheValue: any
}) {
	const { currentIsPrimary, primaryInstance } = await getInstanceInfo()
	
	// 安全检查
	if (currentIsPrimary) {
		throw new Error(
			`updatePrimaryCacheValue should not be called on the primary instance (${primaryInstance})}`,
		)
	}
	
	// 构建目标 URL
	const domain = getInternalInstanceDomain(primaryInstance)
	const token = process.env.INTERNAL_COMMAND_TOKEN
	
	// 发送请求
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

**核验结论**:
1. 这是一个异步函数，返回 `fetch()` 的结果（`Promise<Response>`）
2. 首先检查 `currentIsPrimary`，如果是 Primary 则抛出异常
3. 使用 `getInternalInstanceDomain(primaryInstance)` 构建目标域名
4. 从 `process.env.INTERNAL_COMMAND_TOKEN` 获取认证 Token
5. 发送 `POST` 请求到 `${domain}/admin/cache/sqlite`
6. 请求包含：
   - `Authorization: Bearer ${token}` Header
   - `Content-Type: application/json` Header
   - Body: `{ key, cacheValue }`（JSON 序列化）

**核验方式**:
- 读取 `app/routes/admin/cache/sqlite.server.ts`
- 确认函数签名和实现
- 确认 `fetch()` 调用的参数

---

### 4.3 Primary 接收端点的实现（✅ 仓内代码可证实）

**文件位置**: `app/routes/admin/cache/sqlite.server.ts:35-59`

```typescript
export async function action({ request }: Route.ActionArgs) {
	const { currentIsPrimary, primaryInstance } = await getInstanceInfo()
	
	// 检查 1: 必须是 Primary
	if (!currentIsPrimary) {
		throw new Error(
			`${request.url} should only be called on the primary instance (${primaryInstance})}`,
		)
	}
	
	// 检查 2: Token 认证
	const token = process.env.INTERNAL_COMMAND_TOKEN
	const isAuthorized =
		request.headers.get('Authorization') === `Bearer ${token}`
	if (!isAuthorized) {
		return redirect('https://www.youtube.com/watch?v=dQw4w9WgXcQ')
	}
	
	// 解析请求体
	const { key, cacheValue } = z
		.object({ key: z.string(), cacheValue: z.unknown().optional() })
		.parse(await request.json())
	
	// 执行操作
	if (cacheValue === undefined) {
		await cache.delete(key)  // 删除操作
	} else {
		await cache.set(key, cacheValue)  // 设置操作
	}
	
	return { success: true }
}
```

**核验结论**:
1. 这是一个 Remix `action` 函数（处理 POST 请求）
2. 有两层安全检查：
   - 检查 `currentIsPrimary` 必须为 `true`，否则抛出异常
   - 检查 `Authorization` Header 必须等于 `Bearer ${token}`，否则重定向到 YouTube
3. 使用 `zod` 解析请求体，期望 `{ key: string, cacheValue?: unknown }`
4. 根据 `cacheValue` 是否为 `undefined` 决定：
   - `undefined`: 执行 `cache.delete(key)`
   - 其他值: 执行 `cache.set(key, cacheValue)`
5. 返回 `{ success: true }`（JSON 响应）

**核验方式**:
- 读取 `app/routes/admin/cache/sqlite.server.ts`
- 确认这是 `action` 函数（Remix 约定）
- 确认安全检查逻辑
- 确认 Zod schema 和操作分支

---

### 4.4 转发机制中的待验证点（⚠️ 平台待验证）

| 待验证点 | 对缓存一致性的影响 | 验证方式 |
|----------|---------------------|----------|
| `getInternalInstanceDomain()` 的返回格式 | 构建的 URL 是否正确？是否可达？ | `litefs-js` 源码 |
| Fly.io 内部网络 `.internal` 域名的可靠性 | 网络分区时会怎样？ | Fly.io 文档/实测 |
| `INTERNAL_COMMAND_TOKEN` 的安全性 | 如何生成？如何轮换？ | 项目部署文档 |
| Primary 实例切换时的行为 | 转发过程中 Primary 变了会怎样？ | `litefs-js` 源码 + Fly.io 文档 |

**如果这些假设不成立的风险**:

```
假设: getInternalInstanceDomain() 返回正确的内部域名，网络可靠

风险场景 1: 域名不可达或网络分区
    → 后果: fetch() 失败，缓存更新丢失
    → 影响: 各实例缓存不一致

风险场景 2: INTERNAL_COMMAND_TOKEN 不匹配（不同实例配置不同）
    → 后果: Primary 返回 302 重定向到 YouTube
    → 影响: fetch() 认为 "ok"（302 是 redirect，response.ok 为 false），记录错误日志
    → 实际: 缓存更新失败

风险场景 3: 转发过程中 Primary 发生切换
    → 后果: 请求到达的实例已不再是 Primary
    → 影响: action() 抛出异常，返回 500，缓存更新失败
```

---

## 5. 缓存转发失败的一致性影响

### 5.1 当前错误处理（✅ 仓内代码可证实）

**文件位置**: `app/utils/cache.server.ts:164-176`

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
// ⚠️ 没有 .catch() 块！
```

**核验结论**:
1. 使用 `void` 关键字：不等待 Promise 完成，不阻塞请求
2. 只有 `.then()` 处理 `Response` 对象
3. 检查 `response.ok`，如果为 `false` 则记录错误日志
4. **没有 `.catch()`**：如果 `fetch()` 本身抛出（如网络错误、DNS 失败），会产生未处理的 Promise 拒绝
5. 无论成功与否，都不会影响当前请求的处理

**核验方式**:
- 读取 `app/utils/cache.server.ts` 中的 `cache.set` 和 `cache.delete`
- 确认 Promise 链的结构
- 确认缺少 `.catch()`

---

### 5.2 失败场景的代码级影响（✅ 仓内代码可证实）

| 失败类型 | 代码行为 | 是否可证实 |
|----------|----------|------------|
| HTTP 非 2xx 响应 | 进入 `.then()`，`response.ok` 为 `false`，记录 `console.error` | ✅ |
| `fetch()` 抛出（网络错误） | 无 `.catch()`，产生未处理的 Promise 拒绝 | ✅ |
| 进程崩溃/重启 | 内存中的 `void` Promise 丢失，更新永远丢失 | ⚠️ 平台推断 |

---

### 5.3 对缓存一致性的影响分析

#### 已知事实（✅ 仓内代码可证实）

1. **两层缓存架构**:
   - `lruCache`: 内存缓存，每个实例独立
   - `cache`: SQLite 缓存，通过 LiteFS 同步

2. **写入路径**:
   - LRU Cache: 代码中**没有**在 `cache.set()` 时更新 LRU Cache
   - SQLite Cache: Primary 直接写入，Replica 转发

3. **读取路径**:
   - `cache.get()` 只查询 SQLite，不涉及 LRU
   - `lruCache` 和 `cache` 是两个独立的缓存对象

#### 待验证的影响（⚠️ 平台待验证）

| 问题 | 影响 | 需要验证 |
|------|------|----------|
| Replica 的 LRU Cache 何时更新？ | 如果 LRU 在转发前被更新，失败时会不一致 | 搜索 `lruCache.set` 调用位置 |
| LiteFS 同步延迟是多少？ | 即使转发成功，其他 Replica 多久能看到？ | LiteFS 文档/实测 |
| 缓存 TTL 是多少？ | 不一致状态会持续多久？ | `@epic-web/cachified` 配置 |

---

## 6. 待验证点对关键机制的影响分析

### 6.1 对 OAuth 主实例保障的影响

#### 已知保障（✅ 仓内代码可证实）

```
代码实现:
OAuth Callback Loader
    │
    ├──▶ await ensurePrimary()  ◀── 第一行调用
    │
    └──▶ prisma.connection.create()
    └──▶ prisma.session.create()
    └──▶ ... 其他写入操作
```

**代码保证**:
- `ensurePrimary()` 在任何写入操作之前调用
- 如果 `ensurePrimary()` 抛出异常，后续代码不会执行
- 注释明确说明这是为了 "avoid writing to a read-only replica"

#### 待验证风险（⚠️ 平台待验证）

**风险 1: `ensurePrimary()` 的返回值处理**

```typescript
// 当前代码
await ensurePrimary()
// 没有使用返回值！
```

**问题**:
- 如果 `ensurePrimary()` 返回一个 `Response` 对象，这个返回值被忽略了
- Remix loader 的返回值会被用作 HTTP 响应
- 如果 `ensurePrimary()` 返回 Response，但代码没有 `return`，会发生什么？

**可能的情况**:
1. `ensurePrimary()` 抛出异常 → 行为正确（阻止后续执行）
2. `ensurePrimary()` 返回 Response，但代码没有 return → Response 被忽略，继续执行 → **危险！**
3. `ensurePrimary()` 什么都不返回（`undefined`）→ 继续执行

**验证方式**: 查看 `litefs-js/remix` 源码中 `ensurePrimary` 的实现。

---

**风险 2: Replay Response 的实际效果**

**假设的工作流程**:
```
1. 请求到达 Replica 实例
2. loader 调用 ensurePrimary()
3. ensurePrimary() 检测到不是 Primary，返回 Replay Response
4. Remix 使用这个 Response 作为 HTTP 响应
5. Fly.io 代理检测到 Replay Response
6. 代理将原始请求转发到 Primary 实例
7. Primary 实例处理请求，返回正常响应
8. 用户收到正常响应
```

**每个环节的风险**:

| 环节 | 如果不成立的后果 | 验证方式 |
|------|------------------|----------|
| Step 3: `ensurePrimary()` 确实返回特殊 Response | 可能抛出异常或什么都不做 | `litefs-js` 源码 |
| Step 4: Remix 正确处理这个 Response | 可能需要 `return ensurePrimary()` | Remix 文档 |
| Step 5: Fly.io 代理识别这个 Response | 可能直接返回给用户 | Fly.io 文档 |
| Step 6: 代理正确重放请求 | 可能丢失 Body 或 Headers | Fly.io 文档 |

---

**风险 3: 并发和竞态条件**

**场景**:
1. 用户点击"使用 GitHub 登录"
2. GitHub 重定向到 `/auth/github/callback`
3. 请求被路由到 **Replica** 实例
4. `ensurePrimary()` 检测到不是 Primary
5. 此时 **Primary 实例崩溃**
6. LiteFS/Consul 选举新的 Primary
7. Fly.io 代理转发请求到**旧的 Primary**（已崩溃）

**后果**:
- OAuth 回调失败
- 用户无法登录
- 可能需要用户重试

**缓解措施**:
- Fly.io 应该有健康检查和自动重试
- 但这是平台特性，代码无法控制

---

### 6.2 对缓存一致性判断的影响

#### 已知架构（✅ 仓内代码可证实）

```
缓存写入流程:

┌─────────────────────────────────────────────────────────────────┐
│                     Replica 实例                                  │
│                                                                   │
│   cache.set(key, value)                                          │
│          │                                                        │
│          ├──▶ getInstanceInfo()                                  │
│          │       │                                                │
│          │       └──▶ currentIsPrimary = false                  │
│          │                                                        │
│          └──▶ updatePrimaryCacheValue() ◀── void，不等待         │
│                  │                                                │
│                  └──▶ fetch(primary-domain/admin/cache/sqlite)  │
│                          │                                        │
│                          └──▶ 可能失败（网络、Token、实例切换）   │
└─────────────────────────────────────────────────────────────────┘
```

#### 待验证的一致性问题

**问题 1: 转发失败时的数据一致性**

**代码事实**:
- `void updatePrimaryCacheValue(...)` - 不等待
- 只有 `.then()` 检查 `response.ok` 并记录日志
- 没有重试机制
- 没有持久化队列

**待验证的影响**:

| 失败场景 | 对一致性的影响 | 是否可恢复 |
|----------|----------------|------------|
| 暂时性网络失败 | 缓存更新丢失 | 不可恢复（没有重试） |
| Primary 短暂不可用 | 缓存更新丢失 | 不可恢复 |
| Token 配置错误 | 所有缓存更新失败 | 可恢复（修复配置后新的更新正常，但丢失的无法恢复） |
| 实例进程崩溃 | 内存中的 Promise 丢失 | 不可恢复 |

---

**问题 2: LRU Cache 与 SQLite Cache 的一致性**

**代码事实**:
- `lruCache` 和 `cache` 是两个独立的缓存对象
- 搜索 `lruCache.set` 或 `cache.set` 调用位置

**待验证**:
- `@epic-web/cachified` 的 `cachified()` 函数如何协调这两个缓存？
- 更新 `cache`（SQLite）时，`lruCache` 是否会被更新？
- 如果转发失败，Replica 的 `lruCache` 状态如何？

---

**问题 3: 多实例并发更新同一个 Key**

**场景**:
1. Replica A 尝试更新 `cache.set("config", valueA)`
2. Replica B 同时尝试更新 `cache.set("config", valueB)`
3. 两个请求都转发到 Primary

**待验证**:
- 转发的顺序如何保证？
- Primary 上的执行顺序是怎样的？
- SQLite 的锁机制如何处理并发写入？

**代码事实**:
- `updatePrimaryCacheValue()` 是 fire-and-forget
- 没有任何顺序保证
- 最终状态取决于哪个请求最后到达 Primary

---

### 6.3 风险汇总表

| 风险领域 | 风险描述 | 可能性 | 影响 | 风险等级 | 验证方式 |
|----------|----------|--------|------|----------|----------|
| OAuth | `ensurePrimary()` 返回 Response 但没有 `return` | 中 | 高 | **高** | `litefs-js` 源码 |
| OAuth | Fly.io 代理不处理 Replay Response | 低 | 高 | **中** | Fly.io 文档 |
| OAuth | 请求重放时丢失关键数据 | 低 | 高 | **中** | Fly.io 文档 |
| 缓存 | 缺少 `.catch()` 导致未处理拒绝 | 高 | 中 | **中** | 代码已证实 |
| 缓存 | 无重试机制导致更新丢失 | 高 | 中 | **中** | 代码已证实 |
| 缓存 | 无持久化队列导致崩溃时丢失 | 中 | 中 | **中** | 平台推断 |
| 缓存 | LRU 与 SQLite 不一致 | 中 | 低 | **低** | `@epic-web/cachified` 源码 |
| 缓存 | 并发更新顺序无保证 | 中 | 低 | **低** | 架构设计 |

---

## 7. 总结与建议

### 7.1 已证实的架构事实（✅ 仓内代码可证实）

| 组件 | 行为 | 文件位置 |
|------|------|----------|
| 实例信息获取 | 通过 `getInstanceInfo()` 和 `getInstanceInfoSync()` | `litefs.server.ts` |
| 返回值字段 | `currentIsPrimary`、`currentInstance`、`primaryInstance` | `cache.server.ts`、`sqlite.$cacheKey.ts` |
| OAuth 保障 | `ensurePrimary()` 在写入前调用 | `callback.ts:34` |
| 缓存转发 | Replica 通过 HTTP POST 到 Primary 的 `/admin/cache/sqlite` | `cache.server.ts`、`sqlite.server.ts` |
| 认证机制 | 使用 `INTERNAL_COMMAND_TOKEN` 作为 Bearer Token | `sqlite.server.ts` |
| 响应头注入 | 所有响应注入 `fly-region`、`fly-app`、`fly-primary-instance`、`fly-instance` | `entry.server.tsx` |
| 错误处理 | 缓存转发只有 `.then()`，没有 `.catch()` | `cache.server.ts` |

---

### 7.2 待验证的关键假设（⚠️ 平台待验证）

| 假设 | 验证优先级 | 建议验证方式 |
|------|------------|--------------|
| `ensurePrimary()` 的返回值和行为 | **P0** | 查看 `litefs-js/remix` 源码 |
| Remix 如何处理 loader 的返回值 | **P0** | 查看 Remix 文档或源码 |
| `getInternalInstanceDomain()` 的实现 | **P1** | 查看 `litefs-js` 源码 |
| `INTERNAL_COMMAND_TOKEN` 的配置方式 | **P1** | 查看部署文档 |
| Fly.io Replay Response 机制 | **P1** | 查看 Fly.io 官方文档 |
| LiteFS 同步延迟和一致性保证 | **P2** | 查看 LiteFS 文档 |

---

### 7.3 代码改进建议（✅ 可在仓内实现）

#### 建议 1: 完善缓存转发的错误处理（高优先级）

**问题**: 当前代码缺少 `.catch()`，`fetch()` 抛出时会有未处理的 Promise 拒绝。

**改进代码**:
```typescript
// cache.server.ts 中的 cache.set()
async set(key, entry) {
	const { currentIsPrimary, primaryInstance } = await getInstanceInfo()

	if (currentIsPrimary) {
		const value = JSON.stringify(entry.value, bufferReplacer)
		setStatement.run(key, value, JSON.stringify(entry.metadata))
	} else {
		// 移除 void，添加 .catch()
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
			// 捕获网络错误等
			console.error(
				`Network error updating cache value for key "${key}" on primary instance (${primaryInstance})`,
				{ error },
			)
		})
	}
}
```

**核验点**: 这是纯代码改进，可在仓内实现和验证。

---

#### 建议 2: 确认 ensurePrimary() 的正确用法（高优先级）

**问题**: 当前代码是 `await ensurePrimary()` 但没有 `return`。

**需要验证**:
```typescript
// 选项 A: 如果 ensurePrimary() 只是检查并抛出
await ensurePrimary()  // ✅ 正确

// 选项 B: 如果 ensurePrimary() 可能返回 Response
return ensurePrimary()  // ❌ 当前代码没有 return！

// 选项 C: 需要手动处理
const replayResponse = await ensurePrimary()
if (replayResponse) {
	throw replayResponse  // 或 return
}
```

**建议**:
1. 查看 `litefs-js/remix` 源码或文档
2. 根据实际行为调整代码

---

#### 建议 3: 添加重试机制（中优先级）

对于缓存转发，可以添加指数退避重试：

```typescript
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
			if (response.ok || (response.status >= 400 && response.status < 500)) {
				// 成功或客户端错误（不需要重试）
				return response
			}
			// 5xx 错误，可以重试
			lastError = new Error(`HTTP ${response.status}`)
		} catch (error) {
			lastError = error as Error
		}
		
		// 指数退避
		const delay = Math.pow(2, attempt) * 100
		await new Promise(resolve => setTimeout(resolve, delay))
	}
	
	throw lastError ?? new Error('Max retries exceeded')
}
```

---

### 7.4 验证检查清单

在部署或排查问题前，建议验证以下内容：

| 检查项 | 验证方法 | 预期结果 |
|--------|----------|----------|
| `FLY_REGION` 环境变量 | `console.log(process.env.FLY_REGION)` | 实例所在区域 |
| `FLY_APP_NAME` 环境变量 | `console.log(process.env.FLY_APP_NAME)` | 应用名称 |
| `INTERNAL_COMMAND_TOKEN` | 检查所有实例配置 | 所有实例必须相同 |
| `getInternalInstanceDomain()` 返回值 | 临时添加日志 | 可访问的内部 URL |
| `/admin/cache/sqlite` 可达性 | 从 Replica 实例 curl | 返回 200 或 401 |
| 响应头注入 | 浏览器 DevTools | 包含 `fly-*` headers |

---

## 附录：代码引用速查

| 功能 | 文件位置 | 行号 | 状态 |
|------|----------|------|------|
| `ensurePrimary()` 导入 | `app/utils/litefs.server.ts` | 17 | ✅ |
| `getInstanceInfo()` 导入 | `app/utils/litefs.server.ts` | 4-14 | ✅ |
| `ensurePrimary()` 调用（OAuth） | `app/routes/_auth/auth.$provider/callback.ts` | 34 | ✅ |
| 响应头注入 | `app/entry.server.tsx` | 32-36, 117-120 | ✅ |
| 缓存 `set()` 转发 | `app/utils/cache.server.ts` | 158-178 | ✅ |
| 缓存 `delete()` 转发 | `app/utils/cache.server.ts` | 179-198 | ✅ |
| `updatePrimaryCacheValue()` | `app/routes/admin/cache/sqlite.server.ts` | 10-33 | ✅ |
| Primary 接收端点 `action()` | `app/routes/admin/cache/sqlite.server.ts` | 35-59 | ✅ |
| `ensureInstance()` 使用 | `app/routes/admin/cache/sqlite.$cacheKey.ts` | 18 | ✅ |
| LiteFS 配置 | `other/litefs.yml` | 全文 | ⚠️ 配置文件 |

> 状态说明:
> - ✅ **仓内代码可证实**: 可直接读取代码验证
> - ⚠️ **配置文件**: 配置项本身可证实，但效果依赖平台
