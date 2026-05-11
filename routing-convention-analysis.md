# Epic Stack 路由约定分析报告

> 基于当前仓库真实配置与目录结构分析
> 所有结论均有配置或文件事实举证
> 版本：2026-05-11

---

## 一、配置事实

### 1.1 路由生成配置

**文件**：`app/routes.ts`（第 1-18 行）

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

### 1.2 React Router 全局配置

**文件**：`react-router.config.ts`（第 1-25 行）

```typescript
export default {
	ssr: true,
	routeDiscovery: { mode: 'initial' },
	future: {
		unstable_optimizeDeps: true,
	},
	// ...
} satisfies Config
```

### 1.3 目录结构统计

**路径**：`app/routes/`

| 类别 | 数量 | 示例 |
|------|------|------|
| 路由文件（.ts/.tsx） | 50+ | `login.tsx`, `$.tsx`, `_layout.tsx` |
| 被忽略文件 | 14 | 见下表 |
| `+` 前缀目录 | 2 | `+logos/`, `+shared/` |
| 路径组目录（`_` 前缀） | 3 | `_auth/`, `_marketing/`, `_seo/` |

---

## 二、路由发现流程

```
routeDiscovery: { mode: 'initial' }
        │
        ▼
    扫描 app/routes/ 目录
        │
        ├─► react-router-auto-routes 内置规则
        │       ├─► 解析文件命名：$、_、[.] 等
        │       └─► `+` 前缀目录：不参与路由发现
        │
        └─► ignoredRouteFiles 配置（app/routes.ts:5-17）
                ├─► `.*`
                ├─► `**/*.css`
                ├─► `**/*.test.*`
                ├─► `**/__*.*`
                ├─► `**/*.server.*`
                └─► `**/*.client.*`
        │
        ▼
    生成最终路由清单
```

---

## 三、会参与路由发现的约定

### 3.1 目录级约定

| 约定 | 规则说明 | 真实目录示例 | 对 URL 的影响 |
|------|----------|--------------|---------------|
| `_` 前缀目录 | 路径组（Route Group），组织路由但不影响 URL | `_auth/`, `_marketing/`, `_seo/` | 不影响（URL 不含 `_auth` 等前缀） |
| 普通目录 | 嵌套路由层次 | `admin/`, `resources/`, `settings/` | 创建嵌套路由（如 `/admin/cache`） |
| `$` 前缀目录 | 动态参数目录 | `$username/` | `/users/:username` |

**举证**：
- `_auth/` 下的 `login.tsx` 生成 `/login`（而非 `/_auth/login`）

### 3.2 文件级约定

| 约定 | 规则说明 | 真实文件示例 | 生成的 URL 路径 |
|------|----------|--------------|-----------------|
| `_layout.tsx` | 目录级布局组件 | `settings/profile/_layout.tsx` | 作为父路由，路径为目录路径 |
| `index.tsx` | 目录的索引路由 | `users/index.tsx`, `_marketing/index.tsx` | 对应目录路径 |
| `$` 前缀文件 | 动态参数文件 | `$noteId.tsx`, `$cacheKey.ts` | `/:noteId`, `/:cacheKey` |
| `_` 后缀 | 额外路径段 | `password_.create.tsx`, `$noteId_.edit.tsx` | `/password/create`, `/:noteId/edit` |
| `[.]` | 转义点号 | `robots[.]txt.ts`, `sitemap[.]xml.ts` | `/robots.txt`, `/sitemap.xml` |
| `$`（根级） | 通配符/兜底路由 | `$.tsx` | `/*` |
| 普通 `.ts/.tsx` | 不含特殊符号的路由文件 | `login.tsx`, `logout.tsx`, `about.tsx` | `/login`, `/logout`, `/about` |

---

## 四、被忽略的约定

### 4.1 分类定义

被忽略的文件**不会**被解析为路由。根据是否可被路由文件 import，分为：

- **可共置**：路由文件可以 import 这些文件中的代码
- **不可共置**：路由文件不会 import 这些文件（或无证据表明会 import）

### 4.2 完整对照表

| 约定 | 来源 | 是否可共置 | 真实文件示例 | 证据 |
|------|------|------------|--------------|------|
| `+` 前缀目录 | 库内置（react-router-auto-routes） | ✅ 可共置 | `+logos/`, `+shared/` | 路由文件 import 了这些目录中的文件 |
| `**/*.server.*` | ignoredRouteFiles（app/routes.ts:15） | ✅ 可共置 | `verify.server.ts`, `login.server.ts`, `note-editor.server.tsx` | 路由文件 import 了这些文件 |
| `**/*.client.*` | ignoredRouteFiles（app/routes.ts:16） | 理论可共置 | （当前仓库无实例） | 配置与 `.server.` 对称 |
| `**/*.test.*` | ignoredRouteFiles（app/routes.ts:8） | ✅ 可共置（但目的不同） | `callback.test.ts`, `index.test.tsx` | 测试文件共置在路由旁，不被路由文件 import |
| `**/*.css` | ignoredRouteFiles（app/routes.ts:7） | ❌ 不可共置 | （当前仓库 routes 目录无实例） | 无 import 证据 |
| `.*` | ignoredRouteFiles（app/routes.ts:6） | ❌ 不可共置 | （当前仓库 routes 目录无实例） | 无 import 证据 |
| `**/__*.*` | ignoredRouteFiles（app/routes.ts:9） | ❌ 不可共置 | （当前仓库 routes 目录无实例） | 无 import 证据 |

### 4.3 `+` 前缀目录 - 库内置约定

**关键事实**：`+` 前缀目录**不在** `app/routes.ts` 的 `ignoredRouteFiles` 配置中，但确实不参与路由发现。

**真实目录**：
```
app/routes/_marketing/+logos/     # 24 个文件
app/routes/users/$username/notes/+shared/  # 2 个文件
```

**可共置证据**：

**证据 1**：`_marketing/index.tsx` 引用 `+logos/`（第 8 行）
```typescript
import { logos } from './+logos/logos.ts'
```

**证据 2**：`users/$username/notes/new.tsx` 引用 `+shared/`（第 2、5 行）
```typescript
import { NoteEditor } from './+shared/note-editor.tsx'
export { action } from './+shared/note-editor.server.tsx'
```

**证据 3**：`users/$username/notes/$noteId_.edit.tsx` 引用 `+shared/`（第 5、8 行）
```typescript
import { NoteEditor } from './+shared/note-editor.tsx'
export { action } from './+shared/note-editor.server.tsx'
```

### 4.4 `.server.` 模块 - 配置约定

**配置位置**：`app/routes.ts:15` - `'**/*.server.*'`

**真实文件**（10 个）：
```
app/routes/_auth/login.server.ts
app/routes/_auth/verify.server.ts
app/routes/_auth/reset-password.server.ts
app/routes/_auth/onboarding/index.server.ts
app/routes/_auth/onboarding/$provider.server.ts
app/routes/_auth/webauthn/utils.server.ts
app/routes/admin/cache/sqlite.server.ts
app/routes/settings/profile/change-email.server.tsx
app/routes/users/$username/notes/+shared/note-editor.server.tsx
```

**可共置证据**：

**证据 1**：`_auth/verify.tsx` 引用 `verify.server.ts`（第 14 行）
```typescript
import { validateRequest } from './verify.server.ts'
```

**证据 2**：`_auth/login.tsx` 引用 `login.server.ts`（第 23 行）
```typescript
import { handleNewSession } from './login.server.ts'
```

**证据 3**：`settings/profile/change-email.tsx` 引用 `change-email.server.tsx`（第 21 行）
```typescript
import { EmailChangeEmail } from './change-email.server.tsx'
```

**证据 4**：`_auth/webauthn/registration.ts` 引用 `utils.server.ts`（第 14 行）
```typescript
} from './utils.server.ts'
```

### 4.5 `.test.` 文件 - 配置约定

**配置位置**：`app/routes.ts:8` - `'**/*.test.{js,jsx,ts,tsx}'`

**真实文件**（2 个）：
```
app/routes/_auth/auth.$provider/callback.test.ts
app/routes/users/$username/index.test.tsx
```

**说明**：测试文件共置在路由文件旁，但不被路由文件 import（被测试框架 import）。

---

## 五、转义机制：`[server]` / `[client]`

### 5.1 配置说明

**文件**：`app/routes.ts` 注释（第 10-14 行）

```typescript
// This is for server-side utilities you want to colocate
// next to your routes without making an additional
// directory. If you need a route that includes "server" or
// "client" in the filename, use the escape brackets like:
// my-route.[server].tsx
```

### 5.2 关键澄清

| 问题 | 答案 |
|------|------|
| `[server]` 是忽略规则吗？ | ❌ **不是** |
| `[server]` 的作用是什么？ | ✅ **避免被 `**/*.server.*` 规则忽略** |
| `my-route.[server].tsx` 会生成路由吗？ | ✅ **会**，生成 `/my-route/server` |
| 当前仓库使用了吗？ | ❌ **没有使用** |

### 5.3 逻辑关系

```
文件名包含 "server"
        │
        ▼
    两种情况：
        │
        ├─► xxx.server.tsx   → 匹配 `**/*.server.*`   → ❌ 被忽略
        │
        └─► xxx.[server].tsx → 不匹配 `**/*.server.*` → ✅ 参与路由发现
                                                 （生成 /xxx/server）
```

### 5.4 当前仓库状态

当前仓库所有包含 `server` 的文件均为 `.server.` 格式（服务端专用模块）：
- 10 个 `.server.` 文件
- 0 个 `[server]` 转义文件

---

## 六、真实目录结构标注

```
app/routes/
├── $.tsx                                    ✅ 参与路由 → /*
├── me.tsx                                   ✅ 参与路由 → /me
│
├── _auth/                                   ✅ 路径组（不影响 URL）
│   ├── auth.$provider/                      ✅ 动态参数目录
│   │   ├── callback.test.ts                 ❌ 被忽略（.test.）
│   │   ├── callback.ts                      ✅ 参与路由 → /auth/:provider/callback
│   │   └── index.ts                         ✅ 参与路由 → /auth/:provider
│   ├── onboarding/                          ✅ 普通目录
│   │   ├── $provider.server.ts              ❌ 被忽略（.server.）
│   │   ├── $provider.tsx                    ✅ 参与路由 → /onboarding/:provider
│   │   ├── index.server.ts                  ❌ 被忽略（.server.）
│   │   └── index.tsx                        ✅ 参与路由 → /onboarding
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
│   │   ├── remix.svg
│   │   ├── tailwind.svg
│   │   └── ... (21 个资源文件)
│   ├── about.tsx                            ✅ 参与路由 → /about
│   ├── index.tsx                            ✅ 参与路由 → /
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
│       ├── index.tsx                        ✅ 参与路由 → /admin/cache
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
│       │   ├── index.tsx                    ✅ 参与路由 → /settings/profile/two-factor
│       │   └── verify.tsx                   ✅ 参与路由 → /settings/profile/two-factor/verify
│       ├── _layout.tsx                      ✅ 参与路由（布局）→ /settings/profile
│       ├── change-email.server.tsx          ❌ 被忽略（.server.）
│       ├── change-email.tsx                 ✅ 参与路由 → /settings/profile/change-email
│       ├── connections.tsx                  ✅ 参与路由 → /settings/profile/connections
│       ├── index.tsx                        ✅ 参与路由 → /settings/profile
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
    │   │   ├── index.tsx                    ✅ 参与路由 → /users/:username/notes
    │   │   └── new.tsx                      ✅ 参与路由 → /users/:username/notes/new
    │   ├── index.test.tsx                   ❌ 被忽略（.test.）
    │   └── index.tsx                        ✅ 参与路由 → /users/:username
    └── index.tsx                            ✅ 参与路由 → /users
```

---

## 七、速查表（自洽版）

### 7.1 会参与路由发现

| 约定 | 位置 | 来源 | 真实示例 | 生成路径 |
|------|------|------|----------|----------|
| `$` | 文件名/目录名前缀 | 库内置 | `$username/`, `$noteId.tsx` | `:username`, `:noteId` |
| `_` | 目录名前缀 | 库内置 | `_auth/`, `_marketing/` | 不影响 URL |
| `_` | 文件名后缀 | 库内置 | `password_.create.tsx` | `/password/create` |
| `_layout.tsx` | 文件名 | 库内置 | `settings/profile/_layout.tsx` | 布局组件 |
| `index.tsx` | 文件名 | 库内置 | `users/index.tsx` | 索引路由 |
| `[.]` | 文件名内 | 库内置 | `robots[.]txt.ts` | `/robots.txt` |
| `$` | 根级文件名 | 库内置 | `$.tsx` | `/*` |

### 7.2 被忽略

| 约定 | 来源 | 可共置？ | 真实示例 | 证据 |
|------|------|----------|----------|------|
| `+` 目录前缀 | 库内置 | ✅ 是 | `+logos/`, `+shared/` | 路由文件 import 了这些目录 |
| `**/*.server.*` | app/routes.ts:15 | ✅ 是 | `verify.server.ts` | 路由文件 import 了这些文件 |
| `**/*.client.*` | app/routes.ts:16 | 理论是 | （无实例） | 配置与 `.server.` 对称 |
| `**/*.test.*` | app/routes.ts:8 | ✅ 是（测试共置） | `callback.test.ts` | 测试文件与路由共置 |
| `**/*.css` | app/routes.ts:7 | ❌ 否 | （无实例） | 无 import 证据 |
| `.*` | app/routes.ts:6 | ❌ 否 | （无实例） | 无 import 证据 |
| `**/__*.*` | app/routes.ts:9 | ❌ 否 | （无实例） | 无 import 证据 |

### 7.3 转义机制（不被忽略）

| 转义方式 | 配置说明 | 效果 | 真实状态 |
|----------|----------|------|----------|
| `[server]` | app/routes.ts:14 | 生成路由 `/my-route/server` | 未使用 |
| `[client]` | app/routes.ts:14 | 生成路由 `/my-route/client` | 未使用 |

---

## 八、核心结论

### 8.1 被忽略规则的来源

| 来源 | 规则 | 数量（当前仓库） |
|------|------|------------------|
| **库内置** | `+` 前缀目录 | 2 个目录，26 个文件 |
| **配置** | `**/*.server.*` | 10 个文件 |
| **配置** | `**/*.test.*` | 2 个文件 |
| **配置** | `**/*.css` | 0 个文件 |
| **配置** | `.*` | 0 个文件 |
| **配置** | `**/__*.*` | 0 个文件 |
| **配置** | `**/*.client.*` | 0 个文件 |

### 8.2 可共置的被忽略文件

当前仓库有真实证据支持可共置的：
1. `+` 前缀目录（4 处 import 证据）
2. `**/*.server.*`（多处 import 证据）

无证据但配置支持的：
3. `**/*.client.*`

共置但目的不同的：
4. `**/*.test.*`（被测试框架 import，而非路由文件）

### 8.3 转义与忽略的关系

```
被忽略的规则：
├── 库内置：`+` 前缀目录
└── 配置：`**/*.server.*`, `**/*.client.*`, `**/*.test.*`, `**/*.css`, `.*`, `**/__*.*`

转义机制（[server]/[client]）：
└── 目的：避免文件名中的 "server"/"client" 被 `**/*.server.*`/`**/*.client.*` 规则忽略
└── 效果：转义后的文件会参与路由发现
└── 当前状态：未使用
```
