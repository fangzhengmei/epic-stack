# Server-Timing、Prisma 慢查询与 Sentry 性能监控分析

## 目录

- [Server-Timing 实现](#server-timing-实现)
  - [核心工具：timing.server.ts](#核心工具timingserverts)
  - [Root Loader 中的使用](#root-loader-中的使用)
  - [Cache Provider 中的使用](#cache-provider-中的使用)
  - [SSR Render 中的使用](#ssr-render-中的使用)
- [Prisma 慢查询记录](#prisma-慢查询记录)
- [Sentry 配置](#sentry-配置)
  - [服务端 Sentry 配置](#服务端-sentry-配置)
  - [客户端 Sentry 配置](#客户端-sentry-配置)
  - [入口文件初始化](#入口文件初始化)
  - [环境变量配置](#环境变量配置)

---

## Server-Timing 实现

### 核心工具：timing.server.ts

**文件位置**: `app/utils/timing.server.ts`

这是整个 Server-Timing 系统的核心工具库，提供了计时数据结构、计时器创建、函数执行时间包装、响应头生成等功能。

#### 1. Timings 数据结构

```typescript
export type Timings = Record<
	string,
	Array<
		{ desc?: string } & (
			| { time: number; start?: never }
			| { time?: never; start: number }
		)
	>
>
```

`Timings` 是一个记录类型，键为计时类型（如 `'root loader'`、`'getUserId'`），值为计时信息数组。每个计时项可以是：
- **start 模式**：记录开始时间戳（用于 `makeTimings`）
- **time 模式**：记录已计算的耗时（用于 `createTimer` + `end`）

#### 2. makeTimings - 创建计时对象

```typescript
export function makeTimings(type: string, desc?: string) {
	const timings: Timings = {
		[type]: [{ desc, start: performance.now() }],
	}
	Object.defineProperty(timings, 'toString', {
		value: function () {
			return getServerTimeHeader(timings)
		},
		enumerable: false,
	})
	return timings
}
```

**功能说明**：
- 创建一个新的 `Timings` 对象，初始包含一个指定类型的计时项
- 使用 `performance.now()` 记录高精度开始时间戳
- 为 `timings` 对象添加自定义 `toString` 方法，使其可以直接转换为 `Server-Timing` 响应头格式
- `toString` 方法被设置为不可枚举，避免遍历属性时出现

**使用场景**：
- 在请求开始时创建，用于追踪整个请求生命周期内的各个阶段耗时
- 例如：`root.tsx` 中的 `const timings = makeTimings('root loader')`
- 例如：`entry.server.tsx` 中的 `const timings = makeTimings('render', 'renderToPipeableStream')`

#### 3. createTimer - 创建独立计时器

```typescript
function createTimer(type: string, desc?: string) {
	const start = performance.now()
	return {
		end(timings: Timings) {
			let timingType = timings[type]
			if (!timingType) {
				timingType = timings[type] = []
			}
			timingType.push({ desc, time: performance.now() - start })
		},
	}
}
```

**功能说明**：
- 创建一个独立的计时器，立即记录开始时间
- 返回一个包含 `end` 方法的对象
- 调用 `end(timings)` 时：
  1. 计算从创建到 `end` 调用的时间差（`performance.now() - start`）
  2. 如果 `timings` 中不存在该类型的数组，则创建
  3. 将耗时数据推入对应类型的数组中

**设计特点**：
- 内部函数，不直接导出
- 与 `time()` 函数配合使用，实现自动计时
- 也可在 `cachifiedTimingReporter` 中独立使用

#### 4. time - 函数执行时间包装器

```typescript
export async function time<ReturnType>(
	fn: Promise<ReturnType> | (() => ReturnType | Promise<ReturnType>),
	{
		type,
		desc,
		timings,
	}: {
		type: string
		desc?: string
		timings?: Timings
	},
): Promise<ReturnType> {
	const timer = createTimer(type, desc)
	const promise = typeof fn === 'function' ? fn() : fn
	if (!timings) return promise

	const result = await promise
	timer.end(timings)
	return result
}
```

**功能说明**：
- 这是最常用的计时 API，用于包装异步函数或 Promise 的执行
- **参数说明**：
  - `fn`：可以是一个函数（同步或异步），也可以是一个 Promise 对象
  - `type`：计时类型标识（如 `'getUserId'`、`'find user'`）
  - `desc`：可选的描述文本，用于更清晰地说明计时内容
  - `timings`：可选的 `Timings` 对象，如果不传则不记录计时

**执行流程**：
1. 调用 `createTimer()` 创建计时器，立即记录开始时间
2. 如果 `fn` 是函数则执行它获取 Promise，否则直接使用传入的 Promise
3. 如果没有传入 `timings`，直接返回 Promise（跳过计时）
4. 否则等待 Promise 完成
5. 调用 `timer.end(timings)` 计算并记录耗时
6. 返回函数执行结果

**使用示例**：
```typescript
const userId = await time(() => getUserId(request), {
	timings,
	type: 'getUserId',
	desc: 'getUserId in root',
})
```

**设计亮点**：
- 支持多种传入方式（函数或 Promise），使用灵活
- `timings` 可选，方便在不需要计时的场景复用代码
- 不影响原函数的返回值和错误传播（错误会正常抛出）

#### 5. getServerTimeHeader - 生成 Server-Timing 响应头

```typescript
export function getServerTimeHeader(timings?: Timings) {
	if (!timings) return ''
	return Object.entries(timings)
		.map(([key, timingInfos]) => {
			const dur = timingInfos
				.reduce((acc, timingInfo) => {
					const time = timingInfo.time ?? performance.now() - timingInfo.start
					return acc + time
				}, 0)
				.toFixed(1)
			const desc = timingInfos
				.map((t) => t.desc)
				.filter(Boolean)
				.join(' & ')
			return [
				key.replaceAll(/(:| |@|=|;|,|\/|\\)/g, '_'),
				desc ? `desc=${JSON.stringify(desc)}` : null,
				`dur=${dur}`,
			]
				.filter(Boolean)
				.join(';')
		})
		.join(',')
}
```

**功能说明**：
- 将 `Timings` 对象转换为符合 [W3C Server-Timing](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Server-Timing) 规范的响应头字符串

**处理逻辑**：

1. **类型键清理**：
   ```typescript
   key.replaceAll(/(:| |@|=|;|,|\/|\\)/g, '_')
   ```
   - 替换特殊字符为下划线，确保符合 Server-Timing 规范
   - 特殊字符包括：`:`、空格、`@`、`=`、`;`、`,`、`/`、`\`

2. **耗时计算**：
   ```typescript
   const dur = timingInfos
     .reduce((acc, timingInfo) => {
       const time = timingInfo.time ?? performance.now() - timingInfo.start
       return acc + time
     }, 0)
     .toFixed(1)
   ```
   - 同一类型的多个计时项会被累加（支持多次调用同一类型的 `time()`）
   - 如果计时项是 `start` 模式（只记录了开始时间），则实时计算当前耗时
   - 如果是 `time` 模式（已计算好耗时），直接使用
   - 结果保留 1 位小数

3. **描述合并**：
   ```typescript
   const desc = timingInfos
     .map((t) => t.desc)
     .filter(Boolean)
     .join(' & ')
   ```
   - 同一类型的多个描述用 ` & ` 连接
   - 空描述会被过滤掉

4. **格式拼接**：
   - 格式：`类型;desc="描述";dur=耗时`
   - 多个计时项用逗号分隔
   - `desc` 字段只有在有描述时才包含

**输出示例**：
```
root_loader;desc="root loader";dur=45.2,getUserId;desc="getUserId in root";dur=12.1,find_user;desc="find user in root";dur=28.5
```

#### 6. combineServerTimings - 合并响应头

```typescript
export function combineServerTimings(headers1: Headers, headers2: Headers) {
	const newHeaders = new Headers(headers1)
	newHeaders.append('Server-Timing', headers2.get('Server-Timing') ?? '')
	return newHeaders.get('Server-Timing') ?? ''
}
```

**功能说明**：
- 合并两个 Headers 对象中的 `Server-Timing` 头
- 使用 `Headers.append()` 方法，多个值会用逗号分隔
- 如果第二个 headers 中没有 `Server-Timing`，则使用空字符串

**使用场景**：
- 当需要合并来自不同来源的计时信息时使用
- 例如：子请求的响应头与主请求的响应头合并

#### 7. cachifiedTimingReporter - 缓存操作计时报告器

```typescript
export function cachifiedTimingReporter<Value>(
	timings?: Timings,
): undefined | CreateReporter<Value> {
	if (!timings) return

	return ({ key }) => {
		const cacheRetrievalTimer = createTimer(
			`cache:${key}`,
			`${key} cache retrieval`,
		)
		let getFreshValueTimer: ReturnType<typeof createTimer> | undefined
		return (event) => {
			switch (event.name) {
				case 'getFreshValueStart':
					getFreshValueTimer = createTimer(
						`getFreshValue:${key}`,
						`request forced to wait for a fresh ${key} value`,
					)
					break
				case 'getFreshValueSuccess':
					getFreshValueTimer?.end(timings)
					break
				case 'done':
					cacheRetrievalTimer.end(timings)
					break
			}
		}
	}
}
```

**功能说明**：
- 为 `@epic-web/cachified` 库创建的自定义报告器（reporter）
- 用于记录缓存操作的各个阶段耗时
- 如果没有传入 `timings`，返回 `undefined`（cachified 会忽略 undefined reporter）

**工作原理**：

1. **创建报告器工厂**：
   - 函数接收 `timings` 对象，返回一个符合 `CreateReporter` 接口的函数
   - 这个工厂函数会在 cachified 操作开始时被调用

2. **初始化计时器**：
   ```typescript
   const cacheRetrievalTimer = createTimer(
     `cache:${key}`,
     `${key} cache retrieval`,
   )
   ```
   - 在 cachified 操作开始时立即创建 `cache:{key}` 计时器
   - 这个计时器会追踪整个缓存获取过程（包括缓存命中或未命中等）

3. **事件监听**：
   返回的事件处理函数监听 cachified 的三个关键事件：

   | 事件名 | 触发时机 | 处理逻辑 |
   |--------|----------|----------|
   | `getFreshValueStart` | 缓存未命中，开始获取新值时 | 创建 `getFreshValue:{key}` 计时器 |
   | `getFreshValueSuccess` | 成功获取新值后 | 结束 `getFreshValue` 计时器并记录耗时 |
   | `done` | 整个缓存操作完成时 | 结束 `cacheRetrieval` 计时器并记录总耗时 |

**记录的计时类型**：

| 计时类型 | 说明 | 示例 |
|----------|------|------|
| `cache:{key}` | 整个缓存获取过程的总耗时 | `cache:user-profile` |
| `getFreshValue:{key}` | 缓存未命中时，获取新值的耗时 | `getFreshValue:user-profile` |

**在 cache.server.ts 中的集成**：
```typescript
export async function cachified<Value>(
	{
		timings,
		...options
	}: CachifiedOptions<Value> & {
		timings?: Timings
	},
	reporter: CreateReporter<Value> = verboseReporter<Value>(),
): Promise<Value> {
	return baseCachified(
		options,
		mergeReporters(cachifiedTimingReporter(timings), reporter),
	)
}
```

- 封装了原生的 `cachified` 函数，添加 `timings` 可选参数
- 使用 `mergeReporters` 将 `cachifiedTimingReporter` 与默认的 `verboseReporter` 合并
- 这样既保留了原有的日志输出，又添加了性能计时功能

---

### Root Loader 中的使用

**文件位置**: `app/root.tsx`

root loader 是应用的根路由加载器，负责处理全局数据加载。这里展示了如何在实际业务代码中使用 Server-Timing 系统。

#### 完整实现代码

```typescript
// 第 40 行：导入计时工具
import { makeTimings, time } from './utils/timing.server.ts'

export async function loader({ request }: Route.LoaderArgs) {
	// 第 72 行：创建根 loader 的计时对象
	const timings = makeTimings('root loader')

	// 第 73-77 行：使用 time() 包装 getUserId 调用
	const userId = await time(() => getUserId(request), {
		timings,
		type: 'getUserId',
		desc: 'getUserId in root',
	})

	// 第 79-101 行：使用 time() 包装 Prisma 查询
	const user = userId
		? await time(
				() =>
					prisma.user.findUnique({
						select: {
							id: true,
							name: true,
							username: true,
							image: { select: { objectKey: true } },
							roles: {
								select: {
									name: true,
									permissions: {
										select: { entity: true, action: true, access: true },
									},
								},
							},
						},
						where: { id: userId },
					}),
				{ timings, type: 'find user', desc: 'find user in root' },
			)
		: null

	// ... 其他逻辑 ...

	// 第 111-132 行：返回响应，包含 Server-Timing 头
	return data(
		{
			user,
			requestInfo: { /* ... */ },
			ENV: getEnv(),
			toast,
			honeyProps,
		},
		{
			headers: combineHeaders(
				// 第 128 行：关键！将 timings 转换为 Server-Timing 头
				{ 'Server-Timing': timings.toString() },
				toastHeaders,
			),
		},
	)
}
```

#### 执行流程分析

1. **创建计时对象**（第 72 行）：
   ```typescript
   const timings = makeTimings('root loader')
   ```
   - 在 loader 开始时立即创建 `timings` 对象
   - 类型为 `'root loader'`，记录整个 loader 的执行起点

2. **包装 getUserId 调用**（第 73-77 行）：
   ```typescript
   const userId = await time(() => getUserId(request), {
     timings,
     type: 'getUserId',
     desc: 'getUserId in root',
   })
   ```
   - 包装 `getUserId(request)` 函数调用
   - 计时类型：`'getUserId'`
   - 描述：`'getUserId in root'`
   - 耗时会被自动记录到 `timings` 对象中

3. **包装 Prisma 查询**（第 79-101 行）：
   ```typescript
   const user = userId
     ? await time(
         () => prisma.user.findUnique({ /* ... */ }),
         { timings, type: 'find user', desc: 'find user in root' },
       )
     : null
   ```
   - 只在 `userId` 存在时执行查询
   - 计时类型：`'find user'`
   - 描述：`'find user in root'`
   - 这样可以精确追踪数据库查询的耗时

4. **生成响应头**（第 128 行）：
   ```typescript
   { 'Server-Timing': timings.toString() }
   ```
   - 调用 `timings.toString()` 触发 `getServerTimeHeader(timings)`
   - 将所有记录的计时信息转换为 `Server-Timing` 响应头
   - 通过 `combineHeaders` 与其他响应头（如 toastHeaders）合并

#### 记录的计时项

在 root loader 执行过程中，会记录以下计时项：

| 计时类型 | 描述 | 记录时机 |
|----------|------|----------|
| `root loader` | （无描述，只有开始时间） | `makeTimings()` 调用时 |
| `getUserId` | `getUserId in root` | `time()` 包装的函数完成时 |
| `find user` | `find user in root` | Prisma 查询完成时（如果执行） |

#### 响应头示例

假设各阶段耗时如下：
- root loader 总耗时：100ms
- getUserId：15ms
- find user：45ms

生成的 `Server-Timing` 响应头：
```
root_loader;dur=100.0,getUserId;desc="getUserId in root";dur=15.0,find_user;desc="find user in root";dur=45.0
```

#### 设计要点

1. **渐进式计时**：
   - `makeTimings('root loader')` 在最开始创建，记录整个 loader 的起点
   - 后续的 `time()` 调用会追加更多细粒度的计时项

2. **条件执行处理**：
   - `find user` 计时只在 `userId` 存在时才会记录
   - 这体现了 `time()` 函数的灵活性，可以根据业务逻辑有条件地执行

3. **类型命名规范**：
   - 使用语义化的类型名：`'getUserId'`、`'find user'`
   - 描述中包含上下文：`'in root'`，方便在多个 loader 中区分

4. **与响应头集成**：
   - 通过 `timings.toString()` 无缝集成到响应头
   - 使用 `combineHeaders` 确保与其他响应头正确合并

---

### Cache Provider 中的使用

**文件位置**: `app/utils/cache.server.ts`

cache.server.ts 实现了基于 SQLite 和 LRU 的双层缓存系统，并集成了 Server-Timing 来追踪缓存操作的性能。

#### 完整实现代码

```typescript
// 第 1-20 行：导入依赖
import fs from 'node:fs'
import path from 'node:path'
import { DatabaseSync } from 'node:sqlite'
import {
	cachified as baseCachified,
	verboseReporter,
	mergeReporters,
	type CacheEntry,
	type Cache as CachifiedCache,
	type CachifiedOptions,
	type Cache,
	totalTtl,
	type CreateReporter,
} from '@epic-web/cachified'
import { remember } from '@epic-web/remember'
import { LRUCache } from 'lru-cache'
import { z } from 'zod'
import { updatePrimaryCacheValue } from '#app/routes/admin/cache/sqlite.server.ts'
import { getInstanceInfo, getInstanceInfoSync } from './litefs.server.ts'
// 第 20 行：关键导入 - 计时报告器和类型
import { cachifiedTimingReporter, type Timings } from './timing.server.ts'

// ... 缓存数据库初始化、LRU 缓存配置等代码 ...

// 第 218-230 行：封装的 cachified 函数
export async function cachified<Value>(
	{
		timings,
		...options
	}: CachifiedOptions<Value> & {
		timings?: Timings
	},
	reporter: CreateReporter<Value> = verboseReporter<Value>(),
): Promise<Value> {
	return baseCachified(
		options,
		// 第 228-229 行：合并计时报告器和默认报告器
		mergeReporters(cachifiedTimingReporter(timings), reporter),
	)
}
```

#### 封装的 cachified 函数分析

这个函数是对 `@epic-web/cachified` 库的轻量级封装，添加了 Server-Timing 集成：

1. **扩展参数类型**：
   ```typescript
   {
     timings,
     ...options
   }: CachifiedOptions<Value> & {
     timings?: Timings
   }
   ```
   - 继承 `CachifiedOptions` 的所有参数
   - 添加可选的 `timings` 参数，类型为 `Timings`
   - 使用解构分离 `timings` 和其他选项

2. **默认报告器**：
   ```typescript
   reporter: CreateReporter<Value> = verboseReporter<Value>()
   ```
   - 如果调用者没有传入 `reporter`，使用 `verboseReporter` 作为默认
   - `verboseReporter` 会输出详细的缓存操作日志（命中、未命中等）

3. **报告器合并**：
   ```typescript
   mergeReporters(cachifiedTimingReporter(timings), reporter)
   ```
   - 使用 `mergeReporters` 将两个报告器合并
   - 第一个是 `cachifiedTimingReporter(timings)`：负责记录性能计时
   - 第二个是 `reporter`：负责日志输出（默认 verboseReporter 或自定义）
   - 合并后，缓存事件会同时发送给两个报告器

#### 使用示例

在业务代码中使用带有计时功能的 cachified：

```typescript
// 假设在某个 loader 中
import { cachified } from './utils/cache.server.ts'
import { makeTimings } from './utils/timing.server.ts'

export async function loader({ request }) {
	const timings = makeTimings('notes loader')

	const notes = await cachified({
		key: 'user-notes',
		cache: lruCache,
		ttl: 1000 * 60 * 5, // 5 分钟
		timings, // 传入 timings 对象
		getFreshValue: async () => {
			return prisma.note.findMany({
				where: { userId: getUserId(request) },
			})
		},
	})

	// 返回响应时会包含缓存操作的计时
	return data({ notes }, { headers: { 'Server-Timing': timings.toString() } })
}
```

#### 记录的计时项

根据 `cachifiedTimingReporter` 的实现，缓存操作会记录以下计时项：

| 计时类型 | 描述 | 触发条件 | 说明 |
|----------|------|----------|------|
| `cache:{key}` | `{key} cache retrieval` | 所有缓存操作 | 整个缓存获取过程的总耗时 |
| `getFreshValue:{key}` | `request forced to wait for a fresh {key} value` | 缓存未命中时 | 获取新值（通常是数据库查询）的耗时 |

#### 响应头示例

**场景 1：缓存命中**
- `cache:user-notes` 耗时：5ms
- 没有 `getFreshValue` 计时（因为缓存命中）

响应头：
```
notes_loader;dur=10.0,cache_user_notes;desc="user-notes cache retrieval";dur=5.0
```

**场景 2：缓存未命中**
- `cache:user-notes` 总耗时：150ms
- `getFreshValue:user-notes` 耗时：140ms（数据库查询）

响应头：
```
notes_loader;dur=160.0,cache_user_notes;desc="user-notes cache retrieval";dur=150.0,getFreshValue_user_notes;desc="request forced to wait for a fresh user-notes value";dur=140.0
```

#### 设计要点

1. **非侵入式设计**：
   - `timings` 参数是可选的，不传则不记录计时
   - 这样可以在开发环境启用计时，生产环境根据需要选择是否启用

2. **报告器合并策略**：
   - 使用 `mergeReporters` 而不是替换报告器
   - 确保原有的日志功能（verboseReporter）不受影响
   - 计时报告器和日志报告器各司其职

3. **细粒度计时**：
   - 不仅记录整个缓存操作的总耗时
   - 还单独记录 `getFreshValue` 的耗时（缓存未命中时的关键路径）
   - 这有助于分析缓存命中率和数据库查询性能

4. **类型安全**：
   - 使用 TypeScript 泛型 `cachified<Value>` 确保类型安全
   - `Timings` 类型明确，避免运行时错误

---

### SSR Render 中的使用

**文件位置**: `app/entry.server.tsx`

entry.server.tsx 是 React Router 应用的服务端渲染入口，负责处理 HTTP 请求、渲染 React 组件为 HTML，并返回响应。

#### 完整实现代码

```typescript
// 第 1-18 行：导入依赖
import crypto from 'node:crypto'
import { PassThrough } from 'node:stream'
import { styleText } from 'node:util'
import { contentSecurity } from '@nichtsam/helmet/content'
import { createReadableStreamFromReadable } from '@react-router/node'
import * as Sentry from '@sentry/react-router'
import { isbot } from 'isbot'
import { renderToPipeableStream } from 'react-dom/server'
import {
	ServerRouter,
	type LoaderFunctionArgs,
	type ActionFunctionArgs,
	type HandleDocumentRequestFunction,
} from 'react-router'
import { getEnv, init } from './utils/env.server.ts'
import { getInstanceInfo } from './utils/litefs.server.ts'
import { NonceProvider } from './utils/nonce-provider.ts'
// 第 18 行：导入计时工具
import { makeTimings } from './utils/timing.server.ts'

// 第 20 行：流超时时间
export const streamTimeout = 5000

// 第 22-23 行：初始化环境变量
init()
global.ENV = getEnv()

const MODE = process.env.NODE_ENV ?? 'development'

type DocRequestArgs = Parameters<HandleDocumentRequestFunction>

// 第 29-113 行：主请求处理函数
export default async function handleRequest(...args: DocRequestArgs) {
	const [request, responseStatusCode, responseHeaders, reactRouterContext] =
		args
	const { currentInstance, primaryInstance } = await getInstanceInfo()

	// 设置 Fly.io 实例信息头
	responseHeaders.set('fly-region', process.env.FLY_REGION ?? 'unknown')
	responseHeaders.set('fly-app', process.env.FLY_APP_NAME ?? 'unknown')
	responseHeaders.set('fly-primary-instance', primaryInstance)
	responseHeaders.set('fly-instance', currentInstance)

	// 第 38-40 行：Sentry 性能分析头（生产环境）
	if (process.env.NODE_ENV === 'production' && process.env.SENTRY_DSN) {
		responseHeaders.append('Document-Policy', 'js-profiling')
	}

	// 第 42-44 行：根据 User-Agent 决定回调名称
	// 爬虫使用 onAllReady（等待所有内容就绪），普通用户使用 onShellReady（流式渲染）
	const callbackName = isbot(request.headers.get('user-agent'))
		? 'onAllReady'
		: 'onShellReady'

	// 第 46 行：生成 CSP nonce
	const nonce = crypto.randomBytes(16).toString('hex')

	// 第 47-112 行：返回 Promise，处理流式渲染
	return new Promise(async (resolve, reject) => {
		let didError = false

		// 第 49-51 行：关键！创建 SSR 渲染计时器
		// NOTE: this timing will only include things that are rendered in the shell
		// and will not include suspended components and deferred loaders
		const timings = makeTimings('render', 'renderToPipeableStream')

		// 第 53-109 行：调用 renderToPipeableStream
		const { pipe, abort } = renderToPipeableStream(
			<NonceProvider value={nonce}>
				<ServerRouter
					nonce={nonce}
					context={reactRouterContext}
					url={request.url}
				/>
			</NonceProvider>,
			{
				// 第 62-100 行：onShellReady / onAllReady 回调
				[callbackName]: () => {
					const body = new PassThrough()
					responseHeaders.set('Content-Type', 'text/html')

					// 第 65 行：关键！添加 Server-Timing 响应头
					responseHeaders.append('Server-Timing', timings.toString())

					// 第 67-91 行：设置 Content Security Policy
					contentSecurity(responseHeaders, {
						crossOriginEmbedderPolicy: false,
						contentSecurityPolicy: {
							reportOnly: true,
							directives: {
								fetch: {
									'connect-src': [
										MODE === 'development' ? 'ws:' : undefined,
										process.env.SENTRY_DSN ? '*.sentry.io' : undefined,
										"'self'",
									],
									'font-src': ["'self'"],
									'frame-src': ["'self'"],
									'img-src': ["'self'", 'data:'],
									'script-src': [
										"'strict-dynamic'",
										"'self'",
										`'nonce-${nonce}'`,
									],
									'script-src-attr': [`'nonce-${nonce}'`],
								},
							},
						},
					})

					// 第 93-99 行：解析 Promise，返回 Response
					resolve(
						new Response(createReadableStreamFromReadable(body), {
							headers: responseHeaders,
							status: didError ? 500 : responseStatusCode,
						}),
					)
					// 第 99 行：开始流式输出
					pipe(body)
				},
				// 第 101-103 行：错误处理回调
				onShellError: (err: unknown) => {
					reject(err)
				},
				onError: () => {
					didError = true
				},
				nonce,
			},
		)

		// 第 111 行：设置超时 abort
		setTimeout(abort, streamTimeout + 5000)
	})
}

// ... handleDataRequest 和 handleError 函数 ...
```

#### SSR 计时实现分析

1. **创建计时对象**（第 51 行）：
   ```typescript
   const timings = makeTimings('render', 'renderToPipeableStream')
   ```
   - 在 `renderToPipeableStream` 调用前创建 `timings` 对象
   - 类型：`'render'`
   - 描述：`'renderToPipeableStream'`
   - 记录的是从创建到 `onShellReady`（或 `onAllReady`）回调触发的时间

2. **重要注释说明**（第 49-50 行）：
   ```typescript
   // NOTE: this timing will only include things that are rendered in the shell
   // and will not include suspended components and deferred loaders
   ```
   - **关键提示**：这个计时**只包含 shell（外壳）的渲染时间**
   - **不包含**：
     - Suspended components（使用 `React.Suspense` 延迟加载的组件）
     - Deferred loaders（使用 `defer()` 的加载器数据）
   - 这是因为 `onShellReady` 回调在 shell 渲染完成后立即触发，而 suspended 内容会在后续流式传输

3. **添加响应头**（第 65 行）：
   ```typescript
   responseHeaders.append('Server-Timing', timings.toString())
   ```
   - 在 `onShellReady` 回调中调用
   - 使用 `append` 方法，保留可能已存在的 `Server-Timing` 头（例如来自 loader 的）
   - 这确保了 loader 计时和 render 计时都会出现在响应头中

#### 响应头合并示例

假设一次完整的请求流程：

1. **Root Loader 执行**：
   - `root_loader`：45ms
   - `getUserId`：10ms
   - `find user`：20ms

2. **SSR Render 执行**：
   - `render`：30ms

最终的 `Server-Timing` 响应头（由于使用 `append`，会合并）：
```
root_loader;dur=45.0,getUserId;desc="getUserId in root";dur=10.0,find_user;desc="find user in root";dur=20.0,render;desc="renderToPipeableStream";dur=30.0
```

#### 流式渲染与计时的关系

React 18 的 `renderToPipeableStream` 支持流式 SSR，这对计时有重要影响：

| 阶段 | 触发时机 | 是否计入 `render` 计时 |
|------|----------|------------------------|
| Shell 渲染完成 | `onShellReady` 回调 | **是** |
| Suspended 组件就绪 | 后续流式传输 | **否** |
| Deferred data 到达 | 后续流式传输 | **否** |

**这意味着**：
- `render` 计时反映的是**首屏 HTML 生成时间**
- 完整的页面加载时间还需要考虑客户端 hydration 和 suspended 内容的加载
- 如果需要追踪完整的渲染时间，需要结合前端监控（如 Sentry）

#### 与 Sentry 的集成（第 38-40 行）

```typescript
if (process.env.NODE_ENV === 'production' && process.env.SENTRY_DSN) {
	responseHeaders.append('Document-Policy', 'js-profiling')
}
```

- 在生产环境且配置了 Sentry DSN 时，添加 `Document-Policy: js-profiling` 头
- 这个头启用浏览器的 JavaScript Profiling API
- 配合 Sentry 的 `browserProfilingIntegration` 可以获取更详细的性能分析数据

#### 设计要点

1. **计时时机选择**：
   - 在 `renderToPipeableStream` 调用前创建计时器
   - 在 `onShellReady` 回调中结束（通过 `timings.toString()` 隐式计算）
   - 这准确反映了 shell 渲染的实际耗时

2. **响应头合并策略**：
   - 使用 `responseHeaders.append()` 而不是 `set()`
   - 确保来自 loader 的 `Server-Timing` 头不会被覆盖
   - 最终响应包含所有阶段的计时信息

3. **局限性明确**：
   - 代码注释明确说明了计时的范围（只包含 shell）
   - 这避免了对计时数据的误解
   - 为需要更完整数据的场景指明了方向（结合前端监控）

4. **条件启用**：
   - Sentry 相关的头只在生产环境且有 DSN 时才添加
   - 这符合"按需启用"的最佳实践，避免不必要的开销

---

## Prisma 慢查询记录

### 实现概述

**文件位置**: `app/utils/db.server.ts`

Prisma 慢查询记录通过 Prisma 客户端的事件监听机制实现，当数据库查询执行时间超过预设阈值时，会以彩色日志的形式输出到控制台，便于开发和性能调优。

### 完整实现代码

```typescript
// 第 1-4 行：导入依赖
import { styleText } from 'node:util'
import { remember } from '@epic-web/remember'
// Changed import due to issue: https://github.com/remix-run/react-router/pull/12644
import { PrismaClient } from '@prisma/client/index.js'

// 第 6-37 行：创建并导出 Prisma 客户端实例
export const prisma = remember('prisma', () => {
	// NOTE: if you change anything in this function you'll need to restart
	// the dev server to see your changes.

	// 第 10-11 行：慢查询阈值配置（可调整）
	// Feel free to change this log threshold to something that makes sense for you
	const logThreshold = 20

	// 第 13-19 行：创建 Prisma 客户端，配置日志选项
	const client = new PrismaClient({
		log: [
			{ level: 'query', emit: 'event' },  // 查询日志作为事件触发
			{ level: 'error', emit: 'stdout' }, // 错误日志输出到 stdout
			{ level: 'warn', emit: 'stdout' },  // 警告日志输出到 stdout
		],
	})

	// 第 20-34 行：监听查询事件，实现慢查询记录
	client.$on('query', async (e) => {
		// 第 21 行：快速返回 - 如果耗时低于阈值，不处理
		if (e.duration < logThreshold) return

		// 第 22-31 行：根据耗时确定日志颜色
		const color =
			e.duration < logThreshold * 1.1
				? 'green'      // < 22ms: 绿色（轻微超时）
				: e.duration < logThreshold * 1.2
					? 'blue'     // < 24ms: 蓝色
					: e.duration < logThreshold * 1.3
						? 'yellow' // < 26ms: 黄色（警告）
						: e.duration < logThreshold * 1.4
							? 'redBright' // < 28ms: 亮红色
							: 'red'       // >= 28ms: 红色（严重）

		// 第 32 行：格式化耗时显示（带颜色）
		const dur = styleText(color, `${e.duration}ms`)

		// 第 33 行：输出慢查询日志
		console.info(`prisma:query - ${dur} - ${e.query}`)
	})

	// 第 35 行：建立数据库连接
	void client.$connect()

	// 第 36 行：返回客户端实例
	return client
})
```

### 核心组件详解

#### 1. 单例模式 - remember 函数

```typescript
export const prisma = remember('prisma', () => {
	// ... 客户端创建逻辑 ...
})
```

- 使用 `@epic-web/remember` 的 `remember` 函数确保**全局单例**
- 键名为 `'prisma'`，保证在热重载等场景下不会重复创建客户端
- 这是 Node.js 服务端应用使用 Prisma 的最佳实践，避免连接池耗尽

#### 2. 慢查询阈值配置

```typescript
const logThreshold = 20
```

- **默认阈值**：20 毫秒
- **可调整性**：注释明确说明"Feel free to change this log threshold"
- **设计考虑**：
  - 20ms 是一个相对严格的阈值，适合开发环境发现性能问题
  - 生产环境可根据实际情况适当放宽
  - 修改此值需要重启开发服务器才能生效

#### 3. Prisma 客户端日志配置

```typescript
const client = new PrismaClient({
	log: [
		{ level: 'query', emit: 'event' },  // 关键配置！
		{ level: 'error', emit: 'stdout' },
		{ level: 'warn', emit: 'stdout' },
	],
})
```

**日志级别说明**：

| 级别 | emit 方式 | 说明 |
|------|-----------|------|
| `query` | `'event'` | **关键配置** - 查询日志作为事件触发，可通过 `$on('query')` 监听 |
| `error` | `'stdout'` | 错误日志直接输出到控制台 |
| `warn` | `'stdout'` | 警告日志直接输出到控制台 |

**为什么 `query` 使用 `emit: 'event'`**：
- 这样才能通过 `client.$on('query', callback)` 自定义处理
- 如果使用 `emit: 'stdout'`，Prisma 会直接输出查询日志，但无法实现慢查询过滤和彩色标记

#### 4. 查询事件监听器

```typescript
client.$on('query', async (e) => {
	// 事件处理逻辑
})
```

- `$on` 是 Prisma 客户端的事件订阅方法
- 监听 `'query'` 事件，每次执行数据库查询时触发
- 事件参数 `e` 包含以下属性：
  - `e.query`：执行的 SQL 语句（带参数占位符）
  - `e.params`：参数值（JSON 数组）
  - `e.duration`：执行耗时（毫秒）
  - `e.target`：目标模型（如 `User`）

#### 5. 快速返回优化

```typescript
if (e.duration < logThreshold) return
```

- **性能优化**：对于正常的快速查询，直接返回，不执行后续逻辑
- **减少开销**：避免对每个查询都进行颜色计算和字符串格式化
- **关注点分离**：只处理需要关注的慢查询

#### 6. 颜色分级系统

```typescript
const color =
	e.duration < logThreshold * 1.1
		? 'green'
		: e.duration < logThreshold * 1.2
			? 'blue'
			: e.duration < logThreshold * 1.3
				? 'yellow'
				: e.duration < logThreshold * 1.4
					? 'redBright'
					: 'red'
```

**颜色分级策略**（以阈值 20ms 为例）：

| 颜色 | 耗时范围 | 含义 | 严重程度 |
|------|----------|------|----------|
| `green`（绿色） | 20ms - 22ms | 轻微超时 | 低 |
| `blue`（蓝色） | 22ms - 24ms | 轻度超时 | 低 |
| `yellow`（黄色） | 24ms - 26ms | 中度超时 | 中 |
| `redBright`（亮红） | 26ms - 28ms | 重度超时 | 高 |
| `red`（红色） | >= 28ms | 严重超时 | 极高 |

**设计亮点**：
- 使用**相对阈值**而非绝对值，便于统一调整
- 分级细致，能够区分不同程度的性能问题
- 颜色从冷色调到暖色调渐变，视觉上一目了然

#### 7. 格式化与输出

```typescript
const dur = styleText(color, `${e.duration}ms`)
console.info(`prisma:query - ${dur} - ${e.query}`)
```

- `styleText` 是 Node.js `util` 模块的函数，用于在终端输出彩色文本
- 输出格式：`prisma:query - <彩色耗时> - <SQL语句>`

**日志输出示例**：
```
prisma:query - 25ms - SELECT "public"."User"."id", "public"."User"."name" FROM "public"."User" WHERE "public"."User"."id" = $1
```

- 其中 `25ms` 会以**黄色**显示（因为 25ms 在 24-26ms 区间）

### 与 Sentry 的集成

虽然这段代码没有直接集成 Sentry，但项目通过其他方式实现了 Prisma 查询的 Sentry 监控：

**在 `server/utils/monitoring.ts` 中**：
```typescript
import { PrismaInstrumentation } from '@prisma/instrumentation'
import { nodeProfilingIntegration } from '@sentry/profiling-node'
import * as Sentry from '@sentry/react-router'

export function init() {
	Sentry.init({
		// ...
		integrations: [
			Sentry.prismaIntegration({
				prismaInstrumentation: new PrismaInstrumentation(),
			}),
			// ...
		],
		// ...
	})
}
```

- 使用 `@prisma/instrumentation` 和 `Sentry.prismaIntegration`
- 这样 Prisma 的每个查询都会作为 span 记录在 Sentry 的性能追踪中
- 与控制台的慢查询日志形成互补：
  - **控制台日志**：开发环境实时查看，快速定位
  - **Sentry 追踪**：生产环境持久化记录，聚合分析

### 设计要点总结

1. **单例模式**：使用 `remember` 确保 Prisma 客户端全局唯一
2. **事件驱动**：通过 `emit: 'event'` 和 `$on('query')` 实现灵活的查询监控
3. **快速返回**：只处理慢查询，减少不必要的开销
4. **颜色分级**：直观展示慢查询的严重程度
5. **可配置**：阈值可调整，适应不同环境需求
6. **与 Sentry 互补**：控制台实时查看 + Sentry 持久化追踪

---

## Sentry 配置

### 架构概述

项目采用**前后端分离**的 Sentry 配置策略：

| 层级 | 配置文件 | 初始化时机 | 主要功能 |
|------|----------|------------|----------|
| 服务端 | `server/utils/monitoring.ts` | 服务启动时 | Prisma 追踪、HTTP 监控、Node.js 性能分析 |
| 客户端 | `app/utils/monitoring.client.tsx` | 客户端 hydration 时 | 会话重放、浏览器性能分析、错误过滤 |
| 环境变量 | `app/utils/env.server.ts` | 全局初始化 | SENTRY_DSN 等配置验证和暴露 |

### 服务端 Sentry 配置

#### 启动条件与初始化链路

**触发点**: `server/index.ts:12-21`

服务端 Sentry 监控的开启需要满足**两个必要条件**：

```typescript
// 第 12-16 行：环境变量和模式检查
const MODE = process.env.NODE_ENV ?? 'development'
const IS_PROD = MODE === 'production'
const IS_DEV = MODE === 'development'
// ...
const SENTRY_ENABLED = IS_PROD && process.env.SENTRY_DSN
```

| 条件 | 要求 | 说明 |
|------|------|------|
| 环境模式 | `NODE_ENV === 'production'` | 开发环境禁用，避免噪音数据 |
| DSN 配置 | `process.env.SENTRY_DSN` 存在且非空 | Sentry 项目的唯一标识符 |

**初始化触发逻辑**:

```typescript
// 第 19-21 行：条件初始化
if (SENTRY_ENABLED) {
	void import('./utils/monitoring.ts').then(({ init }) => init())
}
```

**设计要点**：
1. **动态导入**：使用 `import()` 而非静态 `import`
   - 只有在 `SENTRY_ENABLED === true` 时才加载 `monitoring.ts`
   - 减小首屏 bundle 大小（虽然服务端不涉及 bundle，但符合一致性设计）
   - 不使用 Sentry 时，相关代码完全不会执行

2. **void 关键字**：`void import(...)` 显式忽略 Promise 返回值
   - 表示"触发但不等待"异步操作
   - Sentry 初始化不应该阻塞服务启动流程

3. **全局导入**：`server/index.ts:3` 静态导入了 Sentry
   ```typescript
   import * as Sentry from '@sentry/react-router'
   ```
   - 这是为了在其他地方（如 `closeWithGrace`）直接使用 `Sentry.captureException`
   - 但实际的 `Sentry.init()` 只有在 `SENTRY_ENABLED` 时才会调用

---

#### 完整初始化链路

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        服务端 Sentry 初始化链路                            │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  服务启动 (node server/index.ts)                                          │
│       │                                                                   │
│       ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  server/index.ts:12-16                                          │    │
│  │  ├── const MODE = process.env.NODE_ENV ?? 'development'        │    │
│  │  ├── const IS_PROD = MODE === 'production'                      │    │
│  │  └── const SENTRY_ENABLED = IS_PROD && process.env.SENTRY_DSN  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│       │                                                                   │
│       ├── SENTRY_ENABLED === false ─────────────────► 跳过初始化        │
│       │                                              (开发环境或无DSN)   │
│       │                                                                   │
│       ▼ (SENTRY_ENABLED === true)                                        │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  server/index.ts:19-21                                          │    │
│  │  void import('./utils/monitoring.ts')                           │    │
│  │    .then(({ init }) => init())                                   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│       │                                                                   │
│       ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  server/utils/monitoring.ts:5-42                                │    │
│  │  export function init() {                                        │    │
│  │    Sentry.init({                                                 │    │
│  │      dsn: process.env.SENTRY_DSN,                               │    │
│  │      environment: process.env.NODE_ENV,                         │    │
│  │      denyUrls: [...],                                            │    │
│  │      integrations: [                                             │    │
│  │        Sentry.prismaIntegration(...),                           │    │
│  │        Sentry.httpIntegration(),                                 │    │
│  │        nodeProfilingIntegration(),                               │    │
│  │      ],                                                          │    │
│  │      tracesSampler() { ... },                                    │    │
│  │      beforeSendTransaction() { ... },                            │    │
│  │    })                                                             │    │
│  │  }                                                                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│       │                                                                   │
│       ▼                                                                   │
│  ✅ 服务端 Sentry 初始化完成                                              │
│     ├── 自动捕获 HTTP 请求作为 Transaction                                │
│     ├── 自动追踪 Prisma 查询作为 Span                                    │
│     ├── 自动收集 Node.js 性能分析数据                                    │
│     └── 准备好接收错误上报和性能数据                                      │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

#### 错误上报链路

服务端错误上报有**两个主要入口**（注意：以下都是**纯服务端**的错误捕获，不涉及客户端）：

##### 入口 1：React Router 全局错误处理

**文件位置**: `app/entry.server.tsx:125-142`

这是**最主要的错误捕获点**，捕获所有在服务端 loader、action 或 SSR 渲染过程中发生的错误：

```typescript
export function handleError(
	error: unknown,
	{ request }: LoaderFunctionArgs | ActionFunctionArgs,
): void {
	// 第 131-133 行：忽略请求中止的错误
	// 用户关闭页面或取消请求时产生的错误，不需要追踪
	if (request.signal.aborted) {
		return
	}

	// 第 135-139 行：控制台错误输出（开发环境调试用）
	if (error instanceof Error) {
		console.error(styleText('red', String(error.stack)))
	} else {
		console.error(error)
	}

	// 第 141 行：关键！将错误发送到 Sentry
	Sentry.captureException(error)
}
```

**错误来源（纯服务端）**：
- `LoaderFunctionArgs`：服务端 loader 中抛出的错误
- `ActionFunctionArgs`：服务端 action 中抛出的错误
- React 组件 SSR 渲染过程中抛出的错误

**过滤逻辑**：
- `request.signal.aborted`：请求被中止时不追踪
- 这是因为用户主动取消操作（如关闭标签页）产生的错误是预期行为

##### 入口 2：服务优雅关闭时的错误处理

**文件位置**: `server/index.ts:236-248`

使用 `close-with-grace` 库处理服务优雅关闭时的错误：

```typescript
closeWithGrace(async ({ err }) => {
	// 第 237-239 行：关闭 HTTP 服务器
	await new Promise((resolve, reject) => {
		server.close((e) => (e ? reject(e) : resolve('ok')))
	})

	// 第 240-247 行：处理关闭过程中的错误
	if (err) {
		// 控制台输出
		console.error(styleText('red', String(err)))
		console.error(styleText('red', String(err.stack)))

		// 第 243-246 行：发送到 Sentry
		if (SENTRY_ENABLED) {
			Sentry.captureException(err)
			await Sentry.flush(500)  // 等待最多 500ms 确保事件发送
		}
	}
})
```

**特殊处理**：
- `SENTRY_ENABLED` 检查：只有在 Sentry 启用时才调用 `captureException`
- `Sentry.flush(500)`：确保在进程退出前将事件发送到 Sentry
- 正常情况下，Sentry 会异步发送事件，但进程即将退出时需要显式等待
- `500ms` 是超时时间，超过则放弃

---

#### 监控未开启时错误上报的走向

**重要问题**：当 `SENTRY_ENABLED === false`（开发环境或未配置 DSN）时，错误上报会发生什么？

让我们分析两种情况：

##### 情况 1：`handleError` 中的 `Sentry.captureException`

**文件位置**: `app/entry.server.tsx:141`

```typescript
// 第 141 行：无条件调用
Sentry.captureException(error)
```

**行为分析**：
- 这里**没有**检查 `SENTRY_ENABLED`
- 即使 `Sentry.init()` 没有被调用，`Sentry.captureException(error)` 仍然可以执行
- **Sentry SDK 的设计**：
  - 如果 `Sentry.init()` 没有被调用，SDK 处于"未初始化"状态
  - 此时 `captureException` 会：
    1. 检查是否已初始化
    2. 如果未初始化，**静默丢弃事件**，不会抛出异常
    3. 也不会发送任何网络请求

**结论**：
- 在未启用 Sentry 的情况下，`Sentry.captureException(error)` 是一个**安全的空操作**
- 错误不会发送到 Sentry，但也不会影响应用运行
- 但错误**会**输出到控制台（第 135-139 行的 `console.error`）

##### 情况 2：`closeWithGrace` 中的 `Sentry.captureException`

**文件位置**: `server/index.ts:243-246`

```typescript
// 第 243-246 行：有条件检查
if (SENTRY_ENABLED) {
	Sentry.captureException(err)
	await Sentry.flush(500)
}
```

**行为分析**：
- 这里**有**显式检查 `SENTRY_ENABLED`
- 只有在 `SENTRY_ENABLED === true` 时才会执行
- 未启用时，这段代码完全不会执行

##### 两种方式的对比

| 位置 | 是否检查 SENTRY_ENABLED | 未启用时行为 |
|------|-------------------------|--------------|
| `entry.server.tsx:handleError` | ❌ 否 | SDK 静默丢弃 |
| `server/index.ts:closeWithGrace` | ✅ 是 | 代码不执行 |

**为什么设计差异**：
- `handleError` 是 React Router 的全局钩子，可能在 Sentry 初始化前就被调用
- 使用 SDK 内置的静默丢弃机制更安全，不会因为初始化顺序问题导致错误
- `closeWithGrace` 是服务启动后才会执行的逻辑，此时 `SENTRY_ENABLED` 已经确定

##### 监控未开启时的完整错误处理流程

```
┌──────────────────────────────────────────────────────────────────────────┐
│               监控未开启时的错误处理流程（SENTRY_ENABLED = false）        │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  服务端发生错误                                                            │
│       │                                                                   │
│       ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  入口 1: entry.server.tsx:handleError()                          │   │
│  │  ├── 检查 request.signal.aborted                                  │   │
│  │  │   └── true  ──► return (忽略请求中止错误)                      │   │
│  │  │                                                                 │   │
│  │  ├── console.error(错误栈)  ◄── 错误输出到控制台                  │   │
│  │  │                                                                 │   │
│  │  └── Sentry.captureException(error)                               │   │
│  │      └── SDK 检查是否已初始化                                      │   │
│  │          └── 未初始化 ──► 静默丢弃，不发送任何请求                │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│       │                                                                   │
│       └── 如果是服务关闭错误：                                            │
│           │                                                               │
│           ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  入口 2: server/index.ts:closeWithGrace()                        │   │
│  │  ├── 检查 SENTRY_ENABLED                                          │   │
│  │  │   └── false ──► 整个 if 块不执行                              │   │
│  │  │                                                                 │   │
│  │  └── console.error(错误栈)  ◄── 错误仍然输出到控制台              │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  【最终结果】                                                             │
│  ├── ✅ 错误会输出到控制台（可用于开发调试）                              │
│  ├── ❌ 错误不会发送到 Sentry                                            │
│  └── ✅ 应用继续运行，不会因为 Sentry 未配置而崩溃                        │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

#### 服务端错误上报完整流程图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     服务端错误上报完整链路（纯服务端）                      │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  【错误发生场景（纯服务端）】                                              │
│  ├── 场景 A: Loader 抛出错误                                              │
│  ├── 场景 B: Action 抛出错误                                              │
│  └── 场景 C: SSR 组件渲染抛出错误                                          │
│       │                                                                   │
│       ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  entry.server.tsx: handleError()  ◄── React Router 全局钩子      │   │
│  │  ├── 1. 检查 request.signal.aborted                                │   │
│  │  │   └── true  ──► return（用户取消请求，忽略）                    │   │
│  │  │                                                                 │   │
│  │  ├── 2. console.error(错误栈)  ◄── 总是输出到控制台               │   │
│  │  │                                                                 │   │
│  │  └── 3. Sentry.captureException(error)                            │   │
│  │      ├── SENTRY_ENABLED = true                                    │   │
│  │      │   └── SDK 已初始化 ──► 发送到 Sentry 服务器               │   │
│  │      │                                                             │   │
│  │      └── SENTRY_ENABLED = false                                   │   │
│  │          └── SDK 未初始化 ──► 静默丢弃，不发送                    │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  【另一个入口】服务关闭错误                                        │   │
│  ├──────────────────────────────────────────────────────────────────┤ │
│  │                                                                    │ │
│  │  服务收到 SIGTERM/SIGINT 信号                                      │ │
│  │       │                                                            │ │
│  │       ▼                                                            │ │
│  │  server/index.ts: closeWithGrace()                                │ │
│  │  ├── 关闭 HTTP 服务器                                              │ │
│  │  ├── console.error(错误栈)  ◄── 总是输出到控制台                 │ │
│  │  │                                                                 │ │
│  │  └── 检查 SENTRY_ENABLED                                          │ │
│  │      ├── true  ──► Sentry.captureException() + flush()           │ │
│  │      │               └── 发送到 Sentry 服务器                     │ │
│  │      │                                                             │ │
│  │      └── false ──► 不调用 Sentry API（代码不执行）                │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

#### 性能采样与错误上报的并行关系

**重要概念**：性能采样和错误上报是 Sentry 的**两个独立功能**，它们是**并行关系**，不是同一条链路。

##### 功能对比

| 维度 | 性能采样（Performance Sampling） | 错误上报（Error Reporting） |
|------|-----------------------------------|------------------------------|
| **目的** | 追踪请求性能，识别慢操作 | 捕获异常，追踪错误 |
| **触发时机** | 每个 HTTP 请求开始时自动创建 | 错误发生时手动或自动触发 |
| **数据内容** | Transaction + Spans（耗时、操作类型） | Event + Stack Trace（错误信息、调用栈） |
| **采样控制** | `tracesSampler` + `tracesSampleRate` | `beforeSend` 钩子（可选） |
| **是否必须** | 可选，可配置采样率 | 总是发送（除非被过滤） |

##### 并行执行流程图

```
┌──────────────────────────────────────────────────────────────────────────┐
│               性能采样与错误上报的并行关系                                 │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  HTTP 请求到达服务端                                                       │
│       │                                                                   │
│       ├─────────────────────────────────────────────────────────────────┤
│       │                                                                 │
│       ▼                                                                 ▼
│  ┌──────────────────────┐                                    ┌──────────────────────┐
│  │   【性能采样链路】    │                                    │   【错误上报链路】    │
│  │ （独立于错误上报）    │                                    │ （独立于性能采样）    │
│  ├──────────────────────┤                                    ├──────────────────────┤
│  │                      │                                    │                      │
│  │  1. 请求开始时        │                                    │  1. 等待错误发生      │
│  │     自动创建 Transaction │                                  │     （可能发生，也    │
│  │                      │                                    │     可能不发生）      │
│  │  2. tracesSampler()  │                                    │                      │
│  │     决定是否采样      │                                    │  2. 错误发生时：      │
│  │                      │                                    │     - handleError()   │
│  │  3. 采样 = 1 时：     │                                    │     - captureException│
│  │     - 收集 Spans      │                                    │                      │
│  │       (Prisma 查询等) │                                    │  3. 发送错误事件      │
│  │                      │                                    │                      │
│  │  4. 请求完成时        │                                    │                      │
│  │     发送 Transaction  │                                    │                      │
│  │                      │                                    │                      │
│  └──────────────────────┘                                    └──────────────────────┘
│       │                                                                 │
│       │                                                                 │
│       ├─────────────────────────────────────────────────────────────────┤
│       │                                                                 │
│       ▼                                                                 ▼
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                    四种可能的组合情况                               │ │
│  ├────────────────────────────────────────────────────────────────────┤ │
│  │                                                                    │ │
│  │  情况 1: 正常请求，无错误                                          │ │
│  │  ├── 性能采样：采样 = 1 ──► 发送 Transaction                      │ │
│  │  └── 错误上报：无错误 ──► 不发送                                  │ │
│  │                                                                    │ │
│  │  情况 2: 请求有错误，采样 = 1                                      │ │
│  │  ├── 性能采样：采样 = 1 ──► 发送 Transaction                      │ │
│  │  └── 错误上报：有错误 ──► 发送错误 Event                          │ │
│  │      └── 错误 Event 会关联到对应的 Transaction                    │ │
│  │                                                                    │ │
│  │  情况 3: 请求有错误，采样 = 0                                      │ │
│  │  ├── 性能采样：采样 = 0 ──► 不发送 Transaction                    │ │
│  │  └── 错误上报：有错误 ──► 仍然发送错误 Event                      │ │
│  │      └── 错误 Event 没有关联的 Transaction                         │ │
│  │                                                                    │ │
│  │  情况 4: 请求无错误，采样 = 0                                      │ │
│  │  ├── 性能采样：采样 = 0 ──► 不发送 Transaction                    │ │
│  │  └── 错误上报：无错误 ──► 不发送                                  │ │
│  │                                                                    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  【关键结论】                                                             │
│  ├── ✅ 错误上报不受采样率影响，错误总是会被发送（除非被过滤）           │
│  ├── ✅ 性能采样只控制是否收集性能数据，不影响错误捕获                   │
│  ├── ✅ 当采样 = 1 且有错误时，错误会关联到对应的 Transaction           │
│  └── ✅ 当采样 = 0 且有错误时，错误仍然会发送，但没有 Transaction 关联   │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

##### 代码层面的独立性证明

让我们从代码层面证明它们是独立的：

**证据 1：性能采样的配置**

```typescript
// server/utils/monitoring.ts
Sentry.init({
	// ...
	tracesSampler(samplingContext) {
		// 只影响性能采样
		if (samplingContext.request?.url?.includes('/resources/healthcheck')) {
			return 0  // 不采样性能数据
		}
		return process.env.NODE_ENV === 'production' ? 1 : 0
	},
	
	beforeSendTransaction(event) {
		// 只影响 Transaction（性能数据）
		if (event.request?.headers?.['x-healthcheck'] === 'true') {
			return null  // 丢弃性能数据
		}
		return event
	},
	
	// 注意：没有配置 beforeSend 钩子（错误事件的钩子）
})
```

**证据 2：错误上报的代码**

```typescript
// entry.server.tsx
export function handleError(...) {
	// ...
	
	// 直接调用，不涉及任何采样逻辑
	Sentry.captureException(error)
}
```

**证据 3：Sentry SDK 的内部架构**

```
Sentry SDK 内部架构
├── 性能模块（Performance）
│   ├── Transaction 管理
│   ├── Span 收集
│   └── tracesSampler 决策
│
├── 错误模块（Error）
│   ├── captureException 处理
│   ├── Stack Trace 解析
│   └── beforeSend 钩子
│
└── 共享模块
    ├── DSN 配置
    ├── 环境信息
    └── 网络传输层
```

**关键设计**：
- 性能模块和错误模块是**独立的子系统**
- 它们共享同一个 DSN 和网络传输层
- 但**采样决策**和**错误捕获**是完全独立的逻辑

##### 实际场景说明

**场景 A：健康检查请求被过滤**

```typescript
// tracesSampler 返回 0（不采样性能）
if (samplingContext.request?.url?.includes('/resources/healthcheck')) {
	return 0
}

// 但如果健康检查接口抛出错误：
// Sentry.captureException(error) 仍然会发送错误！
```

**这意味着**：
- 健康检查的**性能数据**被过滤（不采样）
- 但健康检查的**错误**仍然会被上报
- 这是正确的行为：健康检查出错是严重问题，需要告警

**场景 B：开发环境禁用采样**

```typescript
// 开发环境 tracesSampler 返回 0
return process.env.NODE_ENV === 'production' ? 1 : 0
```

**这意味着**：
- 开发环境**不收集性能数据**（避免噪音）
- 但开发环境的**错误仍然会被上报**（如果配置了 Sentry）
- 或者在未配置 Sentry 时，错误输出到控制台

---

#### 性能采样链路

性能采样控制哪些请求的性能数据会被发送到 Sentry。

##### 采样策略配置

**文件位置**: `server/utils/monitoring.ts:26-41`

```typescript
// 第 26-31 行：事务采样器 - 决定是否采样
tracesSampler(samplingContext) {
	// 忽略健康检查请求
	if (samplingContext.request?.url?.includes('/resources/healthcheck')) {
		return 0
	}
	// 生产环境 100% 采样，开发环境 0%
	return process.env.NODE_ENV === 'production' ? 1 : 0
},

// 第 33-41 行：事务发送前钩子 - 二次过滤
beforeSendTransaction(event) {
	// 忽略所有健康检查相关的事务
	// 注意：header 名称是区分大小写的
	if (event.request?.headers?.['x-healthcheck'] === 'true') {
		return null
	}

	return event
}
```

##### 采样决策流程

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        性能采样决策流程                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  HTTP 请求到达服务端                                                       │
│           │                                                                │
│           ▼                                                                │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Sentry React Router 集成自动创建 Transaction                     │   │
│  │  ├── 名称：HTTP 方法 + 路由模式（如 GET /users/:id）              │   │
│  │  ├── 开始时间：请求到达时                                          │   │
│  │  └── 上下文：请求信息（URL、headers、method 等）                   │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│           │                                                                │
│           ▼                                                                │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  第 1 层过滤：tracesSampler()                                      │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │  samplingContext.request?.url                                │  │   │
│  │  │  .includes('/resources/healthcheck')                         │  │   │
│  │  │         │                                                     │  │   │
│  │  │         ├── true  ──► return 0  ──► 不采样，丢弃 Transaction │  │   │
│  │  │         │                                                     │  │   │
│  │  │         └── false ──► 继续检查环境                           │  │   │
│  │  │                   │                                           │  │   │
│  │  │                   ├── NODE_ENV === 'production'              │  │   │
│  │  │                   │       ├── true  ──► return 1  ◄── 100%  │  │   │
│  │  │                   │       │                     采样          │  │   │
│  │  │                   │       │                                   │  │   │
│  │  │                   │       └── false ──► return 0  ◄── 开发  │  │   │
│  │  │                   │                           环境不采样      │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│           │                                                                │
│           ├── tracesSampler 返回 0 ──────────► 流程结束，不采样         │
│           │                                                                │
│           ▼ (tracesSampler 返回 1)                                        │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Transaction 被收集，等待发送                                      │   │
│  │  ├── 自动添加的 Span：                                             │   │
│  │  │   ├── Prisma 查询（通过 prismaIntegration）                     │   │
│  │  │   ├── HTTP 出站请求（通过 httpIntegration）                     │   │
│  │  │   └── 其他集成的追踪数据                                        │   │
│  │  └── 等待请求完成（response 发送）                                  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│           │                                                                │
│           ▼                                                                │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  第 2 层过滤：beforeSendTransaction()                              │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │  event.request?.headers?.['x-healthcheck'] === 'true'      │  │   │
│  │  │         │                                                     │  │   │
│  │  │         ├── true  ──► return null  ◄── 丢弃，不发送         │  │   │
│  │  │         │                                                     │  │   │
│  │  │         └── false ──► return event  ◄── 继续，发送到 Sentry │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│           │                                                                │
│           ├── return null ───────────────────► 丢弃，不发送              │
│           │                                                                │
│           ▼ (return event)                                                 │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  ✅ 发送到 Sentry 服务器                                           │   │
│  │  ├── Transaction 数据包含：                                        │   │
│  │  │   ├── 总耗时                                                    │   │
│  │  │   ├── 各阶段 Span（Prisma 查询等）                              │   │
│  │  │   ├── 请求信息（URL、method、status code）                      │   │
│  │  │   └── 环境信息                                                  │   │
│  │  └── Sentry 服务端：                                               │   │
│  │      ├── 性能分析 Dashboard                                         │   │
│  │      ├── 慢请求识别                                                │   │
│  │      └── 趋势分析                                                  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

##### 双层过滤设计说明

| 过滤层 | 函数 | 过滤依据 | 目的 |
|--------|------|----------|------|
| 第 1 层 | `tracesSampler` | URL 路径 | 基于 URL 模式快速过滤 |
| 第 2 层 | `beforeSendTransaction` | HTTP Header | 基于请求头二次确认 |

**为什么需要两层过滤**：
1. **URL 过滤**（`tracesSampler`）：
   - 快速过滤 `/resources/healthcheck` 等已知路径
   - 在 Transaction 创建初期决定，减少不必要的开销

2. **Header 过滤**（`beforeSendTransaction`）：
   - 检查 `x-healthcheck: true` 头
   - 有些健康检查可能通过不同的路径发送，但带有特殊 header
   - 在发送前做最后的确认

**健康检查过滤的重要性**：
- 健康检查通常由负载均衡器（如 Consul、Nginx）频繁调用（可能每秒多次）
- 这些请求没有业务价值，但会产生大量性能数据
- 不过滤会：
  - 增加 Sentry 事件数量（成本增加）
  - 稀释真正的业务请求数据
  - 影响性能趋势分析的准确性

---

#### 自动性能追踪集成

服务端通过三个集成实现自动性能追踪：

```typescript
integrations: [
	// 1. Prisma 查询追踪
	Sentry.prismaIntegration({
		prismaInstrumentation: new PrismaInstrumentation(),
	}),
	// 2. HTTP 请求追踪
	Sentry.httpIntegration(),
	// 3. Node.js 性能分析
	nodeProfilingIntegration(),
]
```

| 集成 | 包来源 | 功能说明 | 数据内容 |
|------|--------|----------|----------|
| `prismaIntegration` | `@sentry/react-router` | 自动追踪 Prisma 查询 | 查询类型、模型、耗时、参数 |
| `PrismaInstrumentation` | `@prisma/instrumentation` | Prisma 官方 OpenTelemetry 集成 | 底层查询拦截和数据收集 |
| `httpIntegration` | `@sentry/react-router` | 自动追踪 HTTP 请求 | 入站/出站请求、URL、状态码、耗时 |
| `nodeProfilingIntegration` | `@sentry/profiling-node` | Node.js 性能分析 | CPU 使用情况、函数调用栈、内存分配 |

**Prisma 集成工作原理**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Prisma 查询自动追踪流程                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  业务代码调用                                                    │
│  prisma.user.findUnique({ where: { id: '123' } })              │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  PrismaInstrumentation (OpenTelemetry)                  │   │
│  │  ├── 自动拦截 Prisma 查询                                │   │
│  │  ├── 收集查询元数据：                                     │   │
│  │  │   ├── 操作类型：findUnique                            │   │
│  │  │   ├── 模型：User                                     │   │
│  │  │   ├── 开始时间                                        │   │
│  │  │   └── 查询参数（可选）                                │   │
│  │  └── 查询完成时收集耗时                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Sentry.prismaIntegration                                │   │
│  │  ├── 接收 OpenTelemetry 数据                             │   │
│  │  ├── 转换为 Sentry Span 格式                            │   │
│  │  └── 添加到当前 Transaction                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Transaction 结构示例                                    │   │
│  │  ├── 名称：GET /users/:id                                │   │
│  │  ├── 总耗时：150ms                                       │   │
│  │  └── Spans：                                              │   │
│  │      ├── Span 1: db.sql.prisma                          │   │
│  │      │   ├── 描述：SELECT FROM "User"                   │   │
│  │      │   ├── 耗时：45ms                                 │   │
│  │      │   └── 数据：{ model: 'User', operation: 'findUnique' } │
│  │      └── Span 2: db.sql.prisma (如果有多个查询)         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Document-Policy 头配合**：

在 `entry.server.tsx:38-40` 中，生产环境会添加特殊的响应头：

```typescript
if (process.env.NODE_ENV === 'production' && process.env.SENTRY_DSN) {
	responseHeaders.append('Document-Policy', 'js-profiling')
}
```

- `Document-Policy: js-profiling` 启用浏览器的 JavaScript Profiling API
- 配合 `browserProfilingIntegration` 可以获取更详细的前端性能数据
- 这是服务端和客户端性能监控的配合点

---

**文件位置**: `server/utils/monitoring.ts`

#### 完整实现代码

```typescript
// 第 1-3 行：导入依赖
import { PrismaInstrumentation } from '@prisma/instrumentation'
import { nodeProfilingIntegration } from '@sentry/profiling-node'
import * as Sentry from '@sentry/react-router'

// 第 5-42 行：Sentry 初始化函数
export function init() {
	Sentry.init({
		// 第 7 行：DSN 配置
		dsn: process.env.SENTRY_DSN,

		// 第 8 行：环境标识
		environment: process.env.NODE_ENV,

		// 第 9-18 行：忽略的 URL 路径
		denyUrls: [
			/\/resources\/healthcheck/,
			// TODO: be smarter about the public assets...
			/\/build\//,
			/\/favicons\//,
			/\/img\//,
			/\/fonts\//,
			/\/favicon.ico/,
			/\/site\.webmanifest/,
		],

		// 第 19-25 行：集成配置
		integrations: [
			// Prisma 查询追踪集成
			Sentry.prismaIntegration({
				prismaInstrumentation: new PrismaInstrumentation(),
			}),
			// HTTP 请求追踪集成
			Sentry.httpIntegration(),
			// Node.js 性能分析集成
			nodeProfilingIntegration(),
		],

		// 第 26-31 行：事务采样器
		tracesSampler(samplingContext) {
			// 忽略健康检查请求
			if (samplingContext.request?.url?.includes('/resources/healthcheck')) {
				return 0
			}
			// 生产环境 100% 采样，开发环境 0%
			return process.env.NODE_ENV === 'production' ? 1 : 0
		},

		// 第 33-41 行：事务发送前钩子
		beforeSendTransaction(event) {
			// 忽略所有健康检查相关的事务
			// 注意：header 名称是区分大小写的
			if (event.request?.headers?.['x-healthcheck'] === 'true') {
				return null
			}

			return event
		},
	})
}
```

#### 核心配置详解

##### 1. DSN 和环境配置

```typescript
dsn: process.env.SENTRY_DSN,
environment: process.env.NODE_ENV,
```

- **DSN（Data Source Name）**：Sentry 项目的唯一标识符
- **environment**：环境标识（`production`、`development`、`test`）
- 这些值从环境变量读取，确保不同环境的数据隔离

##### 2. denyUrls - URL 过滤

```typescript
denyUrls: [
	/\/resources\/healthcheck/,
	/\/build\//,
	/\/favicons\//,
	/\/img\//,
	/\/fonts\//,
	/\/favicon.ico/,
	/\/site\.webmanifest/,
]
```

**过滤规则说明**：

| 正则表达式 | 匹配内容 | 过滤原因 |
|------------|----------|----------|
| `/\/resources\/healthcheck/` | 健康检查接口 | 频繁调用，无业务价值 |
| `/\/build\//` | 构建产物 | 静态资源请求 |
| `/\/favicons\//` | 图标文件 | 静态资源请求 |
| `/\/img\//` | 图片文件 | 静态资源请求 |
| `/\/fonts\//` | 字体文件 | 静态资源请求 |
| `/\/favicon.ico/` | 网站图标 | 静态资源请求 |
| `/\/site\.webmanifest/` | PWA 清单 | 静态资源请求 |

**设计目的**：
- 减少 Sentry 事件数量，降低成本
- 避免无意义的噪音，专注于业务错误和性能问题
- TODO 注释说明"be smarter about the public assets"，表示未来可能优化

##### 3. integrations - 集成配置

```typescript
integrations: [
	Sentry.prismaIntegration({
		prismaInstrumentation: new PrismaInstrumentation(),
	}),
	Sentry.httpIntegration(),
	nodeProfilingIntegration(),
]
```

**集成功能详解**：

| 集成 | 包来源 | 功能说明 |
|------|--------|----------|
| `prismaIntegration` | `@sentry/react-router` | Prisma 查询追踪，自动记录每个数据库查询作为 span |
| `PrismaInstrumentation` | `@prisma/instrumentation` | Prisma 官方的 OpenTelemetry 集成，提供查询详细数据 |
| `httpIntegration` | `@sentry/react-router` | HTTP 请求追踪，记录出站和入站 HTTP 调用 |
| `nodeProfilingIntegration` | `@sentry/profiling-node` | Node.js 性能分析，收集 CPU 和内存使用情况 |

**Prisma 集成工作原理**：
1. `PrismaInstrumentation` 使用 OpenTelemetry 自动拦截 Prisma 查询
2. `Sentry.prismaIntegration` 将这些拦截数据转换为 Sentry 的 span 格式
3. 每个 Prisma 查询会记录：
   - 查询类型（`findUnique`、`findMany`、`update` 等）
   - 目标模型（`User`、`Note` 等）
   - 执行耗时
   - 查询参数（可选）

##### 4. tracesSampler - 事务采样器

```typescript
tracesSampler(samplingContext) {
	// 忽略健康检查事务
	if (samplingContext.request?.url?.includes('/resources/healthcheck')) {
		return 0
	}
	// 生产环境 100% 采样，开发环境 0%
	return process.env.NODE_ENV === 'production' ? 1 : 0
}
```

**采样策略**：

| 场景 | 采样率 | 说明 |
|------|--------|------|
| 健康检查请求 | 0% | 完全忽略，不追踪 |
| 生产环境 | 100% | 完整追踪所有请求 |
| 开发环境 | 0% | 不追踪，减少本地开销 |

**设计考虑**：
- 健康检查请求通常由负载均衡器（如 Consul、Nginx）频繁调用
- 开发环境禁用追踪可以：
  - 减少 Sentry 事件数量（避免噪音）
  - 提高本地开发速度
  - 避免污染生产数据

##### 5. beforeSendTransaction - 事务发送钩子

```typescript
beforeSendTransaction(event) {
	// 忽略所有健康检查相关的事务
	// 注意：header 名称是区分大小写的
	if (event.request?.headers?.['x-healthcheck'] === 'true') {
		return null
	}

	return event
}
```

**第二层过滤**：
- `tracesSampler` 是基于 URL 的过滤
- `beforeSendTransaction` 是基于 HTTP header 的过滤
- 双重过滤确保健康检查请求不会被追踪

**header 名称注意**：
- 代码注释明确说明"header 名称是区分大小写的"
- 使用 `'x-healthcheck'`（全小写）进行匹配
- 这是因为 HTTP/2 规范要求 header 名称全小写

### 客户端 Sentry 配置

**文件位置**: `app/utils/monitoring.client.tsx`

#### 完整实现代码

```typescript
// 第 1 行：导入 Sentry
import * as Sentry from '@sentry/react-router'

// 第 3-34 行：客户端 Sentry 初始化函数
export function init() {
	Sentry.init({
		// 第 5 行：DSN 配置（使用客户端环境变量）
		dsn: ENV.SENTRY_DSN,

		// 第 6 行：环境标识
		environment: ENV.MODE,

		// 第 7-19 行：错误发送前钩子
		beforeSend(event) {
			// 过滤浏览器扩展产生的错误
			if (event.request?.url) {
				const url = new URL(event.request.url)
				if (
					url.protocol === 'chrome-extension:' ||
					url.protocol === 'moz-extension:'
				) {
					// This error is from a browser extension, ignore it
					return null
				}
			}
			return event
		},

		// 第 20-23 行：集成配置
		integrations: [
			Sentry.replayIntegration(),          // 会话重放
			Sentry.browserProfilingIntegration(), // 浏览器性能分析
		],

		// 第 25-28 行：性能采样率
		// Set tracesSampleRate to 1.0 to capture 100%
		// of transactions for performance monitoring.
		// We recommend adjusting this value in production
		tracesSampleRate: 1.0,

		// 第 30-33 行：会话重放采样率
		// Capture Replay for 10% of all sessions,
		// plus for 100% of sessions with an error
		replaysSessionSampleRate: 0.1,
		replaysOnErrorSampleRate: 1.0,
	})
}
```

#### 核心配置详解

##### 1. 环境变量使用

```typescript
dsn: ENV.SENTRY_DSN,
environment: ENV.MODE,
```

- **客户端环境变量**：使用全局 `ENV` 对象而非 `process.env`
- `ENV` 对象由服务端注入（见 `env.server.ts` 的 `getEnv()` 函数）
- 这样确保：
  - 敏感的服务端环境变量不会泄露到客户端
  - 只有明确暴露的变量（如 `SENTRY_DSN`、`MODE`）才能在客户端访问

##### 2. beforeSend - 错误过滤

```typescript
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
}
```

**过滤浏览器扩展错误**：

| 协议 | 浏览器 | 说明 |
|------|--------|------|
| `chrome-extension:` | Chrome/Edge | Chrome 扩展程序 |
| `moz-extension:` | Firefox | Firefox 附加组件 |

**为什么要过滤**：
- 浏览器扩展经常会向页面注入脚本
- 这些脚本可能产生错误，但与应用代码无关
- 过滤后可以：
  - 减少 Sentry 中的噪音
  - 专注于真正的应用错误
  - 避免误报

##### 3. 客户端集成

```typescript
integrations: [
	Sentry.replayIntegration(),
	Sentry.browserProfilingIntegration(),
]
```

**集成功能说明**：

| 集成 | 功能说明 |
|------|----------|
| `replayIntegration` | **会话重放** - 记录用户的屏幕操作、DOM 变化、网络请求等，用于复现错误 |
| `browserProfilingIntegration` | **浏览器性能分析** - 收集 JavaScript 执行、渲染、网络等性能数据 |

**会话重放的价值**：
- 当用户报告错误时，可以"重放"他们的操作过程
- 看到他们看到的内容，理解错误发生的上下文
- 特别适合复现难以重现的前端 bug

##### 4. 采样率配置

```typescript
tracesSampleRate: 1.0,           // 100% 性能采样
replaysSessionSampleRate: 0.1,    // 10% 会话重放采样
replaysOnErrorSampleRate: 1.0,    // 错误会话 100% 重放
```

**采样策略设计**：

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `tracesSampleRate` | 1.0 (100%) | 所有请求都进行性能追踪 |
| `replaysSessionSampleRate` | 0.1 (10%) | 只有 10% 的正常会话会被重放记录 |
| `replaysOnErrorSampleRate` | 1.0 (100%) | **所有**发生错误的会话都会被重放记录 |

**设计考量**：

1. **性能追踪（100%）**：
   - 性能数据相对轻量
   - 需要完整的数据进行性能分析和趋势观察
   - 100% 采样确保不会错过任何性能问题

2. **会话重放（分层采样）**：
   - **成本考虑**：会话重放会记录视频、DOM 快照等，数据量较大
   - **正常会话（10%）**：用于了解整体用户行为，抽样即可
   - **错误会话（100%）**：错误是稀有事件，但价值极高，必须完整记录
   - 这种策略平衡了**成本**和**价值**

### 入口文件初始化

#### 服务端入口 - entry.server.tsx

```typescript
// 第 6 行：导入 Sentry
import * as Sentry from '@sentry/react-router'

// 第 38-40 行：性能分析头（生产环境）
if (process.env.NODE_ENV === 'production' && process.env.SENTRY_DSN) {
	responseHeaders.append('Document-Policy', 'js-profiling')
}

// 第 125-142 行：错误处理函数
export function handleError(
	error: unknown,
	{ request }: LoaderFunctionArgs | ActionFunctionArgs,
): void {
	// 第 131-133 行：忽略请求中止的错误
	if (request.signal.aborted) {
		return
	}

	// 第 135-139 行：控制台错误输出
	if (error instanceof Error) {
		console.error(styleText('red', String(error.stack)))
	} else {
		console.error(error)
	}

	// 第 141 行：关键！将错误发送到 Sentry
	Sentry.captureException(error)
}
```

**服务端初始化要点**：

1. **Document-Policy 头**：
   - 生产环境且配置了 SENTRY_DSN 时添加
   - `Document-Policy: js-profiling` 启用浏览器的 JavaScript Profiling API
   - 配合 `browserProfilingIntegration` 提供更详细的性能数据

2. **handleError 函数**：
   - React Router 的全局错误处理钩子
   - **忽略请求中止**：用户取消请求（如关闭页面）产生的错误不需要追踪
   - **双重记录**：
     - 控制台输出（带颜色的错误栈）
     - 发送到 Sentry（`Sentry.captureException(error)`）

#### 客户端入口 - entry.client.tsx

```typescript
// 第 5-7 行：条件初始化
if (ENV.MODE === 'production' && ENV.SENTRY_DSN) {
	void import('./utils/monitoring.client.tsx').then(({ init }) => init())
}
```

**客户端初始化设计**：

1. **条件初始化**：
   - 只在**生产环境**且**配置了 SENTRY_DSN** 时才初始化
   - 开发环境不初始化 Sentry，避免：
     - 本地开发产生噪音数据
     - 影响开发时的热重载速度

2. **动态导入**：
   ```typescript
   void import('./utils/monitoring.client.tsx').then(({ init }) => init())
   ```
   - 使用 `import()` 动态导入，而非静态 `import`
   - **优点**：
     - 减小首屏 bundle 大小（Sentry 相关代码按需加载）
     - 不影响不使用 Sentry 的场景
     - 初始化失败不会影响应用主流程

3. **void 关键字**：
   - `void import(...)` 显式忽略 Promise 的返回值
   - 表示"触发但不等待"这个异步操作
   - 符合语义：Sentry 初始化不应该阻塞应用启动

### 环境变量配置

**文件位置**: `app/utils/env.server.ts`

#### 完整实现代码

```typescript
import { z } from 'zod'

// 第 3-29 行：环境变量 Schema
const schema = z.object({
	NODE_ENV: z.enum(['production', 'development', 'test'] as const),
	DATABASE_PATH: z.string(),
	DATABASE_URL: z.string(),
	SESSION_SECRET: z.string(),
	INTERNAL_COMMAND_TOKEN: z.string(),
	HONEYPOT_SECRET: z.string(),
	CACHE_DATABASE_PATH: z.string(),
	// 第 11-12 行：Sentry DSN 配置（可选）
	// If you plan on using Sentry, remove the .optional()
	SENTRY_DSN: z.string().optional(),
	// ... 其他环境变量 ...
})

// ... 全局类型声明 ...

// 第 37-48 行：初始化函数
export function init() {
	const parsed = schema.safeParse(process.env)

	if (parsed.success === false) {
		console.error(
			'❌ Invalid environment variables:',
			parsed.error.flatten().fieldErrors,
		)
		throw new Error('Invalid environment variables')
	}
}

// 第 50-65 行：获取公开环境变量
export function getEnv() {
	return {
		MODE: process.env.NODE_ENV,
		SENTRY_DSN: process.env.SENTRY_DSN,  // 暴露给客户端
		ALLOW_INDEXING: process.env.ALLOW_INDEXING,
	}
}

// ... 全局类型声明 ...
```

#### 环境变量配置要点

##### 1. Zod Schema 验证

```typescript
SENTRY_DSN: z.string().optional(),
```

- 使用 `z.string().optional()` 定义为**可选**字符串
- 注释说明："If you plan on using Sentry, remove the `.optional()`"
- 这意味着：
  - 默认情况下，Sentry 是可选功能
  - 实际使用时，应该移除 `.optional()` 使其成为必填项
  - 这样可以在启动时验证 DSN 是否配置正确

##### 2. 公开环境变量暴露

```typescript
export function getEnv() {
	return {
		MODE: process.env.NODE_ENV,
		SENTRY_DSN: process.env.SENTRY_DSN,
		ALLOW_INDEXING: process.env.ALLOW_INDEXING,
	}
}
```

**为什么要暴露 `SENTRY_DSN`**：
- 客户端 Sentry 初始化需要 DSN
- `getEnv()` 返回的变量会被注入到客户端的 `window.ENV` 对象
- 这是安全的，因为：
  - Sentry DSN 本身就是设计为公开的（客户端需要它来发送事件）
  - DSN 不包含敏感的 API 密钥或 token
  - 即使被恶意获取，也只能向你的 Sentry 项目发送数据（有速率限制保护）

**安全注意事项**：
- 只暴露必要的变量
- `DATABASE_URL`、`SESSION_SECRET` 等敏感变量**不会**出现在 `getEnv()` 的返回值中

### 配置流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        服务端启动流程                              │
├─────────────────────────────────────────────────────────────────┤
│  1. env.server.ts:init()                                         │
│     └── 验证环境变量（包括 SENTRY_DSN）                           │
│                                                                   │
│  2. server/utils/monitoring.ts:init()                            │
│     ├── Sentry.init({...})                                       │
│     │   ├── dsn: process.env.SENTRY_DSN                         │
│     │   ├── integrations: [prisma, http, profiling]             │
│     │   ├── tracesSampler: 生产 100%, 开发 0%                   │
│     │   └── beforeSendTransaction: 过滤健康检查                  │
│     └── 服务端 Sentry 初始化完成                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        客户端初始化流程                            │
├─────────────────────────────────────────────────────────────────┤
│  1. entry.client.tsx 检查条件                                    │
│     └── if (ENV.MODE === 'production' && ENV.SENTRY_DSN)        │
│                                                                   │
│  2. 动态导入并初始化                                              │
│     ├── void import('./utils/monitoring.client.tsx')             │
│     │   └── .then(({ init }) => init())                          │
│     └── app/utils/monitoring.client.tsx:init()                   │
│         ├── Sentry.init({...})                                   │
│         │   ├── dsn: ENV.SENTRY_DSN                              │
│         │   ├── environment: ENV.MODE                             │
│         │   ├── beforeSend: 过滤浏览器扩展错误                    │
│         │   ├── integrations: [replay, browserProfiling]        │
│         │   ├── tracesSampleRate: 1.0 (100%)                    │
│         │   ├── replaysSessionSampleRate: 0.1 (10%)              │
│         │   └── replaysOnErrorSampleRate: 1.0 (100%)            │
│         └── 客户端 Sentry 初始化完成                               │
└─────────────────────────────────────────────────────────────────┘
```

### 设计要点总结

#### Server-Timing 系统

| 设计要点 | 实现方式 | 价值 |
|----------|----------|------|
| **结构化数据** | `Timings` 类型 + `makeTimings()` | 统一的计时数据格式 |
| **自动计时** | `time()` 函数包装异步操作 | 无需手动管理计时器 |
| **响应头转换** | `getServerTimeHeader()` + 自定义 `toString()` | 无缝集成 HTTP 响应 |
| **缓存集成** | `cachifiedTimingReporter` | 缓存操作性能可见 |
| **流式兼容** | `responseHeaders.append()` | SSR 阶段计时与 loader 计时合并 |

#### Prisma 慢查询监控

| 设计要点 | 实现方式 | 价值 |
|----------|----------|------|
| **事件驱动** | `$on('query')` + `emit: 'event'` | 灵活的查询监控 |
| **可配置阈值** | `logThreshold = 20` | 适应不同环境需求 |
| **快速返回** | `if (e.duration < logThreshold) return` | 减少不必要的开销 |
| **颜色分级** | 5 级颜色系统 | 直观展示严重程度 |
| **单例模式** | `remember('prisma', () => ...)` | 避免连接池耗尽 |

#### Sentry 监控架构

| 设计要点 | 实现方式 | 价值 |
|----------|----------|------|
| **前后端分离** | `monitoring.ts` + `monitoring.client.tsx` | 各环境独立配置 |
| **条件初始化** | 动态 `import()` + 环境检查 | 开发环境无额外开销 |
| **多层过滤** | `denyUrls` + `tracesSampler` + `beforeSend` | 减少噪音数据 |
| **Prisma 集成** | `PrismaInstrumentation` + `prismaIntegration` | 数据库查询性能追踪 |
| **会话重放** | `replayIntegration` + 分层采样 | 错误复现能力 |
| **环境变量验证** | Zod Schema + `getEnv()` 白名单 | 类型安全 + 安全暴露 |

---

## 总结

本文档详细分析了 Epic Stack 项目中的三个核心性能监控和错误追踪系统：

### 1. Server-Timing 系统

- **核心工具**：`timing.server.ts` 提供了完整的计时 API（`makeTimings`、`time`、`getServerTimeHeader`）
- **使用场景**：
  - **Root Loader**：追踪 `getUserId`、Prisma 查询等关键操作
  - **Cache Provider**：通过 `cachifiedTimingReporter` 记录缓存操作耗时
  - **SSR Render**：追踪 `renderToPipeableStream` 的 shell 渲染时间
- **输出格式**：符合 W3C Server-Timing 规范的响应头，可在浏览器 DevTools 中直接查看

### 2. Prisma 慢查询记录

- **实现方式**：通过 `$on('query')` 事件监听 + 自定义日志逻辑
- **核心配置**：
  - 阈值：默认 20ms（可调整）
  - 颜色分级：5 级颜色系统直观展示慢查询严重程度
  - 快速返回：只处理超过阈值的查询，减少性能开销
- **与 Sentry 互补**：
  - 控制台日志：开发环境实时查看，快速定位
  - Sentry 追踪：生产环境持久化记录，聚合分析

### 3. Sentry 监控架构

- **服务端配置**（`server/utils/monitoring.ts`）：
  - 集成：Prisma 追踪、HTTP 监控、Node.js 性能分析
  - 采样：生产环境 100%，开发环境 0%
  - 过滤：健康检查、静态资源请求

- **客户端配置**（`app/utils/monitoring.client.tsx`）：
  - 集成：会话重放、浏览器性能分析
  - 采样：性能追踪 100%，会话重放 10%（错误会话 100%）
  - 过滤：浏览器扩展错误

- **初始化流程**：
  - 服务端：启动时立即初始化
  - 客户端：生产环境动态导入，不阻塞首屏加载

### 关键设计理念

1. **非侵入式监控**：所有监控功能都是可选的，不影响核心业务逻辑
2. **开发/生产差异**：开发环境减少噪音，生产环境完整追踪
3. **多层过滤**：通过 URL、header、采样率等多层次减少无效数据
4. **性能与价值平衡**：会话重放等高成本功能采用分层采样策略
5. **类型安全**：使用 Zod 验证环境变量，TypeScript 确保类型一致性

这些系统共同构成了一个完整的**可观测性体系**：
- **开发阶段**：Server-Timing + Prisma 慢查询日志，实时性能反馈
- **生产阶段**：Sentry 错误追踪 + 性能监控，持续改进的依据