# SSR Nonce 与客户端 Hints 机制分析

## 1. 服务端 Nonce 生成机制

### 1.1 Nonce 生成方式

服务端在 `entry.server.tsx` 中使用 Node.js 内置的 `crypto` 模块生成随机 nonce：

```typescript
// entry.server.tsx:46
const nonce = crypto.randomBytes(16).toString('hex')
```

- 生成 16 字节（128位）的随机数据
- 转换为 32 个字符的十六进制字符串
- 每个请求生成独立的 nonce，确保安全性

### 1.2 Nonce 的传递与使用

Nonce 通过多种方式在应用中传递：

1. **React Context 传递**：
   - `NonceProvider` 组件将 nonce 注入 Context
   - `useNonce()` hook 在客户端组件中获取 nonce

2. **Server Router 配置**：
   ```typescript
   <ServerRouter
     nonce={nonce}
     context={reactRouterContext}
     url={request.url}
   />
   ```

3. **Content Security Policy (CSP)**：
   Nonce 被添加到 CSP 指令中，允许特定脚本执行：
   ```typescript
   'script-src': [
     "'strict-dynamic'",
     "'self'",
     `'nonce-${nonce}'`,
   ],
   'script-src-attr': [`'nonce-${nonce}'`],
   ```

4. **渲染选项**：
   传递给 `renderToPipeableStream` 的 `nonce` 选项

---

## 2. Bot 与 Browser 渲染时机选择

### 2.1 判断逻辑

服务端通过 `isbot` 库检测 User-Agent 来区分客户端类型：

```typescript
// entry.server.tsx:42-44
const callbackName = isbot(request.headers.get('user-agent'))
  ? 'onAllReady'
  : 'onShellReady'
```

### 2.2 两种渲染策略

| 策略 | 客户端类型 | 行为 | 目的 |
|------|-----------|------|------|
| `onAllReady` | Bot（搜索引擎爬虫等） | 等待所有组件渲染完成（包括 Suspense 边界内的内容） | 确保爬虫获取完整的 HTML 内容，利于 SEO |
| `onShellReady` | 普通浏览器 | 只等待 shell（外层框架）渲染完成，Suspense 内容后续流式传输 | 提供更快的首屏加载体验，支持流式 SSR |

### 2.3 技术实现

```typescript
const { pipe, abort } = renderToPipeableStream(
  <NonceProvider value={nonce}>
    <ServerRouter ... />
  </NonceProvider>,
  {
    [callbackName]: () => {
      // 回调触发时开始流式传输
      const body = new PassThrough()
      responseHeaders.set('Content-Type', 'text/html')
      resolve(new Response(createReadableStreamFromReadable(body), {
        headers: responseHeaders,
        status: didError ? 500 : responseStatusCode,
      }))
      pipe(body)
    },
    // ...
  },
)
```

---

## 3. Root Loader 下发主题和时区 Hints

### 3.1 整体架构

应用使用 `@epic-web/client-hints` 库处理客户端提示（client hints），解决"服务端需要浏览器才能知道的信息"这一问题。

### 3.2 Hints 定义

在 `client-hints.tsx` 中定义需要的 hints：

```typescript
const hintsUtils = getHintUtils({
  theme: colorSchemeHint,      // 主题偏好（light/dark）
  timeZone: timeZoneHint,       // 时区
})

export const { getHints } = hintsUtils
```

### 3.3 Root Loader 实现

在 `root.tsx` 的 loader 函数中收集并下发 hints：

```typescript
// root.tsx:71-133
export async function loader({ request }: Route.LoaderArgs) {
  // ... 用户信息获取等逻辑

  return data(
    {
      user,
      requestInfo: {
        hints: getHints(request),        // 从 cookie 获取客户端 hints
        origin: getDomainUrl(request),
        path: new URL(request.url).pathname,
        userPrefs: {
          theme: getTheme(request),      // 从 cookie 获取用户显式主题偏好
        },
      },
      ENV: getEnv(),
      toast,
      honeyProps,
    },
    { headers: ... },
  )
}
```

### 3.4 主题偏好获取（theme.server.ts）

```typescript
const cookieName = 'en_theme'

export function getTheme(request: Request): Theme | null {
  const cookieHeader = request.headers.get('cookie')
  const parsed = cookieHeader ? cookie.parse(cookieHeader)[cookieName] : 'light'
  if (parsed === 'light' || parsed === 'dark') return parsed
  return null
}
```

#### 3.4.1 分支逻辑的关键差异

这段代码有一个微妙但重要的分支差异，取决于请求是否包含 cookie header：

| 场景 | cookieHeader | parsed 值 | 最终返回值 |
|------|-------------|-----------|-----------|
| **无 cookie header** | `null` | `'light'`（三元运算符 else 分支） | `'light'` |
| **有 cookie header，但无 en_theme cookie** | 存在（非 null） | `undefined`（`cookie.parse()[cookieName]` 不存在） | `null` |
| **有 en_theme cookie，值为 'light' 或 'dark'** | 存在 | `'light'` 或 `'dark'` | 对应值 |
| **有 en_theme cookie，值无效（如 'auto'）** | 存在 | `'auto'` | `null` |

**代码执行流程图**：

```
getTheme(request)
       │
       ▼
cookieHeader = request.headers.get('cookie')
       │
       ├───────────────────┬─────────────────────┐
       │                   │                     │
       ▼                   ▼                     ▼
   cookieHeader      cookieHeader          cookieHeader
   === null          !== null，但          !== null，且
                     无 en_theme           有 en_theme
       │               cookie                cookie
       │                   │                     │
       ▼                   ▼                     ▼
parsed = 'light'   parsed = undefined    parsed = cookie 值
       │                   │                     │
       └───────────────────┼─────────────────────┘
                           │
                           ▼
              parsed === 'light' or 'dark' ?
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
           Yes → return parsed      No → return null
```

#### 3.4.2 与 useTheme 的组合逻辑

在客户端 `useTheme` hook 中，这个返回值会与 hints 组合：

```typescript
// theme-switch.tsx:125-133
export function useTheme() {
  const hints = useHints()
  const requestInfo = useRequestInfo()
  const optimisticMode = useOptimisticThemeMode()
  
  if (optimisticMode) {
    return optimisticMode === 'system' ? hints.theme : optimisticMode
  }
  return requestInfo.userPrefs.theme ?? hints.theme  // ← 这里使用 nullish coalescing
}
```

**优先级分析**：

| requestInfo.userPrefs.theme | hints.theme | 最终结果 | 说明 |
|------------------------------|-------------|---------|------|
| `'light'` | `'dark'` | `'light'` | 用户显式设置了 light，优先使用 |
| `'dark'` | `'light'` | `'dark'` | 用户显式设置了 dark，优先使用 |
| `null` | `'dark'` | `'dark'` | 无用户偏好，使用系统提示 |
| `undefined` | `'light'` | `'light'` | 无用户偏好，使用系统提示 |

**关键点**：`null ?? hints.theme` 会返回 `hints.theme`，因为 `??` 只有在左侧是 `null` 或 `undefined` 时才返回右侧。这意味着：
- 当 `getTheme` 返回 `null` 时，使用系统主题提示
- 当 `getTheme` 返回 `'light'` 或 `'dark'` 时，使用用户显式设置的值

### 3.5 ClientHintCheck 组件

在 `Document` 组件中渲染 `ClientHintCheck` 脚本：

```typescript
// root.tsx:152
<ClientHintCheck nonce={nonce} />
```

该组件的作用：
1. 注入一个内联脚本，检查客户端 hints 并设置对应的 cookies
2. 如果 cookie 值不准确，会触发页面刷新
3. 使用 `nonce` 确保 CSP 允许执行

---

## 4. 客户端订阅偏好变化与重新校验

### 4.0 主题与时区 Hints 处理机制对比

在深入分析之前，首先明确主题和时区两种 hints 的核心差异：

| 特性 | 主题 (theme/color-scheme) | 时区 (timeZone) |
|------|---------------------------|-----------------|
| 订阅函数 | ✅ 有 `subscribeToSchemeChange` | ❌ 无对应订阅函数 |
| 变化检测方式 | 实时监听媒体查询变化 | 仅页面加载时检查 cookie |
| 更新方式 | `revalidate()` 重新校验，无刷新 | 页面刷新/重载 |
| 用户体验 | 平滑过渡，无闪烁 | 完整页面重载 |
| 依赖的 `@epic-web/client-hints` 模块 | `color-scheme` (导出订阅函数) | `time-zone` (仅导出 hint 定义) |

**@epic-web/client-hints 支持的订阅函数**：
- `@epic-web/client-hints/color-scheme` → `subscribeToSchemeChange`
- `@epic-web/client-hints/reduced-motion` → `subscribeToMotionChange`
- `@epic-web/client-hints/time-zone` → **无订阅函数**

---

### 4.1 系统主题变化订阅

在 `ClientHintCheck` 组件中订阅系统主题变化：

```typescript
// client-hints.tsx:41-56
export function ClientHintCheck({ nonce }: { nonce: string }) {
  const { revalidate } = useRevalidator()
  
  React.useEffect(
    () => subscribeToSchemeChange(() => revalidate()),
    [revalidate],
  )

  return (
    <script
      nonce={nonce}
      dangerouslySetInnerHTML={{
        __html: hintsUtils.getClientHintCheckScript(),
      }}
    />
  )
}
```

**关键点**：
- 使用 `subscribeToSchemeChange` 监听 `prefers-color-scheme` 媒体查询变化
- 变化时调用 `revalidate()` 触发 React Router 的数据重新校验
- 这会重新运行 root loader，获取最新的 hints

### 4.2 用户手动切换主题

#### 4.2.1 资源路由 Action（theme-switch.tsx）

```typescript
export async function action({ request }: Route.ActionArgs) {
  const formData = await request.formData()
  const submission = parseWithZod(formData, {
    schema: ThemeFormSchema,  // z.object({ theme: z.enum(['system', 'light', 'dark']), ... })
  })

  const { theme, redirectTo } = submission.value

  const responseInit = {
    headers: { 'set-cookie': setTheme(theme) },
  }
  if (redirectTo) {
    return redirect(redirectTo, responseInit)
  } else {
    return data({ result: submission.reply() }, responseInit)
  }
}
```

- 接收 `theme` 参数（system/light/dark）
- 调用 `setTheme()` 设置 cookie
- `theme = 'system'` 时会删除 cookie，让系统偏好生效

#### 4.2.2 setTheme 函数

```typescript
// theme.server.ts:13-19
export function setTheme(theme: Theme | 'system') {
  if (theme === 'system') {
    return cookie.serialize(cookieName, '', { path: '/', maxAge: -1 })  // 删除 cookie
  } else {
    return cookie.serialize(cookieName, theme, { path: '/', maxAge: 31536000 })  // 保存一年
  }
}
```

### 4.3 乐观 UI 更新

#### 4.3.1 useOptimisticThemeMode

```typescript
export function useOptimisticThemeMode() {
  const fetchers = useFetchers()
  const themeFetcher = fetchers.find(
    (f) => f.formAction === '/resources/theme-switch',
  )

  if (themeFetcher && themeFetcher.formData) {
    const submission = parseWithZod(themeFetcher.formData, {
      schema: ThemeFormSchema,
    })

    if (submission.status === 'success') {
      return submission.value.theme
    }
  }
}
```

- 使用 `useFetchers` 监听所有活跃的 fetcher 请求
- 检测到主题切换请求时，立即返回新的主题值
- 实现乐观更新，无需等待服务端响应

#### 4.3.2 useTheme 组合逻辑

```typescript
export function useTheme() {
  const hints = useHints()
  const requestInfo = useRequestInfo()
  const optimisticMode = useOptimisticThemeMode()
  
  if (optimisticMode) {
    return optimisticMode === 'system' ? hints.theme : optimisticMode
  }
  return requestInfo.userPrefs.theme ?? hints.theme
}
```

**优先级顺序**：
1. **乐观模式**（如果有正在进行的主题切换请求）
2. **用户显式偏好**（从 cookie 读取的 `userPrefs.theme`）
3. **客户端提示**（从系统获取的 `hints.theme`）

### 4.4 数据流总结

```
┌─────────────────────────────────────────────────────────────────────┐
│                         主题偏好变化触发源                              │
├─────────────────────────┬───────────────────────────────────────────┤
│    系统主题变化           │         用户手动切换主题                    │
│  (prefers-color-scheme) │  (点击 ThemeSwitch 按钮)                   │
├─────────────────────────┼───────────────────────────────────────────┤
│  subscribeToSchemeChange │      useFetcher 提交 POST 请求            │
│      → revalidate()      │         → /resources/theme-switch         │
├─────────────────────────┴───────────────────────────────────────────┤
│                         数据重新校验 (Revalidation)                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  1. root loader 重新执行                                       │   │
│  │  2. getHints(request) 从 cookie 获取最新 hints                 │   │
│  │  3. getTheme(request) 从 cookie 获取最新用户偏好                │   │
│  │  4. 返回新的 requestInfo 数据                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
├───────────────────────────────────────────────────────────────────────┤
│                         客户端状态更新                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  useTheme() 重新计算：                                         │   │
│  │    optimisticMode → userPrefs.theme → hints.theme            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ↓                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Document 组件的 theme prop 更新，html className 变化           │   │
│  │  <html lang="en" className={`${theme} h-full overflow-x-hidden`}>│
│  └─────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────┘
```

---

### 4.5 时区 Hints 变化的处理机制

与主题不同，时区 hints **没有实时订阅机制**，只能通过页面刷新来更新。

#### 4.5.1 getClientHintCheckScript 的工作原理

`ClientHintCheck` 组件注入的内联脚本是时区 hints 检测的核心：

```typescript
// client-hints.tsx:48-55
return (
  <script
    nonce={nonce}
    dangerouslySetInnerHTML={{
      __html: hintsUtils.getClientHintCheckScript(),
    }}
  />
)
```

**脚本执行逻辑**（根据 `@epic-web/client-hints` 源码）：

```javascript
// 伪代码展示 getClientHintCheckScript 生成的脚本逻辑
(function() {
  const hints = {
    theme: window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light',
    timeZone: Intl.DateTimeFormat().resolvedOptions().timeZone,
  };
  
  const cookieValue = document.cookie;
  let needsReload = false;
  
  for (const [key, hint] of Object.entries(hintsConfig)) {
    const currentValue = hints[key];
    const cookieName = hint.cookieName;
    
    // 检查 cookie 中是否有该 hint
    const cookieMatch = document.cookie.match(new RegExp(`${cookieName}=([^;]+)`));
    const storedValue = cookieMatch ? cookieMatch[1] : null;
    
    // 如果没有存储或值不匹配，设置 cookie 并标记需要刷新
    if (!storedValue || storedValue !== currentValue) {
      document.cookie = `${cookieName}=${currentValue}; path=/; max-age=31536000`;
      needsReload = true;
    }
  }
  
  if (needsReload) {
    window.location.reload();
  }
})();
```

#### 4.5.2 时区变化的触发场景

时区变化**不会自动触发更新**，只能在以下时机检测：

| 场景 | 时区变化是否会被检测 | 处理方式 |
|------|---------------------|---------|
| 首次访问 | ✅ 检测并设置 cookie | 页面刷新（如果之前无 cookie） |
| 页面刷新/导航 | ✅ 检测 cookie 与当前值是否匹配 | 不匹配则刷新页面 |
| 用户在系统中更改时区（页面已打开） | ❌ 不会实时检测 | 需手动刷新页面 |
| 用户跨时区旅行后重新访问 | ✅ 下次页面加载时检测 | 检测到时区变化后刷新 |

#### 4.5.3 时区 Hint 的定义

时区 hint 的定义非常简单，没有订阅机制：

```typescript
// @epic-web/client-hints/time-zone 源码
import { type ClientHint } from '@epic-web/client-hints'

export const clientHint = {
  cookieName: 'CH-time-zone',
  getValueCode: 'Intl.DateTimeFormat().resolvedOptions().timeZone',
  fallback: 'UTC',
} as const satisfies ClientHint<string>
```

相比之下，主题 hint 支持订阅是因为它需要监听媒体查询变化：

```typescript
// @epic-web/client-hints/color-scheme 支持订阅的原因
// 主题可以通过 window.matchMedia 监听变化
const mq = window.matchMedia('(prefers-color-scheme: dark)')
mq.addEventListener('change', callback)
```

**时区无法实时订阅的技术原因**：
- JavaScript 没有原生 API 来监听时区变化事件
- `Intl.DateTimeFormat().resolvedOptions().timeZone` 只能同步获取当前值
- 没有类似 `matchMedia('(prefers-time-zone: ...)')` 的媒体查询

#### 4.5.4 时区数据流

```
┌─────────────────────────────────────────────────────────────────────┐
│                      时区 Hints 变化触发源                             │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   只有以下时机才会检测时区变化：                                        │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  1. 首次访问页面                                                │   │
│   │  2. 页面刷新 (F5 / window.location.reload())                  │   │
│   │  3. 导航到新页面 (React Router 的导航不算，需要整页加载)        │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│                              ↓                                        │
│                                                                       │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  ClientHintCheck 内联脚本执行 (getClientHintCheckScript)     │   │
│   │  ┌─────────────────────────────────────────────────────────┐ │   │
│   │  │ 1. 读取当前时区: Intl.DateTimeFormat().resolvedOptions() │ │   │
│   │  │ 2. 读取 cookie 中的 CH-time-zone 值                      │ │   │
│   │  │ 3. 比较两者是否一致                                        │ │   │
│   │  │ 4. 不一致 → 设置新 cookie + window.location.reload()     │ │   │
│   │  │ 5. 一致 → 不做任何操作                                    │ │   │
│   │  └─────────────────────────────────────────────────────────┘ │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│                              ↓ (如果需要刷新)                          │
│                                                                       │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                     页面整页刷新                               │   │
│   │  ┌─────────────────────────────────────────────────────────┐ │   │
│   │  │ 1. 浏览器重新发起请求                                      │ │   │
│   │  │ 2. 服务端收到请求，cookie 中已包含新的 CH-time-zone       │ │   │
│   │  │ 3. root loader 执行，getHints(request) 读取新时区         │ │   │
│   │  │ 4. 返回新的 requestInfo.hints.timeZone                    │ │   │
│   │  │ 5. 页面渲染时使用正确的时区                                 │ │   │
│   │  └─────────────────────────────────────────────────────────┘ │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

### 4.6 主题 vs 时区：完整对比总结

| 维度 | 主题 (Theme) | 时区 (TimeZone) |
|------|-------------|-----------------|
| **实时订阅** | ✅ `subscribeToSchemeChange` 监听 `prefers-color-scheme` 媒体查询 | ❌ 无原生 API 可监听时区变化 |
| **变化检测** | 实时检测，用户在系统设置中切换主题立即生效 | 仅页面加载时检测 |
| **更新机制** | `useRevalidator().revalidate()` → 重新运行 loader，无页面刷新 | `window.location.reload()` → 整页刷新 |
| **用户体验** | 平滑过渡，无闪烁，状态保持 | 页面重载，状态可能丢失（除非有恢复机制） |
| **首次访问** | 检测后刷新（与时区相同） | 检测后刷新 |
| **Cookie 名** | `CH-prefers-color-scheme` (hint) + `en_theme` (用户偏好) | `CH-time-zone` |
| **服务端获取** | `getHints(request).theme` + `getTheme(request)` (用户偏好) | `getHints(request).timeZone` |
| **常见使用场景** | UI 配色、深色/浅色模式切换 | 日期时间格式化、时间显示 |

**设计决策原因**：
1. **主题变化频繁**：用户可能频繁切换深色/浅色模式，需要平滑体验
2. **时区变化罕见**：用户跨时区旅行是低频事件，整页刷新的影响较小
3. **技术限制**：浏览器没有提供监听时区变化的原生 API，而主题变化可以通过 `matchMedia` 监听

---

## 5. 关键文件索引

| 功能 | 文件路径 |
|------|----------|
| Nonce Context 定义 | `app/utils/nonce-provider.ts` |
| 服务端入口（nonce 生成、渲染策略） | `app/entry.server.tsx` |
| Root Loader（hints 下发） | `app/root.tsx` |
| 客户端 Hints 工具 | `app/utils/client-hints.tsx` |
| 主题服务端工具 | `app/utils/theme.server.ts` |
| 主题切换资源路由 | `app/routes/resources/theme-switch.tsx` |
