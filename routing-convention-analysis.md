# Epic Stack 路由约定分析报告

> 基于当前仓库真实配置与目录结构分析

---

## 一、路由生成链路

### 1.1 配置文件位置

| 配置文件 | 路径 | 作用 |
|----------|------|------|
| React Router 全局配置 | `react-router.config.ts` | 配置 SSR、路由发现模式等 |
| 路由生成配置 | `app/routes.ts` | 配置 `react-router-auto-routes` 的忽略规则 |

### 1.2 生成链路图

```
react-router.config.ts
        │
        ▼
routeDiscovery: { mode: 'initial' }
        │
        ▼
    读取 app/routes/ 目录
        │
        ▼
    app/routes.ts 配置
        │
        ▼
    autoRoutes({ ignoredRouteFiles: [...] })
        │
        ▼
    扫描文件 → 应用忽略规则 → 解析命名约定
        │
        ▼
    生成路由清单（Routes）
```

### 1.3 真实配置内容

**`react-router.config.ts`（第 1-25 行）：**

```typescript
import { type Config } from '@react-router/dev/config'
import { sentryOnBuildEnd } from '@sentry/react-router'

const MODE = process.env.NODE_ENV

export default {
  ssr: true,
  routeDiscovery: { mode: 'initial' },
  future: {
    unstable_optimizeDeps: true,
  },
  buildEnd: async ({ viteConfig, reactRouterConfig, buildManifest }) => {
    if (MODE === 'production' && process.env.SENTRY_AUTH_TOKEN) {
      await sentryOnBuildEnd({
        viteConfig,
        reactRouterConfig,
        buildManifest,
      })
    }
  },
} satisfies Config
```

**`app/routes.ts`（第 1-18 行）：**

```typescript
import { type RouteConfig } from '@react-router/dev/routes'
import { autoRoutes } from 'react-router-auto-routes'

export default autoRoutes({
  ignoredRouteFiles: [
    '.*',
    '**/*.css',
    '**/*.test.{js,jsx,ts,tsx}',
    '**/__*.*',
    // This is for server-side utilities you want to colocate
    // next to your routes without making an additional
    // directory. If you need a route that includes "server" or
    // "client" in the filename, use the escape brackets like:
    // my-route.[server].tsx
    '**/*.server.*',
    '**/*.client.*',
  ],
}) satisfies RouteConfig
```

### 1.4 忽略规则明细

| 模式 | 匹配示例 | 说明 |
|------|----------|------|
| `.*` | `.gitkeep`, `.env` | 隐藏文件 |
| `**/*.css` | `styles.css`, `tailwind.css` | 样式文件 |
| `**/*.test.{js,jsx,ts,tsx}` | `callback.test.ts`, `index.test.tsx` | 测试文件 |
| `**/__*.*` | `__utils.ts`, `__helpers.js` | 双下划线前缀文件 |
| `**/*.server.*` | `verify.server.ts`, `login.server.ts` | 服务端专用模块 |
| `**/*.client.*` | `utils.client.ts` | 客户端专用模块 |

### 1.5 生成后的路由清单（基于 docs/routing.md）

```tsx
<Routes>
  <Route file="root.tsx">
    <Route path="*" file="routes/$.tsx" />
    <Route path="auth/:provider/callback" file="routes/_auth/auth.$provider/callback.ts" />
    <Route path="auth/:provider" index file="routes/_auth/auth.$provider/index.ts" />
    <Route path="forgot-password" file="routes/_auth/forgot-password.tsx" />
    <Route path="login" file="routes/_auth/login.tsx" />
    <Route path="logout" file="routes/_auth/logout.tsx" />
    <Route path="onboarding/:provider" file="routes/_auth/onboarding/$provider.tsx" />
    <Route path="onboarding" index file="routes/_auth/onboarding/index.tsx" />
    <Route path="reset-password" file="routes/_auth/reset-password.tsx" />
    <Route path="signup" file="routes/_auth/signup.tsx" />
    <Route path="verify" file="routes/_auth/verify.tsx" />
    <Route path="webauthn/authentication" file="routes/_auth/webauthn/authentication.ts" />
    <Route path="webauthn/registration" file="routes/_auth/webauthn/registration.ts" />
    <Route path="about" file="routes/_marketing/about.tsx" />
    <Route index file="routes/_marketing/index.tsx" />
    <Route path="privacy" file="routes/_marketing/privacy.tsx" />
    <Route path="support" file="routes/_marketing/support.tsx" />
    <Route path="tos" file="routes/_marketing/tos.tsx" />
    <Route path="robots.txt" file="routes/_seo/robots[.]txt.ts" />
    <Route path="sitemap.xml" file="routes/_seo/sitemap[.]xml.ts" />
    <Route path="admin/cache" index file="routes/admin/cache/index.tsx" />
    <Route path="admin/cache/lru/:cacheKey" file="routes/admin/cache/lru.$cacheKey.ts" />
    <Route path="admin/cache/sqlite" file="routes/admin/cache/sqlite.tsx">
      <Route path=":cacheKey" file="routes/admin/cache/sqlite.$cacheKey.ts" />
    </Route>
    <Route path="me" file="routes/me.tsx" />
    <Route path="resources/download-user-data" file="routes/resources/download-user-data.tsx" />
    <Route path="resources/healthcheck" file="routes/resources/healthcheck.tsx" />
    <Route path="resources/images" file="routes/resources/images.tsx" />
    <Route path="resources/theme-switch" file="routes/resources/theme-switch.tsx" />
    <Route path="settings/profile" file="routes/settings/profile/_layout.tsx">
      <Route path="change-email" file="routes/settings/profile/change-email.tsx" />
      <Route path="connections" file="routes/settings/profile/connections.tsx" />
      <Route index file="routes/settings/profile/index.tsx" />
      <Route path="passkeys" file="routes/settings/profile/passkeys.tsx" />
      <Route path="password" file="routes/settings/profile/password.tsx" />
      <Route path="password/create" file="routes/settings/profile/password_.create.tsx" />
      <Route path="photo" file="routes/settings/profile/photo.tsx" />
      <Route path="two-factor" file="routes/settings/profile/two-factor/_layout.tsx">
        <Route path="disable" file="routes/settings/profile/two-factor/disable.tsx" />
        <Route index file="routes/settings/profile/two-factor/index.tsx" />
        <Route path="verify" file="routes/settings/profile/two-factor/verify.tsx" />
      </Route>
    </Route>
    <Route path="users/:username" index file="routes/users/$username/index.tsx" />
    <Route path="users/:username/notes" file="routes/users/$username/notes/_layout.tsx">
      <Route path=":noteId" file="routes/users/$username/notes/$noteId.tsx" />
      <Route path=":noteId/edit" file="routes/users/$username/notes/$noteId_.edit.tsx" />
      <Route index file="routes/users/$username/notes/index.tsx" />
      <Route path="new" file="routes/users/$username/notes/new.tsx" />
    </Route>
    <Route path="users" index file="routes/users/index.tsx" />
  </Route>
</Routes>
```

---

## 二、对照表：会参与路由发现 vs 被忽略但可共置

### 2.1 会参与路由发现的约定

| 约定符号 | 规则说明 | 真实文件示例 | 生成的 URL 路径 |
|----------|----------|--------------|-----------------|
| `$` 前缀（动态参数） | 文件名或目录名以 `$` 开头，映射为 URL 动态段 | `$username/`, `$noteId.tsx`, `$cacheKey.ts` | `/users/:username`, `/:noteId`, `/:cacheKey` |
| `_` 前缀（路径组目录） | 目录名以 `_` 开头，不影响 URL，仅用于组织 | `_auth/`, `_marketing/`, `_seo/` | 不影响（其子路由路径不含 `_auth` 等前缀） |
| `_layout.tsx`（布局文件） | 目录内的 `_layout.tsx` 作为该级别的布局组件 | `settings/profile/_layout.tsx`, `users/$username/notes/_layout.tsx` | 作为父路由，路径为目录路径 |
| `index.tsx`（索引路由） | 目录内的 `index.tsx` 作为该目录的索引路由 | `users/index.tsx`, `_marketing/index.tsx` | 对应目录路径（如 `/users`, `/`） |
| `$`（通配符） | 根级 `$.tsx` 作为兜底路由 | `$.tsx` | `/*` |
| `[.]`（转义点号） | `[.]` 转义为真实点号 | `robots[.]txt.ts`, `sitemap[.]xml.ts` | `/robots.txt`, `/sitemap.xml` |
| `_` 后缀（路径后缀） | 文件名末尾 `_` 后接内容，映射为额外路径段 | `password_.create.tsx`, `$noteId_.edit.tsx` | `/password/create`, `/:noteId/edit` |
| 普通文件 | 不含特殊符号的 `.ts`/`.tsx` 文件 | `login.tsx`, `logout.tsx`, `about.tsx` | `/login`, `/logout`, `/about` |
| 普通目录 | 不含特殊前缀的目录 | `admin/`, `resources/`, `settings/` | 嵌套路由层次（如 `/admin/cache`） |

### 2.2 被忽略但可共置的约定

| 约定符号 | 规则说明 | 真实文件示例 | 用途说明 |
|----------|----------|--------------|----------|
| `+` 前缀目录 | 目录名以 `+` 开头，不会被识别为路由 | `+logos/`, `+shared/` | 存放与路由相关的共享代码、资源 |
| `.server.` | 文件名包含 `.server.` | `verify.server.ts`, `login.server.ts`, `index.server.ts`, `note-editor.server.tsx` | 服务端专用逻辑，仅在服务端执行，不会泄露到客户端 |
| `.client.` | 文件名包含 `.client.` | （当前仓库无实例，配置支持） | 客户端专用逻辑，仅在浏览器执行 |
| `.test.` | 文件名包含 `.test.` | `callback.test.ts`, `index.test.tsx` | 测试文件，与路由共置但不参与路由 |
| `.css` | 扩展名为 `.css` | （当前仓库 routes 目录无，配置支持） | 样式文件，通过其他方式引入 |
| `.*` | 隐藏文件（以 `.` 开头） | （当前仓库 routes 目录无） | Git 忽略文件、环境文件等 |
| `__*` | 双下划线前缀 | （当前仓库 routes 目录无） | 内部工具文件 |

---

## 三、真实目录结构与路由映射对照

### 3.1 完整目录结构

```
app/routes/
├── $.tsx                                    → /*
├── me.tsx                                   → /me
├── _auth/                                   ← 路径组（不影响 URL）
│   ├── auth.$provider/
│   │   ├── callback.test.ts                 ← 测试文件（忽略）
│   │   ├── callback.ts                      → /auth/:provider/callback
│   │   └── index.ts                         → /auth/:provider（索引）
│   ├── onboarding/
│   │   ├── $provider.server.ts              ← 服务端模块（忽略）
│   │   ├── $provider.tsx                    → /onboarding/:provider
│   │   ├── index.server.ts                  ← 服务端模块（忽略）
│   │   └── index.tsx                        → /onboarding（索引）
│   ├── webauthn/
│   │   ├── authentication.ts                → /webauthn/authentication
│   │   ├── registration.ts                  → /webauthn/registration
│   │   └── utils.server.ts                  ← 服务端模块（忽略）
│   ├── forgot-password.tsx                  → /forgot-password
│   ├── login.server.ts                      ← 服务端模块（忽略）
│   ├── login.tsx                            → /login
│   ├── logout.tsx                           → /logout
│   ├── reset-password.server.ts             ← 服务端模块（忽略）
│   ├── reset-password.tsx                   → /reset-password
│   ├── signup.tsx                           → /signup
│   ├── verify.server.ts                     ← 服务端模块（忽略）
│   └── verify.tsx                           → /verify
├── _marketing/                              ← 路径组（不影响 URL）
│   ├── +logos/                              ← 共置目录（忽略）
│   │   ├── logos.ts
│   │   ├── docker.svg
│   │   ├── remix.svg
│   │   └── ...
│   ├── about.tsx                            → /about
│   ├── index.tsx                            → /（索引）
│   ├── privacy.tsx                          → /privacy
│   ├── support.tsx                          → /support
│   └── tos.tsx                              → /tos
├── _seo/                                    ← 路径组（不影响 URL）
│   ├── robots[.]txt.ts                      → /robots.txt
│   └── sitemap[.]xml.ts                     → /sitemap.xml
├── admin/
│   └── cache/
│       ├── index.tsx                        → /admin/cache（索引）
│       ├── lru.$cacheKey.ts                 → /admin/cache/lru/:cacheKey
│       ├── sqlite.$cacheKey.ts              → /admin/cache/sqlite/:cacheKey
│       ├── sqlite.server.ts                 ← 服务端模块（忽略）
│       └── sqlite.tsx                       → /admin/cache/sqlite
├── resources/
│   ├── download-user-data.tsx               → /resources/download-user-data
│   ├── healthcheck.tsx                      → /resources/healthcheck
│   ├── images.tsx                           → /resources/images
│   └── theme-switch.tsx                     → /resources/theme-switch
├── settings/
│   └── profile/
│       ├── two-factor/
│       │   ├── _layout.tsx                  → /settings/profile/two-factor（布局）
│       │   ├── disable.tsx                  → /settings/profile/two-factor/disable
│       │   ├── index.tsx                    → /settings/profile/two-factor（索引）
│       │   └── verify.tsx                   → /settings/profile/two-factor/verify
│       ├── _layout.tsx                      → /settings/profile（布局）
│       ├── change-email.server.tsx          ← 服务端模块（忽略）
│       ├── change-email.tsx                 → /settings/profile/change-email
│       ├── connections.tsx                  → /settings/profile/connections
│       ├── index.tsx                        → /settings/profile（索引）
│       ├── passkeys.tsx                     → /settings/profile/passkeys
│       ├── password.tsx                     → /settings/profile/password
│       ├── password_.create.tsx             → /settings/profile/password/create
│       └── photo.tsx                        → /settings/profile/photo
└── users/
    ├── $username/                           ← 动态参数
    │   ├── notes/
    │   │   ├── +shared/                     ← 共置目录（忽略）
    │   │   │   ├── note-editor.server.tsx   ← 服务端模块（忽略）
    │   │   │   └── note-editor.tsx
    │   │   ├── $noteId.tsx                  → /users/:username/notes/:noteId
    │   │   ├── $noteId_.edit.tsx            → /users/:username/notes/:noteId/edit
    │   │   ├── _layout.tsx                  → /users/:username/notes（布局）
    │   │   ├── index.tsx                    → /users/:username/notes（索引）
    │   │   └── new.tsx                      → /users/:username/notes/new
    │   ├── index.test.tsx                   ← 测试文件（忽略）
    │   └── index.tsx                        → /users/:username（索引）
    └── index.tsx                            → /users（索引）
```

---

## 四、共置模式详解

### 4.1 `+` 前缀目录共置

**示例 1：`+logos/` - 资源集合**

位置：`app/routes/_marketing/+logos/`

```
+logos/
├── logos.ts        # 图标数据配置模块
├── docker.svg
├── eslint.svg
├── remix.svg
├── tailwind.svg
└── ...
```

使用方式（`_marketing/index.tsx`）：
```typescript
import { logos, stars } from './+logos/logos.ts'
```

**示例 2：`+shared/` - 共享组件**

位置：`app/routes/users/$username/notes/+shared/`

```
+shared/
├── note-editor.tsx          # 笔记编辑器组件
└── note-editor.server.tsx   # 服务端 action 逻辑
```

被多个路由复用：
- `$noteId_.edit.tsx` - 编辑笔记
- `new.tsx` - 新建笔记

### 4.2 `.server.` 模块共置

**模式 A：独立服务端文件**

```
verify.tsx              # 路由组件（客户端+服务端）
verify.server.ts        # 服务端逻辑（被忽略，可被路由文件 import）
```

**`verify.server.ts` 真实内容（第 1-200 行）：**
```typescript
export async function validateRequest(request: Request, body: URLSearchParams | FormData) {
  // 纯服务端逻辑：数据库操作、验证等
}

export async function prepareVerification({ ... }) { ... }
export async function isCodeValid({ ... }) { ... }
```

**模式 B：路由与服务端逻辑配对**

```
onboarding/
├── index.tsx           # 路由组件
├── index.server.ts     # 服务端 action
├── $provider.tsx       # 动态路由组件
└── $provider.server.ts # 动态路由服务端 action
```

### 4.3 `.test.` 文件共置

```
auth.$provider/
├── callback.ts         # 路由文件
└── callback.test.ts    # 测试文件（被忽略）

users/$username/
├── index.tsx           # 路由文件
└── index.test.tsx      # 测试文件（被忽略）
```

### 4.4 `[server]` 转义（配置说明）

根据 `app/routes.ts` 注释（第 10-14 行）：

> If you need a route that includes "server" or "client" in the filename, use the escape brackets like: my-route.[server].tsx

| 需要的路由路径 | 使用方式 | 说明 |
|----------------|----------|------|
| `/my-route/server` | `my-route.[server].tsx` | 方括号内的内容会被当作普通路径段 |
| `/my-route/client` | `my-route.[client].tsx` | 避免被 `*.server.*` / `*.client.*` 规则忽略 |

**当前仓库未使用此转义，所有 `.server.` 文件均为服务端专用模块。**

---

## 五、核心约定速查

### 5.1 路由发现符号

| 符号 | 位置 | 作用 |
|------|------|------|
| `$` | 文件名/目录名前缀 | 动态 URL 参数 |
| `_` | 目录名前缀 | 路径组（不影响 URL） |
| `_` | 文件名后缀 | 额外路径段 |
| `_layout.tsx` | 文件名 | 布局组件 |
| `index.tsx` | 文件名 | 索引路由 |
| `[.]` | 文件名内 | 转义点号 |
| `$` | 根级文件名 | 通配符路由 |

### 5.2 忽略规则符号

| 符号/模式 | 作用 | 可共置？ |
|-----------|------|----------|
| `+` 目录前缀 | 共置资源目录 | ✅ 是 |
| `*.server.*` | 服务端专用模块 | ✅ 是 |
| `*.client.*` | 客户端专用模块 | ✅ 是 |
| `*.test.*` | 测试文件 | ✅ 是 |
| `*.css` | 样式文件 | ❌ 否 |
| `.*` | 隐藏文件 | ❌ 否 |
| `__*.*` | 双下划线文件 | ❌ 否 |
