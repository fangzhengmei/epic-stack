# Epic Stack 路由约定分析报告

> 基于当前仓库真实配置与目录结构分析
> 版本：2026-05-11

---

## 一、配置来源

### 1.1 核心配置文件

| 配置文件 | 路径 | 作用 |
|----------|------|------|
| React Router 全局配置 | `react-router.config.ts:10` | `routeDiscovery: { mode: 'initial' }` |
| 路由生成配置 | `app/routes.ts:4-17` | 配置 `react-router-auto-routes` 及忽略规则 |

### 1.2 真实配置内容

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
    '**/*.server.*',
    '**/*.client.*',
  ],
}) satisfies RouteConfig
```

---

## 二、路由生成链路

```
react-router.config.ts
        │
        ▼
routeDiscovery: { mode: 'initial' }
        │
        ▼
    扫描 app/routes/ 目录
        │
        ▼
    app/routes.ts 配置
        │
        ├─► react-router-auto-routes 内置约定
        │       ├─► `+` 前缀目录：不参与路由发现
        │       └─► 文件命名解析：$、_、[.] 等
        │
        └─► ignoredRouteFiles 配置
                ├─► `.*`：隐藏文件（不可共置）
                ├─► `**/*.css`：样式文件（不可共置）
                ├─► `**/*.test.*`：测试文件（可共置）
                ├─► `**/__*.*`：双下划线文件（不可共置）
                ├─► `**/*.server.*`：服务端模块（可共置）
                └─► `**/*.client.*`：客户端模块（可共置）
        │
        ▼
    生成最终路由清单
```

---

## 三、会参与路由发现的约定

> 这些文件/目录会被解析为路由，出现在最终路由清单中

### 3.1 目录级约定

| 约定符号 | 规则说明 | 真实目录示例 | 对 URL 的影响 |
|----------|----------|--------------|---------------|
| `_` 前缀目录 | 路径组（Route Group），组织路由但不影响 URL | `_auth/`, `_marketing/`, `_seo/` | 不影响（URL 不含 `_auth` 等前缀） |
| 普通目录 | 嵌套路由层次 | `admin/`, `resources/`, `settings/` | 创建嵌套路由（如 `/admin/cache`） |
| `$` 前缀目录 | 动态参数目录 | `$username/` | `/users/:username` |

### 3.2 文件级约定

| 约定符号 | 规则说明 | 真实文件示例 | 生成的 URL 路径 |
|----------|----------|--------------|-----------------|
| `_layout.tsx` | 目录级布局组件，通过 `<Outlet />` 渲染子路由 | `settings/profile/_layout.tsx`, `users/$username/notes/_layout.tsx` | 作为父路由，路径为目录路径 |
| `index.tsx` | 目录的索引路由 | `users/index.tsx`, `_marketing/index.tsx` | 对应目录路径（如 `/users`, `/`） |
| `$` 前缀文件 | 动态参数文件 | `$noteId.tsx`, `$cacheKey.ts` | `/:noteId`, `/:cacheKey` |
| `_` 后缀 | 额外路径段 | `password_.create.tsx`, `$noteId_.edit.tsx` | `/password/create`, `/:noteId/edit` |
| `[.]` | 转义点号 | `robots[.]txt.ts`, `sitemap[.]xml.ts` | `/robots.txt`, `/sitemap.xml` |
| `$`（根级） | 通配符/兜底路由 | `$.tsx` | `/*` |
| 普通 `.ts/.tsx` | 不含特殊符号的路由文件 | `login.tsx`, `logout.tsx`, `about.tsx` | `/login`, `/logout`, `/about` |

### 3.3 真实项目路由清单（基于目录结构推导）

根据 `app/routes/` 目录结构，生成的路由清单如下：

```tsx
<Routes>
  <Route file="root.tsx">
    <Route path="*" file="routes/$.tsx" />
    <Route path="me" file="routes/me.tsx" />
    
    {/* _auth 路径组 */}
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
    
    {/* _marketing 路径组 */}
    <Route path="about" file="routes/_marketing/about.tsx" />
    <Route index file="routes/_marketing/index.tsx" />
    <Route path="privacy" file="routes/_marketing/privacy.tsx" />
    <Route path="support" file="routes/_marketing/support.tsx" />
    <Route path="tos" file="routes/_marketing/tos.tsx" />
    
    {/* _seo 路径组 */}
    <Route path="robots.txt" file="routes/_seo/robots[.]txt.ts" />
    <Route path="sitemap.xml" file="routes/_seo/sitemap[.]xml.ts" />
    
    {/* admin 嵌套路由 */}
    <Route path="admin/cache" index file="routes/admin/cache/index.tsx" />
    <Route path="admin/cache/lru/:cacheKey" file="routes/admin/cache/lru.$cacheKey.ts" />
    <Route path="admin/cache/sqlite" file="routes/admin/cache/sqlite.tsx">
      <Route path=":cacheKey" file="routes/admin/cache/sqlite.$cacheKey.ts" />
    </Route>
    
    {/* resources 路由 */}
    <Route path="resources/download-user-data" file="routes/resources/download-user-data.tsx" />
    <Route path="resources/healthcheck" file="routes/resources/healthcheck.tsx" />
    <Route path="resources/images" file="routes/resources/images.tsx" />
    <Route path="resources/theme-switch" file="routes/resources/theme-switch.tsx" />
    
    {/* settings/profile 嵌套布局 */}
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
    
    {/* users 嵌套路由 */}
    <Route path="users" index file="routes/users/index.tsx" />
    <Route path="users/:username" index file="routes/users/$username/index.tsx" />
    <Route path="users/:username/notes" file="routes/users/$username/notes/_layout.tsx">
      <Route path=":noteId" file="routes/users/$username/notes/$noteId.tsx" />
      <Route path=":noteId/edit" file="routes/users/$username/notes/$noteId_.edit.tsx" />
      <Route index file="routes/users/$username/notes/index.tsx" />
      <Route path="new" file="routes/users/$username/notes/new.tsx" />
    </Route>
  </Route>
</Routes>
```

---

## 四、被忽略的约定

> 这些文件/目录**不会**被解析为路由

### 4.1 分类说明

被忽略的文件分为两类：

| 类别 | 来源 | 可被路由文件 import（共置） |
|------|------|----------------------------|
| **A. 库内置约定** | `react-router-auto-routes` | ✅ 是 |
| **B. ignoredRouteFiles 配置** | `app/routes.ts:5-17` | 部分是 |

### 4.2 完整对照表

| 约定符号 | 来源 | 真实文件/目录示例 | 是否可共置 | 说明 |
|----------|------|-------------------|------------|------|
| `+` 前缀目录 | 库内置 | `+logos/`, `+shared/` | ✅ 是 | 存放共享组件、资源 |
| `*.server.*` | 配置（ignored） | `verify.server.ts`, `login.server.ts`, `note-editor.server.tsx` | ✅ 是 | 服务端专用逻辑 |
| `*.client.*` | 配置（ignored） | （当前仓库无实例） | ✅ 是 | 客户端专用逻辑 |
| `*.test.*` | 配置（ignored） | `callback.test.ts`, `index.test.tsx` | ✅ 是 | 测试文件 |
| `*.css` | 配置（ignored） | （当前仓库 routes 目录无） | ❌ 否 | 样式文件 |
| `.*` | 配置（ignored） | （当前仓库 routes 目录无） | ❌ 否 | 隐藏文件 |
| `__*.*` | 配置（ignored） | （当前仓库 routes 目录无） | ❌ 否 | 双下划线文件 |
| `[server]/[client]` 转义 | 配置说明 | （当前仓库无实例） | 不适用 | 特殊转义，**不被忽略**，反而会生成路由 |

---

## 五、共置模式详解

### 5.1 `+` 前缀目录（库内置约定）

**说明**：`+` 前缀目录是 `react-router-auto-routes` 的内置约定，不在 `ignoredRouteFiles` 配置中。

**真实示例 1：`+logos/` - 资源集合**

位置：`app/routes/_marketing/+logos/`

```
+logos/
├── logos.ts        # 图标数据配置
├── docker.svg
├── remix.svg
├── tailwind.svg
└── ... (21 个资源文件)
```

引用方式（`_marketing/index.tsx`）：
```typescript
import { logos, stars } from './+logos/logos.ts'
```

**真实示例 2：`+shared/` - 共享组件**

位置：`app/routes/users/$username/notes/+shared/`

```
+shared/
├── note-editor.tsx          # 笔记编辑器组件
└── note-editor.server.tsx   # 服务端 action 逻辑
```

被多个路由文件引用：

**`new.tsx`（第 1-12 行）：**
```typescript
import { requireUserId } from '#app/utils/auth.server.ts'
import { NoteEditor } from './+shared/note-editor.tsx'
import { type Route } from './+types/new.ts'

export { action } from './+shared/note-editor.server.tsx'

export async function loader({ request }: Route.LoaderArgs) {
  await requireUserId(request)
  return {}
}

export default NoteEditor
```

**`$noteId_.edit.tsx`（第 1-9 行）：**
```typescript
import { invariantResponse } from '@epic-web/invariant'
import { GeneralErrorBoundary } from '#app/components/error-boundary.tsx'
import { requireUserId } from '#app/utils/auth.server.ts'
import { prisma } from '#app/utils/db.server.ts'
import { NoteEditor } from './+shared/note-editor.tsx'
import { type Route } from './+types/$noteId_.edit.ts'

export { action } from './+shared/note-editor.server.tsx'
```

**真实示例 3：`+types/` - 类型定义（运行时生成）**

代码中引用示例（`users/$username/notes/index.tsx:1-2`）：
```typescript
import { type Route as NotesRoute } from './+types/_layout.ts'
import { type Route } from './+types/index.ts'
```

说明：
- `+types/` 目录由 `npm run typecheck` 或 `npx react-router typegen` 自动生成
- 当前仓库中未提交到 Git（`.gitignore:26` 包含 `.react-router/`）
- 属于 `+` 前缀目录约定，不参与路由发现

### 5.2 `.server.` 模块（ignoredRouteFiles 配置）

**说明**：`**/*.server.*` 在 `app/routes.ts:15` 中被配置为忽略，可被路由文件 import 共置。

**真实示例 1：独立服务端文件**

```
verify.tsx              # 路由文件（参与路由发现）
verify.server.ts        # 服务端逻辑（被忽略，可被引用）
```

**`verify.tsx` 引用 `verify.server.ts`（第 14 行）：**
```typescript
import { validateRequest } from './verify.server.ts'
```

**`verify.server.ts` 真实内容（第 140-199 行，核心函数）：**
```typescript
export async function validateRequest(
  request: Request,
  body: URLSearchParams | FormData,
) {
  const submission = await parseWithZod(body, { ... })
  // 纯服务端逻辑：数据库操作、验证等
}
```

**真实示例 2：路由与服务端逻辑配对**

```
onboarding/
├── index.tsx           # 路由文件
├── index.server.ts     # 服务端 action（被忽略）
├── $provider.tsx       # 动态路由文件
└── $provider.server.ts # 动态路由服务端 action（被忽略）
```

**真实示例 3：嵌套引用**

`verify.server.ts` 引用其他 `.server.` 文件（第 5-18 行）：
```typescript
import { handleVerification as handleChangeEmailVerification } 
  from '#app/routes/settings/profile/change-email.server.tsx'
import { handleVerification as handleOnboardingVerification } 
  from './onboarding/index.server.ts'
import { handleVerification as handleResetPasswordVerification } 
  from './reset-password.server.ts'
```

### 5.3 `.test.` 文件（ignoredRouteFiles 配置）

**说明**：`**/*.test.{js,jsx,ts,tsx}` 在 `app/routes.ts:8` 中被配置为忽略。

**真实示例：**

```
_auth/auth.$provider/
├── callback.ts         # 路由文件
└── callback.test.ts    # 测试文件（被忽略）

users/$username/
├── index.tsx           # 路由文件
└── index.test.tsx      # 测试文件（被忽略）
```

### 5.4 `[server]` / `[client]` 转义（配置说明，无真实实例）

**`app/routes.ts` 注释（第 10-14 行）：**

```typescript
// This is for server-side utilities you want to colocate
// next to your routes without making an additional
// directory. If you need a route that includes "server" or
// "client" in the filename, use the escape brackets like:
// my-route.[server].tsx
```

**关键点**：
- `[server]` / `[client]` 是**转义机制**，不是忽略规则
- `my-route.[server].tsx` **会**生成路由 `/my-route/server`
- 目的是避免被 `**/*.server.*` 规则忽略
- 当前仓库**未使用**此转义，所有 `.server.` 文件均为服务端专用模块

---

## 六、真实目录结构完整标注

```
app/routes/
├── $.tsx                                    ✅ 参与路由 → /*
├── me.tsx                                   ✅ 参与路由 → /me
│
├── _auth/                                   ✅ 路径组（不影响 URL）
│   ├── auth.$provider/                      ✅ 动态参数目录
│   │   ├── callback.test.ts                 ❌ 被忽略（.test.）
│   │   ├── callback.ts                      ✅ 参与路由 → /auth/:provider/callback
│   │   └── index.ts                         ✅ 参与路由 → /auth/:provider（索引）
│   ├── onboarding/                          ✅ 普通目录
│   │   ├── $provider.server.ts              ❌ 被忽略（.server.）
│   │   ├── $provider.tsx                    ✅ 参与路由 → /onboarding/:provider
│   │   ├── index.server.ts                  ❌ 被忽略（.server.）
│   │   └── index.tsx                        ✅ 参与路由 → /onboarding（索引）
│   ├── webauthn/                            ✅ 普通目录
│   │   ├── authentication.ts                ✅ 参与路由 → /webauthn/authentication
│   │   ├── registration.ts                  ✅ 参与路由 → /webauthn/registration
│   │   └── utils.server.ts                  ❌ 被忽略（.server.）
│   ├── forgot-password.tsx                  ✅ 参与路由 → /forgot-password
│   ├── login.server.ts                      ❌ 被忽略（.server.）
│   ├── login.tsx                            ✅ 参与路由 → /login
│   ├── logout.tsx                           ✅ 参与路由 → /logout
│   ├── reset-password.server.ts             ❌ 被忽略（.server.）
│   ├── reset-password.tsx                   ✅ 参与路由 → /reset-password
│   ├── signup.tsx                           ✅ 参与路由 → /signup
│   ├── verify.server.ts                     ❌ 被忽略（.server.）
│   └── verify.tsx                           ✅ 参与路由 → /verify
│
├── _marketing/                              ✅ 路径组（不影响 URL）
│   ├── +logos/                              ❌ 被忽略（+ 前缀，库内置）
│   │   ├── logos.ts
│   │   ├── docker.svg
│   │   ├── remix.svg
│   │   └── ... (21 个资源文件)
│   ├── about.tsx                            ✅ 参与路由 → /about
│   ├── index.tsx                            ✅ 参与路由 → /（索引）
│   ├── privacy.tsx                          ✅ 参与路由 → /privacy
│   ├── support.tsx                          ✅ 参与路由 → /support
│   └── tos.tsx                              ✅ 参与路由 → /tos
│
├── _seo/                                    ✅ 路径组（不影响 URL）
│   ├── robots[.]txt.ts                      ✅ 参与路由 → /robots.txt
│   └── sitemap[.]xml.ts                     ✅ 参与路由 → /sitemap.xml
│
├── admin/                                   ✅ 普通目录
│   └── cache/                               ✅ 普通目录
│       ├── index.tsx                        ✅ 参与路由 → /admin/cache（索引）
│       ├── lru.$cacheKey.ts                 ✅ 参与路由 → /admin/cache/lru/:cacheKey
│       ├── sqlite.$cacheKey.ts              ✅ 参与路由 → /admin/cache/sqlite/:cacheKey
│       ├── sqlite.server.ts                 ❌ 被忽略（.server.）
│       └── sqlite.tsx                       ✅ 参与路由 → /admin/cache/sqlite
│
├── resources/                               ✅ 普通目录
│   ├── download-user-data.tsx               ✅ 参与路由 → /resources/download-user-data
│   ├── healthcheck.tsx                      ✅ 参与路由 → /resources/healthcheck
│   ├── images.tsx                           ✅ 参与路由 → /resources/images
│   └── theme-switch.tsx                     ✅ 参与路由 → /resources/theme-switch
│
├── settings/                                ✅ 普通目录
│   └── profile/                             ✅ 普通目录
│       ├── two-factor/                      ✅ 普通目录
│       │   ├── _layout.tsx                  ✅ 参与路由（布局）→ /settings/profile/two-factor
│       │   ├── disable.tsx                  ✅ 参与路由 → /settings/profile/two-factor/disable
│       │   ├── index.tsx                    ✅ 参与路由 → /settings/profile/two-factor（索引）
│       │   └── verify.tsx                   ✅ 参与路由 → /settings/profile/two-factor/verify
│       ├── _layout.tsx                      ✅ 参与路由（布局）→ /settings/profile
│       ├── change-email.server.tsx          ❌ 被忽略（.server.）
│       ├── change-email.tsx                 ✅ 参与路由 → /settings/profile/change-email
│       ├── connections.tsx                  ✅ 参与路由 → /settings/profile/connections
│       ├── index.tsx                        ✅ 参与路由 → /settings/profile（索引）
│       ├── passkeys.tsx                     ✅ 参与路由 → /settings/profile/passkeys
│       ├── password.tsx                     ✅ 参与路由 → /settings/profile/password
│       ├── password_.create.tsx             ✅ 参与路由 → /settings/profile/password/create
│       └── photo.tsx                        ✅ 参与路由 → /settings/profile/photo
│
└── users/                                   ✅ 普通目录
    ├── $username/                           ✅ 动态参数目录
    │   ├── notes/                           ✅ 普通目录
    │   │   ├── +shared/                     ❌ 被忽略（+ 前缀，库内置）
    │   │   │   ├── note-editor.server.tsx   ❌ 被忽略（.server.）
    │   │   │   └── note-editor.tsx
    │   │   ├── $noteId.tsx                  ✅ 参与路由 → /users/:username/notes/:noteId
    │   │   ├── $noteId_.edit.tsx            ✅ 参与路由 → /users/:username/notes/:noteId/edit
    │   │   ├── _layout.tsx                  ✅ 参与路由（布局）→ /users/:username/notes
    │   │   ├── index.tsx                    ✅ 参与路由 → /users/:username/notes（索引）
    │   │   └── new.tsx                      ✅ 参与路由 → /users/:username/notes/new
    │   ├── index.test.tsx                   ❌ 被忽略（.test.）
    │   └── index.tsx                        ✅ 参与路由 → /users/:username（索引）
    └── index.tsx                            ✅ 参与路由 → /users（索引）
```

---

## 七、核心约定速查（自洽版）

### 7.1 路由发现符号

| 符号 | 位置 | 来源 | 作用 | 真实示例 |
|------|------|------|------|----------|
| `$` | 文件名/目录名前缀 | 库内置 | 动态 URL 参数 | `$username/`, `$noteId.tsx` |
| `_` | 目录名前缀 | 库内置 | 路径组（不影响 URL） | `_auth/`, `_marketing/` |
| `_` | 文件名后缀 | 库内置 | 额外路径段 | `password_.create.tsx` |
| `_layout.tsx` | 文件名 | 库内置 | 布局组件 | `settings/profile/_layout.tsx` |
| `index.tsx` | 文件名 | 库内置 | 索引路由 | `users/index.tsx` |
| `[.]` | 文件名内 | 库内置 | 转义点号 | `robots[.]txt.ts` |
| `$` | 根级文件名 | 库内置 | 通配符路由 | `$.tsx` |

### 7.2 忽略规则符号

| 符号/模式 | 来源 | 可共置？ | 真实示例 |
|-----------|------|----------|----------|
| `+` 目录前缀 | 库内置 | ✅ 是 | `+logos/`, `+shared/`, `+types/` |
| `**/*.server.*` | `app/routes.ts:15` | ✅ 是 | `verify.server.ts`, `login.server.ts` |
| `**/*.client.*` | `app/routes.ts:16` | ✅ 是 | （无实例） |
| `**/*.test.*` | `app/routes.ts:8` | ✅ 是 | `callback.test.ts`, `index.test.tsx` |
| `**/*.css` | `app/routes.ts:7` | ❌ 否 | （无实例） |
| `.*` | `app/routes.ts:6` | ❌ 否 | （无实例） |
| `**/__*.*` | `app/routes.ts:9` | ❌ 否 | （无实例） |

### 7.3 特殊转义（不被忽略）

| 转义方式 | 配置说明 | 效果 | 真实状态 |
|----------|----------|------|----------|
| `[server]` | `app/routes.ts:14` | 生成路由 `/my-route/server` | 未使用 |
| `[client]` | `app/routes.ts:14` | 生成路由 `/my-route/client` | 未使用 |

**注意**：`[server]` / `[client]` 不是忽略规则，而是**避免被忽略**的转义机制。

---

## 八、关键澄清

### 8.1 `+` 前缀目录的来源

- **不在** `app/routes.ts` 的 `ignoredRouteFiles` 配置中
- 是 `react-router-auto-routes` 库的**内置约定**
- 文档依据：`docs/routing.md:210` - "react-router-auto-routes takes `+` prefix for colocated modules"

### 8.2 `.server.` 的双重身份

- 在 `ignoredRouteFiles` 中配置为忽略 → **不参与路由发现**
- 可被路由文件 import → **可共置**
- 真实证据：`verify.tsx:14` 引用 `./verify.server.ts`

### 8.3 `[server]` 转义的真实含义

- **不是**忽略规则
- **是**避免被 `**/*.server.*` 忽略的转义
- 当前仓库**未使用**，所有 `.server.` 文件都是真正的服务端模块
