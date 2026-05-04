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

- 从 `en_theme` cookie 读取用户显式设置的主题
- 默认返回 `'light'`，如果值无效则返回 `null`

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

## 5. 关键文件索引

| 功能 | 文件路径 |
|------|----------|
| Nonce Context 定义 | `app/utils/nonce-provider.ts` |
| 服务端入口（nonce 生成、渲染策略） | `app/entry.server.tsx` |
| Root Loader（hints 下发） | `app/root.tsx` |
| 客户端 Hints 工具 | `app/utils/client-hints.tsx` |
| 主题服务端工具 | `app/utils/theme.server.ts` |
| 主题切换资源路由 | `app/routes/resources/theme-switch.tsx` |
