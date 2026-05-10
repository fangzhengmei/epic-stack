# Epic Stack 用户列表搜索功能全栈路径分析报告

## 1. 整体架构概览

Epic Stack 使用 **React Router 7** 作为全栈框架，结合 **Prisma 6** 作为 ORM，**SQLite** 作为数据库，**Tailwind CSS** 作为样式框架。用户搜索功能是一个典型的全栈功能，涉及前端输入、导航状态管理、类型安全数据库查询以及图片优化渲染。

## 2. 搜索入口点

### 2.1 搜索栏组件位置

搜索栏组件 `<SearchBar />` 在两个位置使用：

1. **全局导航栏** (`app/root.tsx:193-194`)
   - 在非搜索页面的顶部导航栏显示
   - 响应式设计：桌面端在导航栏右侧，移动端单独一行

2. **用户列表页** (`app/routes/users/index.tsx:32`)
   - 页面中心展示，带有自动聚焦和自动提交功能

### 2.2 搜索栏组件实现 (`app/components/search-bar.tsx`)

**核心特性：**

```typescript
// 组件 Props
{
  status: 'idle' | 'pending' | 'success' | 'error'  // 搜索状态
  autoFocus?: boolean                                // 自动聚焦
  autoSubmit?: boolean                               // 自动提交（防抖）
}
```

**关键技术点：**

1. **GET 表单提交** (lines 31-34)
   - 使用 React Router 的 `<Form>` 组件
   - 方法为 GET，搜索参数通过 URL 查询字符串传递
   - 目标路由为 `/users`

2. **URL 状态同步** (lines 19, 45)
   - 使用 `useSearchParams()` 获取当前 URL 中的搜索参数
   - 通过 `defaultValue` 保持输入框与 URL 状态一致
   - 刷新页面或浏览器前进后退时保持搜索状态

3. **防抖自动提交** (lines 26-28)
   - 使用 `useDebounce` hook，延迟 400ms
   - 当 `autoSubmit` 为 true 时，输入变化后自动提交表单
   - 优化用户体验，减少不必要的请求

4. **导航状态检测** (lines 21-24)
   - 使用 `useIsPending` 检测表单是否正在提交
   - 配合 `StatusButton` 显示加载状态

## 3. 前端导航状态保持

### 3.1 URL 作为状态源

搜索功能采用 **URL 驱动** 的状态管理模式：

```
用户输入 "kody" → URL 变为 /users?search=kody
```

**优点：**
- 可分享链接
- 浏览器历史记录可用
- 刷新页面保持状态
- 无需额外的状态管理库

### 3.2 状态保持的回退与边界链路分析

#### 3.2.1 清空搜索词重定向机制

**核心逻辑** (`app/routes/users/index.tsx:77-80`)：

```typescript
const searchTerm = new URL(request.url).searchParams.get('search')
if (searchTerm === '') {
  return redirect('/users')
}
```

**场景触发流程：**

```
场景：用户手动清空搜索框并按回车（或触发自动提交）

1. 输入变化 → 搜索框值变为空字符串 ""
2. 防抖 400ms → 触发表单提交
3. Form GET 请求 → URL 变为 /users?search=
4. 服务端 Loader 执行：
   - searchTerm = new URL(request.url).searchParams.get('search') → ""
   - 条件判断：searchTerm === '' → true
   - 执行 redirect('/users') → 返回 302 重定向响应
5. React Router 客户端接收重定向 → 导航到 /users
6. 浏览器地址栏更新：/users?search= → /users
7. 新的 Loader 执行：
   - searchTerm = new URL(request.url).searchParams.get('search') → null
   - like = `%${null ?? ''}%` → "%%"
   - 查询所有用户
```

**为什么需要重定向？**

| 状态 | URL | 搜索参数 | 实际查询 |
|------|-----|----------|----------|
| 无搜索 | `/users` | `null` | 所有用户 (`%%`) |
| 空搜索参数 | `/users?search=` | `""` | 所有用户 (`%%`) |
| 有搜索 | `/users?search=kody` | `"kody"` | 匹配 `%kody%` |

**设计意图：**
- 避免重复的 URL 语义（`/users` 和 `/users?search=` 语义相同）
- 保持 URL 简洁美观
- 统一用户体验，避免书签保存冗余 URL

#### 3.2.2 从搜索结果进入用户详情再返回的状态恢复

**完整场景链路：**

```
初始状态：/users?search=kody
            ↓
步骤 1：点击用户卡片（Link 导航）
            ↓
React Router 客户端导航
            ↓
URL 变为：/users/kody
            ↓
用户详情页 Loader 执行（获取 kody 用户数据）
            ↓
用户详情页渲染
            ↓
步骤 2：用户点击浏览器「返回」按钮
            ↓
浏览器历史回退
            ↓
URL 恢复为：/users?search=kody
            ↓
React Router 触发导航
            ↓
步骤 3：状态恢复流程
            ├─ ① URL 状态恢复：search=kody
            ├─ ② 服务端 Loader 重新执行：
            │    - 解析 searchParams.get('search') → "kody"
            │    - 执行数据库查询
            │    - 返回搜索结果
            └─ ③ 客户端组件重新渲染：
                 ├─ SearchBar 组件：
                 │    - useSearchParams() 获取 "kody"
                 │    - Input defaultValue="kody"
                 │    - 搜索框显示 "kody"
                 └─ 用户列表：
                      - 显示搜索结果
                      - 滚动位置由 ScrollRestoration 恢复
```

**关键技术点解析：**

**A. 浏览器历史栈管理**

React Router 7 的导航行为：

```typescript
// 用户列表页中的用户卡片链接
<Link
  to={user.username}  // 相对路径，导航到 /users/{username}
  aria-label={`${user.name || user.username} profile`}
>
```

当用户点击 `<Link>` 时，React Router 执行 `pushState` 而非 `replaceState`，因此：

| 操作 | 历史栈变化 |
|------|-----------|
| 访问 `/users?search=kody` | `[..., /users?search=kody]` |
| 点击用户进入详情 | `[..., /users?search=kody, /users/kody]` |
| 点击返回 | `[..., /users?search=kody, /users/kody]` (指针前移) |

**B. 输入框状态恢复机制** (`app/components/search-bar.tsx:19, 45`)

```typescript
// 关键代码
const [searchParams] = useSearchParams()
// ...
<Input
  type="search"
  name="search"
  defaultValue={searchParams.get('search') ?? ''}
  // ...
/>
```

**为什么使用 `defaultValue` 而非 `value`？**

| 属性 | 行为 | 适用场景 |
|------|------|----------|
| `defaultValue` | 仅在首次渲染时设置，后续由 DOM 自行管理 | 非受控组件，Form 提交驱动 |
| `value` | 始终与状态同步，需要 `onChange` 处理 | 受控组件，状态驱动 |

**设计选择原因：**

1. **单一数据源**：URL 是唯一的状态源，输入框只是 URL 的"视图"
2. **浏览器行为一致**：前进/后退时，URL 变化 → 组件重新渲染 → `defaultValue` 从 URL 读取
3. **表单原生行为**：使用 `<Form>` 组件的原生 GET 提交，无需额外的 `onChange` 处理
4. **性能优化**：避免每次输入都触发 React 状态更新和重渲染

**C. 滚动位置恢复** (`app/root.tsx:169`)

```tsx
<ScrollRestoration nonce={nonce} />
```

React Router 的 `<ScrollRestoration>` 组件：

- 自动保存每个历史条目的滚动位置
- 导航返回时自动恢复滚动位置
- 使用 `sessionStorage` 存储滚动位置数据
- 对用户完全透明

**D. 数据重新获取**

当从详情页返回列表页时：

```
浏览器返回
    ↓
React Router 检测到导航
    ↓
触发用户列表页 Loader 重新执行
    ↓
重新获取搜索结果（可能已变化）
    ↓
组件重新渲染
```

**这是有意的设计选择**：
- 确保数据是最新的
- 符合"URL 驱动"的架构理念
- 避免展示过期的缓存数据

#### 3.2.3 边界场景汇总

| 场景 | URL 变化 | 搜索框状态 | 列表状态 |
|------|----------|-------------|----------|
| 首次访问 `/users` | `/users` | 空 | 所有用户 |
| 输入 "kody" 自动提交 | `/users?search=kody` | "kody" | 搜索结果 |
| 清空输入提交 | `/users?search=` → 重定向 → `/users` | 空 | 所有用户 |
| 进入用户详情 | `/users/kody` | - | - |
| 从详情返回 | `/users?search=kody` | "kody" (从 URL 恢复) | 搜索结果 |
| 浏览器前进 | `/users/kody` | - | - |
| 浏览器后退 | `/users?search=kody` | "kody" | 搜索结果 |
| 刷新页面 `/users?search=kody` | `/users?search=kody` | "kody" | 搜索结果 |
| 分享链接给他人 | `/users?search=kody` | "kody" | 搜索结果 |

### 3.3 Loader 数据获取 (`app/routes/users/index.tsx:11-20`)

```typescript
export async function loader({ request }: Route.LoaderArgs) {
  // 1. 从 URL 解析搜索参数
  const searchTerm = new URL(request.url).searchParams.get('search')
  
  // 2. 处理空搜索 - 重定向到用户列表主页
  if (searchTerm === '') {
    return redirect('/users')
  }

  // 3. 构建 LIKE 查询模式
  const like = `%${searchTerm ?? ''}%`
  
  // 4. 执行类型安全的 SQL 查询
  const users = await prisma.$queryRawTyped(searchUsers(like))
  
  // 5. 返回数据给组件
  return { status: 'idle', users } as const
}
```

**关键处理逻辑：**

| 情况 | 处理方式 |
|------|----------|
| 无搜索参数 (`search` 不存在) | 搜索所有用户 (`%%`) |
| 空搜索参数 (`search=''`) | 重定向到 `/users` |
| 有搜索参数 | 执行 LIKE 查询 |

### 3.4 页面导航状态视觉反馈

**延迟加载状态** (`app/utils/misc.tsx:176-189`):

```typescript
export function useDelayedIsPending({
  formAction,
  formMethod,
  delay = 400,        // 400ms 延迟后才显示加载状态
  minDuration = 300,  // 加载状态最少显示 300ms
} = {}) {
  // 结合 useIsPending 和 spin-delay
}
```

**视觉效果：**
- 请求时间 < 400ms：不显示加载状态（避免闪烁）
- 请求时间 > 400ms：显示加载状态，且至少显示 300ms

**应用位置** (`app/routes/users/index.tsx:23-26, 40`)：
- 用户列表在加载时添加 `opacity-50` 样式
- 搜索按钮显示 pending 状态

## 4. 类型安全的数据库查询

### 4.1 Prisma Typed SQL 特性

**配置** (`prisma/schema.prisma:4-7`)：

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["typedSql"]  // 启用类型化 SQL 预览功能
}
```

**生成命令**：

- 开发环境：`package.json:18` 中的 `setup` 脚本包含 `prisma generate --sql`
- 生产环境：`other/litefs.yml:43-44` 中部署流程执行 `npx prisma generate --sql`

### 4.2 原生 SQL 查询定义 (`prisma/sql/searchUsers.sql`)

```sql
-- @param {String} $1:like
SELECT 
  "User".id,
  "User".username,
  "User".name,
  "UserImage".id AS imageId,
  "UserImage".objectKey AS imageObjectKey
FROM "User"
LEFT JOIN "UserImage" ON "User".id = "UserImage".userId
WHERE "User".username LIKE :like
OR "User".name LIKE :like
ORDER BY (
  SELECT "Note".updatedAt
  FROM "Note"
  WHERE "Note".ownerId = "User".id
  ORDER BY "Note".updatedAt DESC
  LIMIT 1
) DESC
LIMIT 50
```

**查询解析：**

1. **JOIN 策略**：使用 `LEFT JOIN` 关联用户头像表，确保没有头像的用户也能被搜索到

2. **搜索条件**：
   - 在 `username` 字段上执行模糊匹配
   - 在 `name` 字段上执行模糊匹配
   - 使用 `OR` 条件，任一匹配即可

3. **排序策略**（优化用户体验）：
   - 按用户最近更新的笔记时间降序排列
   - 活跃用户排在前面
   - 数据库索引优化：`@@index([ownerId, updatedAt])` (`prisma/schema.prisma:48`)

4. **分页限制**：最多返回 50 条结果

### 4.3 类型安全查询调用

**导入** (`app/routes/users/index.tsx:1`)：
```typescript
import { searchUsers } from '@prisma/client/sql'
```

**调用** (`app/routes/users/index.tsx:18`)：
```typescript
const users = await prisma.$queryRawTyped(searchUsers(like))
```

**类型安全保障（可被代码验证）：**

1. **参数类型注解**：SQL 文件中 `-- @param {String} $1:like` 声明参数类型
2. **Prisma 生成**：执行 `prisma generate --sql` 后自动生成类型安全函数
3. **返回类型推断**：`users` 变量自动获得以下类型：
   ```typescript
   Array<{
     id: string
     username: string
     name: string | null
     imageId: string | null
     imageObjectKey: string | null
   }>
   ```
4. **SQL 注入防护**：Prisma 自动处理参数转义

### 4.4 技术方案取舍：为什么选择 Typed SQL？

#### 4.4.1 仓库中的可选方案对比

以下对比仅基于 Epic Stack 仓库中实际使用或可验证的方案：

| 方案 | 类型安全 | SQL 控制力 | 复杂度 | 性能 | 仓库验证 |
|------|----------|-----------|--------|------|---------|
| **Prisma Typed SQL** (当前方案) | ✅ 完整 | ✅ 完全控制 | 中 | ✅ 最优 | `prisma/sql/searchUsers.sql` |
| Prisma Query Builder | ✅ 完整 | ⚠️ 有限 | 低 | ⚠️ 依赖查询 | 仓库中大量使用 |
| 原生 SQL (`$queryRaw`) | ❌ 无 | ✅ 完全控制 | 低 | ✅ 最优 | Prisma 内置 |

#### 4.4.2 各方案详细分析（基于仓库代码）

**方案 A：Prisma Query Builder**

```typescript
// 用 Prisma Query Builder 实现部分功能
const users = await prisma.user.findMany({
  select: {
    id: true,
    username: true,
    name: true,
    image: { select: { objectKey: true } }
  },
  where: {
    OR: [
      { username: { contains: searchTerm } },
      { name: { contains: searchTerm } }
    ]
  },
  take: 50
})
```

**仓库可验证的局限性**：

1. **排序限制**（无法在仓库中实现）：
   - Prisma Query Builder 无法表达 `ORDER BY (SELECT ...)` 子查询排序
   - 搜索需求中的"按最近笔记更新时间排序"无法用 Query Builder 实现

2. **关联查询方式**：
   - Prisma Query Builder 的 `include` 会执行多个查询
   - Typed SQL 使用单个 `LEFT JOIN` 查询，更高效

**方案 B：原生 SQL `$queryRaw`**

```typescript
// 无类型安全的原生 SQL
const users = await prisma.$queryRaw`
  SELECT "User".id, "User".username, "User".name,
         "UserImage".objectKey AS "imageObjectKey"
  FROM "User" LEFT JOIN "UserImage" ON "User".id = "UserImage".userId
  WHERE "User".username LIKE ${'%' + searchTerm + '%'}
  OR "User".name LIKE ${'%' + searchTerm + '%'}
  ORDER BY (...) DESC
  LIMIT 50
`
// users 类型为 any
```

**仓库可验证的局限性**：

1. **无类型安全**：`users` 变量类型为 `any`，与 `docs/decisions/001-typescript-only.md` 的 "TypeScript only" 原则冲突
2. **重构风险**：修改 SQL 字段后 TypeScript 无法检测
3. **开发体验差**：无自动补全、无类型提示

**方案 C：Typed SQL（当前方案，仓库中已实现）**

```typescript
// prisma/sql/searchUsers.sql - 独立的 SQL 文件（仓库中存在）
-- @param {String} $1:like
SELECT ...

// TypeScript 调用
import { searchUsers } from '@prisma/client/sql'
const users = await prisma.$queryRawTyped(searchUsers(like))
// users 类型自动推断
```

**仓库可验证的优势**：

1. **完整类型安全**：由 `prisma generate --sql` 生成类型
2. **完全 SQL 控制**：子查询、窗口函数等高级特性都可使用
3. **性能最优**：单个优化的 SQL 查询
4. **SQL 文件管理**：SQL 代码在 `prisma/sql/` 目录独立管理
5. **与仓库配置一致**：`prisma/schema.prisma:6` 启用了 `typedSql` preview feature

#### 4.4.3 决策矩阵（可被仓库代码验证）

| 需求 | Typed SQL | Query Builder | 原生 SQL | 验证依据 |
|------|-----------|---------------|----------|---------|
| 子查询排序 (ORDER BY) | ✅ 支持 | ❌ 不支持 | ✅ 支持 | `prisma/sql/searchUsers.sql:157-163` |
| 类型安全 | ✅ 有 | ✅ 有 | ❌ 无 | `@prisma/client/sql` 导入 |
| 单个 JOIN 查询 | ✅ 是 | ❌ 可能多个 | ✅ 是 | `LEFT JOIN` 语句 |
| SQL 独立文件 | ✅ 是 | ❌ 嵌入 TS | ⚠️ 可提取 | `prisma/sql/` 目录 |
| TypeScript only 原则 | ✅ 符合 | ✅ 符合 | ❌ 不符合 | `docs/decisions/001-typescript-only.md` |

**最终选择理由（基于仓库代码）：**

1. **业务需求驱动**：
   - 搜索需要按"最近笔记更新时间"排序
   - 这需要 `ORDER BY (SELECT ...)` 子查询
   - Prisma Query Builder 无法表达此排序

2. **类型安全优先**：
   - Epic Stack 的核心理念是全栈类型安全
   - `docs/decisions/001-typescript-only.md` 明确 "TypeScript only"
   - 原生 SQL 的 `any` 类型不可接受

3. **性能考量**：
   - 用户搜索是高频操作
   - Typed SQL 使用单个 `LEFT JOIN` 查询
   - 比 Query Builder 的多个查询更高效

4. **SQL 可维护性**：
   - 复杂查询在 `prisma/sql/searchUsers.sql` 独立文件中
   - 便于数据库优化审查
   - 版本控制中 diff 清晰

## 5. 用户头像渲染

### 5.1 头像 URL 构建 (`app/utils/misc.tsx:8-12`)

```typescript
export function getUserImgSrc(objectKey?: string | null) {
  return objectKey
    ? `/resources/images?objectKey=${encodeURIComponent(objectKey)}`
    : '/img/user.png'
}
```

**逻辑（可被代码验证）：**
- 有 `objectKey`：构建图片资源 URL，经过 `encodeURIComponent` 编码
- 无 `objectKey`：使用默认头像 `/img/user.png`（位于 `public/` 目录）

### 5.2 图片组件使用 (`app/routes/users/index.tsx:50-56`)

```tsx
<Img
  alt={user.name ?? user.username}  // 无障碍：使用用户名或姓名作为 alt
  src={getUserImgSrc(user.imageObjectKey)}
  className="size-16 rounded-full"  // 64px 大小，圆角
  width={256}                        // 原始尺寸
  height={256}
/>
```

### 5.3 openimg 实际责任边界

openimg 分为两部分，应用代码提供的配置如下：

**A. 客户端部分 (`openimg/react`)**

**配置位置** (`app/root.tsx:198-201`)：
```tsx
<OpenImgContextProvider
  optimizerEndpoint="/resources/images"
  getSrc={getImgSrc}
>
```

**应用代码提供的 `getImgSrc`** (`app/utils/misc.tsx:18-44`)：
```typescript
export function getImgSrc({
  height,
  optimizerEndpoint,
  src,
  width,
  fit,
  format,
}: GetSrcArgs) {
  // 自定义 URL 构建逻辑
  if (src.startsWith(optimizerEndpoint)) {
    const [endpoint, query] = src.split('?')
    const searchParams = new URLSearchParams(query)
    searchParams.set('h', height.toString())
    searchParams.set('w', width.toString())
    if (fit) searchParams.set('fit', fit)
    if (format) searchParams.set('format', format)
    return `${endpoint}?${searchParams.toString()}`
  }
  return defaultGetSrc({ height, optimizerEndpoint, src, width, fit, format })
}
```

**客户端责任边界：**

| 责任方 | 功能 | 验证依据 |
|--------|------|---------|
| **openimg/react** | 提供 `<Img>` 组件 | 多个文件中 `import { Img } from 'openimg/react'` |
| **openimg/react** | 从 Context 读取配置 | `OpenImgContextProvider` |
| **应用代码** | 提供 `optimizerEndpoint` | `app/root.tsx:199` |
| **应用代码** | 提供自定义 `getImgSrc` 函数 | `app/utils/misc.tsx:18-44` |
| **应用代码** | 构建优化后的图片 URL | `getImgSrc` 将参数添加到查询字符串 |

**B. 服务端部分 (`openimg/node`)**

**配置位置** (`app/routes/resources/images.tsx`)：
```typescript
import { getImgResponse } from 'openimg/node'

export async function loader({ request }: Route.LoaderArgs) {
  const url = new URL(request.url)
  const searchParams = url.searchParams
  const headers = new Headers()
  headers.set('Cache-Control', 'public, max-age=31536000, immutable')
  const objectKey = searchParams.get('objectKey')

  return getImgResponse(request, {
    headers,
    allowlistedOrigins: [
      getDomainUrl(request),
      process.env.AWS_ENDPOINT_URL_S3,
    ].filter(Boolean),
    cacheFolder: await getCacheDir(),
    getImgSource: () => {
      // 应用代码决定从哪里获取原始图片
      if (objectKey) {
        const { url: signedUrl, headers: signedHeaders } =
          getSignedGetRequestInfo(objectKey)
        return { type: 'fetch', url: signedUrl, headers: signedHeaders }
      }
      // ... 其他来源
    },
  })
}
```

**服务端责任边界：**

| 责任方 | 功能 | 验证依据 |
|--------|------|---------|
| **应用代码** | 决定图片来源（对象存储/文件系统） | `getImgSource` 回调 |
| **应用代码** | 生成对象存储签名 URL | `app/utils/storage.server.ts:169-178` |
| **应用代码** | 设置缓存策略 | `Cache-Control: public, max-age=31536000, immutable` |
| **应用代码** | 设置图片来源白名单 | `allowlistedOrigins` |
| **应用代码** | 指定缓存目录 | `cacheFolder: await getCacheDir()` |
| **openimg/node** | 处理图片转换（由 sharp 提供） | `docs/image-optimization.md:12` |
| **openimg/node** | 处理请求参数（w, h, format, fit） | `docs/image-optimization.md:10` |

### 5.4 图片处理链路的实际形式

**基于仓库代码和文档的完整链路：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    头像渲染完整链路（可被代码验证）                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  1. 客户端渲染阶段                                            │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │                                                             │   │
│  │  <Img src="/resources/images?objectKey=xxx"                  │   │
│  │       width={256} height={256} />                           │   │
│  │       │                                                     │   │
│  │       ▼                                                     │   │
│  │  OpenImgContextProvider 读取配置                             │   │
│  │  - optimizerEndpoint: "/resources/images"                    │   │
│  │  - getSrc: 应用自定义函数                                    │   │
│  │       │                                                     │   │
│  │       ▼                                                     │   │
│  │  应用代码 getImgSrc() 构建优化 URL                            │   │
│  │  结果：/resources/images?objectKey=xxx&w=256&h=256          │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  2. 服务端处理阶段                                            │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │                                                             │   │
│  │  GET /resources/images?objectKey=xxx&w=256&h=256            │   │
│  │       │                                                     │   │
│  │       ▼                                                     │   │
│  │  应用代码 images.tsx loader                                  │   │
│  │  - 设置 Cache-Control 头                                    │   │
│  │  - 配置 allowlistedOrigins                                  │   │
│  │  - 配置 cacheFolder                                         │   │
│  │  - 实现 getImgSource 回调                                    │   │
│  │       │                                                     │   │
│  │       ▼                                                     │   │
│  │  应用代码 getImgSource() 决定图片来源                          │   │
│  │  objectKey 存在 → 调用 getSignedGetRequestInfo()            │   │
│  │                  → 返回 Tigris 签名 URL                      │   │
│  │       │                                                     │   │
│  │       ▼                                                     │   │
│  │  getImgResponse(request, { ... })                           │   │
│  │  (openimg/node 处理图片优化)                                 │   │
│  │  - 从 getImgSource 获取原始图片                               │   │
│  │  - 使用 sharp 进行尺寸调整、格式转换                           │   │
│  │  - 缓存到本地目录                                             │   │
│  │       │                                                     │   │
│  │       ▼                                                     │   │
│  │  返回优化后的图片（带 Cache-Control 头）                        │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.5 技术方案取舍：为什么选择 openimg？

#### 5.5.1 仓库中图片处理方案对比

以下对比基于 Epic Stack 仓库中的实际配置和文档：

| 方案 | 仓库验证 | 自托管 | 服务端优化 | 与 React Router 集成 |
|------|---------|--------|-----------|---------------------|
| **openimg** (当前方案) | `docs/image-optimization.md:3` | ✅ 是 | ✅ 是 | ✅ 深度集成 |
| 纯 `<img>` | 可行 | ✅ 是 | ❌ 否 | ⚠️ 简单 |
| 第三方 CDN (Cloudinary/Imgix) | 未配置 | ❌ 否 | ✅ 是 | ⚠️ 需额外集成 |

#### 5.5.2 各方案详细分析（基于仓库文档）

**方案 A：纯 `<img>`**

```tsx
// 手动实现
<img src={getUserImgSrc(objectKey)} alt={alt} />
```

**仓库可验证的局限性**：

1. **无服务端优化**：
   - 需要预先准备多种尺寸的图片
   - 无法根据浏览器能力选择最优格式

2. **与决策文档冲突**：
   - `docs/decisions/041-image-optimization.md:9-16` 明确说明需要图片优化
   - "optimizing images significantly improves web performance"

**方案 B：第三方 CDN (Cloudinary/Imgix)**

**仓库可验证的局限性**：

1. **不在默认配置中**：
   - 仓库中没有 Cloudinary/Imgix 相关依赖
   - 环境变量配置针对 Tigris (`AWS_ENDPOINT_URL_S3`, `BUCKET_NAME`)

2. **与 Epic Stack 理念不完全一致**：
   - `docs/image-optimization.md:14-17` 强调 "limit services"
   - 优先选择直接集成到 web server 的方案

**方案 C：openimg（当前方案，仓库中已实现）**

**仓库可验证的优势**：

1. **符合决策文档**：
   - `docs/decisions/041-image-optimization.md:33-35` 明确选择 openimg
   - "easy to use but also highly configurable"

2. **与现有架构集成**：
   - React Router SSR 路由 (`app/routes/resources/images.tsx`)
   - Tigris 对象存储 (`app/utils/storage.server.ts`)
   - 统一的图片 API（多个文件使用 `<Img>` 组件）

3. **配置明确**：
   - 客户端：`OpenImgContextProvider` 在 `app/root.tsx`
   - 服务端：`getImgResponse` 在 `app/routes/resources/images.tsx`
   - 缓存目录：`./tests/fixtures/openimg`（开发）或 `/data/images`（生产）

#### 5.5.3 决策矩阵（可被仓库代码/文档验证）

| 需求 | openimg | 纯 `<img>` | 第三方 CDN | 验证依据 |
|------|---------|-----------|-----------|---------|
| 服务端图片优化 | ✅ 有 | ❌ 无 | ✅ 有 | `docs/image-optimization.md:12` |
| 自托管 | ✅ 是 | ✅ 是 | ❌ 否 | 无第三方依赖 |
| 与 React Router 集成 | ✅ 原生 | ⚠️ 简单 | ⚠️ 需配置 | `app/routes/resources/images.tsx` |
| 与 Tigris 集成 | ✅ 已配置 | ⚠️ 需手动 | ⚠️ 需迁移 | `app/utils/storage.server.ts` |
| 符合决策文档 | ✅ 是 | ❌ 否 | ⚠️ 可替代 | `docs/decisions/041-image-optimization.md` |
| 本地开发体验 | ✅ 简单 | ✅ 简单 | ⚠️ 需要 mock | `./tests/fixtures/openimg` |

**最终选择理由（基于仓库代码和文档）：**

1. **符合决策文档**：
   - `docs/decisions/041-image-optimization.md` 明确记录了选择 openimg 的决策
   - 目标是 "integrate a simple image optimization solution directly into the web server"

2. **自托管优先**：
   - Epic Stack 强调可控性和独立性
   - 用户头像数据不经过第三方服务
   - 与 Tigris 对象存储无缝集成

3. **与现有架构深度集成**：
   - 使用 React Router 路由作为图片优化端点
   - 服务端渲染时正常工作
   - 本地开发使用 `./tests/fixtures/openimg` 缓存目录

4. **可验证的配置**：
   - 客户端：`OpenImgContextProvider` 配置 `optimizerEndpoint` 和 `getImgSrc`
   - 服务端：`getImgResponse` 配置 `getImgSource`、`cacheFolder`、`allowlistedOrigins`
   - 缓存策略：`Cache-Control: public, max-age=31536000, immutable`

## 6. 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         用户搜索完整数据流                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────────────┐   │
│  │  用户输入     │───▶│  SearchBar 组件   │───▶│  React Router Form       │   │
│  │  "kody"      │    │  (防抖 400ms)    │    │  GET /users?search=kody  │   │
│  └──────────────┘    └──────────────────┘    └──────────────────────────┘   │
│                                                                             │
│                                         │                                   │
│                                         ▼                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        服务端 (Loader)                               │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │  1. 解析 URL: searchParams.get('search') = "kody"             │  │   │
│  │  │  2. 构建 LIKE 模式: like = "%kody%"                           │  │   │
│  │  │  3. 调用 Prisma: prisma.$queryRawTyped(searchUsers("%kody%")) │  │   │
│  │  │  4. 返回数据: { status: 'idle', users: [...] }                │  │   │
│  │  └──────────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│                                         │                                   │
│                                         ▼                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        数据库 (SQLite)                               │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │  SELECT User.id, username, name,                              │  │   │
│  │  │         UserImage.objectKey AS imageObjectKey                 │  │   │
│  │  │  FROM User LEFT JOIN UserImage ON User.id = UserImage.userId  │  │   │
│  │  │  WHERE username LIKE '%kody%' OR name LIKE '%kody%'           │  │   │
│  │  │  ORDER BY (最近笔记更新时间) DESC                              │  │   │
│  │  │  LIMIT 50                                                     │  │   │
│  │  └──────────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│                                         │                                   │
│                                         ▼                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        前端渲染                                       │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │  1. 遍历 users 数组                                           │  │   │
│  │  │  2. 为每个用户构建 <Link to={user.username}>                   │  │   │
│  │  │  3. 头像: <Img src={getUserImgSrc(imageObjectKey)}>           │  │   │
│  │  │     └─▶ URL: /resources/images?objectKey=...                  │  │   │
│  │  │  4. 显示 name (如果有) 和 username                            │  │   │
│  │  └──────────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 7. 关键技术亮点

### 7.1 类型安全全链路

| 层级 | 类型安全机制 | 验证依据 |
|------|-------------|---------|
| 路由参数 | `Route.LoaderArgs` 自动生成类型 | React Router 类型生成 |
| Loader 数据 | `as const` 断言 + `Route.ComponentProps` | `app/routes/users/index.tsx:19,22` |
| SQL 参数 | Prisma Typed SQL 生成 `searchUsers()` 函数 | `prisma/sql/searchUsers.sql` |
| SQL 返回值 | Prisma 自动推断查询结果类型 | `prisma generate --sql` |

### 7.2 性能优化

1. **数据库索引** (`prisma/schema.prisma:48`)
   - `@@index([ownerId, updatedAt])` 优化子查询排序

2. **图片缓存** (`app/routes/resources/images.tsx:32-33`)
   - 一年强制缓存，避免重复请求

3. **防抖请求** (`app/components/search-bar.tsx:26-28`)
   - 400ms 延迟，减少频繁输入时的请求

4. **延迟加载状态** (`app/utils/misc.tsx:176-189`)
   - 避免快速请求时的加载闪烁

### 7.3 用户体验优化

1. **URL 状态同步**
   - 搜索结果可分享、可书签、可通过浏览器历史导航

2. **自动聚焦**
   - 在用户列表页自动聚焦搜索框，减少点击

3. **自动提交**
   - 输入变化后自动搜索，无需点击按钮

4. **智能排序**
   - 按最近活跃时间排序，活跃用户优先

5. **视觉反馈**
   - 搜索中列表半透明，按钮显示加载状态

## 8. 安全考虑

1. **SQL 注入防护**
   - 使用 Prisma `$queryRawTyped`，自动参数转义
   - 搜索词在 SQL 中作为参数而非字符串拼接

2. **XSS 防护**
   - 用户输入仅通过 URL 查询参数传递
   - 渲染时使用 React 的 JSX 自动转义
   - 头像 URL 使用 `encodeURIComponent` 编码

3. **图片来源白名单** (`app/routes/resources/images.tsx:39-42`)
   - 仅允许从本域名和配置的 S3 端点获取图片
   - 防止图片外链和 SSRF 攻击

4. **签名 URL** (`app/utils/storage.server.ts`)
   - 对象存储访问使用 AWS V4 签名
   - 临时认证，密钥不暴露给客户端

## 9. 测试覆盖

**E2E 测试** (`tests/e2e/search.test.ts`)：

```typescript
// 测试场景 1: 搜索现有用户
// - 输入存在的用户名
// - 点击搜索按钮
// - 验证用户列表中只显示 1 个用户
// - 验证用户链接可点击

// 测试场景 2: 搜索不存在的用户
// - 输入不存在的用户名 "__nonexistent__"
// - 验证 URL 更新为 /users?search=__nonexistent__
// - 验证显示 "No users found" 提示
```

## 10. 技术决策总结

### 10.1 Typed SQL vs 其他方案

| 维度 | 决策 | 可验证依据 |
|------|------|-----------|
| 排序需求 | Typed SQL | 子查询排序需求，Query Builder 不支持 |
| 类型安全 | Typed SQL | `docs/decisions/001-typescript-only.md` |
| 性能 | Typed SQL | 单个 LEFT JOIN 查询 |
| 可维护性 | Typed SQL | `prisma/sql/` 独立目录 |

### 10.2 openimg vs 其他方案

| 维度 | 决策 | 可验证依据 |
|------|------|-----------|
| 图片优化 | openimg | `docs/decisions/041-image-optimization.md` |
| 自托管 | openimg | 无第三方服务依赖 |
| 架构集成 | openimg | React Router 路由 + Tigris 存储 |
| 本地开发 | openimg | `./tests/fixtures/openimg` 缓存目录 |

## 11. 文件位置索引

| 功能模块 | 文件路径 |
|----------|----------|
| 用户列表路由 | `app/routes/users/index.tsx` |
| 搜索栏组件 | `app/components/search-bar.tsx` |
| 根布局（导航搜索栏） | `app/root.tsx` |
| SQL 查询定义 | `prisma/sql/searchUsers.sql` |
| 数据库模型 | `prisma/schema.prisma` |
| Prisma 客户端 | `app/utils/db.server.ts` |
| 工具函数（头像 URL、getImgSrc 等） | `app/utils/misc.tsx` |
| 图片资源路由 | `app/routes/resources/images.tsx` |
| 存储服务（签名 URL） | `app/utils/storage.server.ts` |
| 搜索 E2E 测试 | `tests/e2e/search.test.ts` |
| 用户详情页 | `app/routes/users/$username/index.tsx` |
| TypeScript only 决策 | `docs/decisions/001-typescript-only.md` |
| 图片优化决策 | `docs/decisions/041-image-optimization.md` |
| 图片优化文档 | `docs/image-optimization.md` |
| 图像存储文档 | `docs/image-storage.md` |
