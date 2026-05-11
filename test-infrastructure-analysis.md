# Epic Stack 测试基础设施分析报告

## 1. 核心架构概览

Epic Stack 的测试体系包含**两套独立的测试框架**，各自有独立的数据库隔离机制和 MSW 实例：

| 框架 | 用途 | 进程 | 数据库 | MSW 加载方式 |
|------|------|------|--------|--------------|
| **Vitest** | 单元测试、集成测试 | Vitest 进程 | 测试池隔离 (`data.${poolId}.db`) | 通过 `setupFiles` 导入 |
| **Playwright** | E2E 测试 | Playwright + 应用服务器 (Node.js) | 默认数据库 (`.env` 配置) | 通过 `MOCKS=true` 环境变量 |

**关键理解**：Vitest 和 Playwright 是**完全独立**的两个系统，它们不共享进程、不共享数据库、不共享 MSW 实例。

---

## 2. 数据库隔离机制详解

### 2.1 Vitest 数据库隔离

Vitest 的数据库隔离机制是**最完善**的，采用了三层隔离策略。

#### 2.1.1 配置位置

```
vite.config.ts:
  test: {
    globalSetup: ['./tests/setup/global-setup.ts'],  // 全局设置
    setupFiles: ['./tests/setup/setup-test-env.ts'],  // 每个测试池
  }
```

#### 2.1.2 全局设置 (`tests/setup/global-setup.ts`)

```typescript
export const BASE_DATABASE_PATH = path.join(
  process.cwd(),
  `./tests/prisma/base.db`,  // 基础数据库模板
)

export async function setup() {
  // 1. 检查 base.db 是否已存在
  const databaseExists = await fsExtra.pathExists(BASE_DATABASE_PATH)

  if (databaseExists) {
    // 2. 比较 schema 和数据库的修改时间
    const databaseLastModifiedAt = (await fsExtra.stat(BASE_DATABASE_PATH)).mtime
    const prismaSchemaLastModifiedAt = (
      await fsExtra.stat('./prisma/schema.prisma')
    ).mtime

    // 3. 如果 schema 没有更新，直接复用现有数据库
    if (prismaSchemaLastModifiedAt < databaseLastModifiedAt) {
      return
    }
  }

  // 4. 如果 schema 更新或数据库不存在，重新创建
  await execaCommand(
    'npx prisma migrate reset --force --skip-seed --skip-generate',
    {
      env: {
        ...process.env,
        DATABASE_URL: `file:${BASE_DATABASE_PATH}`,  // 临时设置为 base.db
      },
    },
  )
}
```

**职责**：创建或验证 `base.db`（基础数据库模板）

#### 2.1.3 测试池隔离 (`tests/setup/db-setup.ts`)

```typescript
import { afterAll, beforeEach } from 'vitest'

// 每个测试池有独立的 ID
const poolId = process.env.VITEST_POOL_ID || '0'
const databaseFile = `./tests/prisma/data.${poolId}.db`
const databasePath = path.join(process.cwd(), databaseFile)

// 关键：在导入 Prisma 之前设置 DATABASE_URL
process.env.DATABASE_URL = `file:${databasePath}`

// 同样处理缓存数据库
const cacheDatabasePath = process.env.CACHE_DATABASE_PATH
if (cacheDatabasePath && cacheDatabasePath !== ':memory:') {
  const parsed = path.parse(cacheDatabasePath)
  const cacheFileName = parsed.ext
    ? `${parsed.name}.${poolId}${parsed.ext}`
    : `${parsed.name}.${poolId}`
  process.env.CACHE_DATABASE_PATH = path.join(parsed.dir || '.', cacheFileName)
}

// 每个测试用例前：复制基础数据库
beforeEach(async () => {
  await fsExtra.copyFile(BASE_DATABASE_PATH, databasePath)
})

// 测试池结束后：断开连接并删除数据库
afterAll(async () => {
  // 动态导入确保 process.env.DATABASE_URL 已设置
  const { prisma } = await import('#app/utils/db.server.ts')
  await prisma.$disconnect()
  await fsExtra.remove(databasePath)
})
```

**关键技术点**：
1. **测试池隔离**：每个 Vitest 测试池有独立的数据库文件 `data.${poolId}.db`
2. **环境变量优先**：必须在导入 `db.server.ts` 之前设置 `process.env.DATABASE_URL`
3. **用例级重置**：每个 `beforeEach` 都从 `base.db` 复制，确保测试用例间完全隔离
4. **动态导入**：`afterAll` 中使用动态导入，确保此时 `DATABASE_URL` 已设置

#### 2.1.4 Vitest 数据库工作流

```
Vitest 启动
    ↓
global-setup.ts (执行一次)
    ├─ 检查 base.db 是否存在
    ├─ 比较 schema 修改时间
    └─ 如有需要，执行 prisma migrate reset 创建 base.db
    ↓
每个测试池 (独立 Node.js 进程)
    ↓
db-setup.ts (每个测试池执行)
    ├─ 读取 VITEST_POOL_ID
    ├─ 设置 DATABASE_URL = file:./tests/prisma/data.${poolId}.db
    └─ 设置 CACHE_DATABASE_PATH (带 poolId)
    ↓
每个测试用例
    ↓
beforeEach
    └─ 复制 base.db → data.${poolId}.db
    ↓
测试运行 (操作 data.${poolId}.db)
    ↓
afterEach (可选清理)
    ↓
测试池结束
    ↓
afterAll
    ├─ 断开 Prisma 连接
    └─ 删除 data.${poolId}.db
```

---

### 2.2 Playwright 数据库隔离

Playwright 的数据库隔离机制**与 Vitest 完全不同**，它不使用 `db-setup.ts`。

#### 2.2.2 Playwright 配置 (`playwright.config.ts`)

```typescript
export default defineConfig({
  testDir: './tests/e2e',
  
  webServer: {
    // CI 模式：使用构建产物 + MOCKS=true
    // 开发模式：使用 dev 服务器
    command: process.env.CI ? 'npm run start:mocks' : 'npm run dev',
    port: Number(PORT),
    timeout: 60 * 1000,
    reuseExistingServer: true,
    
    // 只设置 NODE_ENV=test，不设置 DATABASE_URL
    env: {
      PORT,
      NODE_ENV: 'test',
    },
  },
})
```

**关键点**：
- Playwright **不设置** `DATABASE_URL`
- 它启动一个独立的应用服务器进程
- 应用服务器使用 `.env` 中配置的默认数据库

#### 2.2.3 应用服务器启动流程

```
package.json:
  "dev": "cross-env NODE_ENV=development MOCKS=true node index.ts"
  "start:mocks": "cross-env NODE_ENV=production MOCKS=true node index.ts"
```

```typescript
// index.ts
import 'dotenv/config'  // 加载 .env 中的 DATABASE_URL

if (process.env.MOCKS === 'true') {
  await import('./tests/mocks/index.ts')  // 加载 MSW
}

await import('./server/index.ts')  // 启动 Express 服务器
```

```typescript
// server/index.ts
// 直接使用 .env 中的 DATABASE_URL，不做任何修改
```

#### 2.2.4 Playwright 测试中的数据库操作

Playwright 测试通过**两种方式**与数据库交互：

**方式一：夹具直接操作数据库** (`tests/playwright-utils.ts`)

```typescript
import { prisma } from '#app/utils/db.server.ts'  // 直接导入 Prisma

export const test = base.extend<{
  insertNewUser(options?: GetOrInsertUserOptions): Promise<User>
  login(options?: GetOrInsertUserOptions): Promise<User>
  prepareGitHubUser(): Promise<GitHubUser>
}>({
  insertNewUser: async ({}, use) => {
    let userId: string | undefined = undefined
    await use(async (options) => {
      const user = await getOrInsertUser(options)  // 直接操作数据库
      userId = user.id
      return user
    })
    // 测试结束后清理
    await prisma.user.delete({ where: { id: userId } }).catch(() => {})
  },

  login: async ({ page }, use) => {
    let userId: string | undefined = undefined
    await use(async (options) => {
      const user = await getOrInsertUser(options)
      userId = user.id
      
      // 创建会话
      const session = await prisma.session.create({
        data: {
          expirationDate: getSessionExpirationDate(),
          userId: user.id,
        },
      })

      // 注入 Cookie 到浏览器
      const authSession = await authSessionStorage.getSession()
      authSession.set(sessionKey, session.id)
      const cookieConfig = setCookieParser.parseString(
        await authSessionStorage.commitSession(authSession),
      )
      await page.context().addCookies([{
        ...cookieConfig,
        domain: 'localhost',
        expires: cookieConfig.expires?.getTime(),
        sameSite: cookieConfig.sameSite as 'Strict' | 'Lax' | 'None',
      }])
      return user
    })
    await prisma.user.deleteMany({ where: { id: userId } })
  },
})
```

**方式二：测试文件直接操作数据库**

```typescript
// tests/e2e/notes.test.ts
import { prisma } from '#app/utils/db.server.ts'

test('Users can edit notes', async ({ page, navigate, login }) => {
  const user = await login()

  // 直接在测试中创建数据
  const note = await prisma.note.create({
    select: { id: true },
    data: { ...createNote(), ownerId: user.id },
  })
  
  await navigate('/users/:username/notes/:noteId', {
    username: user.username,
    noteId: note.id,
  })
  // ...
})
```

#### 2.2.5 Playwright 数据库工作流

```
Playwright 启动
    ↓
playwright.config.ts
    └─ 启动 webServer (应用服务器)
        ↓
应用服务器启动 (独立 Node.js 进程)
    ├─ 加载 .env 中的 DATABASE_URL (默认数据库)
    ├─ 如果 MOCKS=true，加载 MSW
    └─ 启动 Express + React Router
    ↓
每个 Playwright 测试
    ↓
夹具执行 (login, insertNewUser 等)
    ├─ 导入 prisma (连接默认数据库)
    ├─ 操作数据库 (创建用户、创建笔记等)
    └─ 注入认证 Cookie 到浏览器
    ↓
Playwright 浏览器操作
    └─ 通过 HTTP 请求与应用服务器交互
        └─ 应用服务器操作默认数据库
    ↓
测试断言
    ├─ 检查页面内容
    └─ (可选) 直接查询数据库验证
    ↓
夹具清理
    └─ 删除测试创建的数据 (用户、笔记等)
```

---

### 2.3 数据库隔离对比

| 特性 | Vitest | Playwright |
|------|--------|------------|
| **数据库文件** | `tests/prisma/data.${poolId}.db` | `.env` 中配置的默认数据库 |
| **隔离级别** | 测试池级 + 用例级 (每次复制 base.db) | 测试级 (夹具清理) |
| **隔离机制** | 文件复制 + 环境变量覆盖 | 夹具的 use/cleanup 生命周期 |
| **并行支持** | 完全支持 (每个 pool 独立数据库) | 需要谨慎 (共享同一数据库) |
| **数据重置** | `beforeEach` 自动复制 base.db | 依赖夹具手动清理 |
| **配置位置** | `vite.config.ts` + `tests/setup/*` | `playwright.config.ts` + 夹具 |

---

## 3. MSW (Mock Service Worker) 生效范围

### 3.1 MSW 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                        MSW 实例分布                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────┐      ┌──────────────────────────────┐   │
│  │   Vitest 进程      │      │    Playwright 测试进程        │   │
│  │                   │      │                              │   │
│  │  MSW Server (A)   │      │  ┌────────────────────────┐  │   │
│  │  (通过 setupFiles) │      │  │  应用服务器进程           │  │   │
│  │                   │      │  │                        │  │   │
│  │  拦截:            │      │  │  MSW Server (B)         │  │   │
│  │  - 测试中的 HTTP   │      │  │  (通过 MOCKS=true)      │  │   │
│  │    请求            │      │  │                        │  │   │
│  └───────────────────┘      │  │  拦截:                  │  │   │
│                             │  │  - 应用服务器发出的      │  │   │
│                             │  │    HTTP 请求             │  │   │
│                             │  └────────────────────────┘  │   │
│                             └──────────────────────────────┘   │
│                                                                 │
│  MSW Server (A) 和 (B) 是完全独立的实例！                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Vitest 中的 MSW

#### 3.2.1 加载方式

```typescript
// vite.config.ts
test: {
  setupFiles: ['./tests/setup/setup-test-env.ts'],
}
```

```typescript
// tests/setup/setup-test-env.ts
import './db-setup.ts'  // 先设置数据库
import '#app/utils/env.server.ts'

import { server } from '#tests/mocks/index.ts'  // 导入 MSW Server

afterEach(() => server.resetHandlers())  // 每个测试后重置 handlers
```

```typescript
// tests/mocks/index.ts
import { setupServer } from 'msw/node'
import { handlers as githubHandlers } from './github.ts'
import { handlers as resendHandlers } from './resend.ts'
// ... 其他 handlers

export const server = setupServer(
  ...resendHandlers,
  ...githubHandlers,
  ...tigrisHandlers,
  ...pwnedPasswordApiHandlers,
)

server.listen({
  onUnhandledRequest(request, print) {
    if (request.url.includes('.sentry.io')) return
    if (request.url.includes('__rrdt')) return
    print.warning()
  },
})
```

#### 3.2.2 生效范围

**Vitest MSW 拦截**：
- Vitest 进程中发出的所有 HTTP 请求
- 包括：
  - 测试代码直接发起的请求
  - 被测试的函数/组件发起的请求
  - loader/action 测试中的请求

**Vitest MSW 不拦截**：
- Playwright 浏览器中的请求
- 应用服务器中的请求（除非在同一进程）

### 3.3 Playwright 中的 MSW

#### 3.3.1 加载方式

```typescript
// index.ts (应用入口)
import 'dotenv/config'

// 关键：只有 MOCKS=true 时才加载 MSW
if (process.env.MOCKS === 'true') {
  await import('./tests/mocks/index.ts')
}

await import('./server/index.ts')
```

```json
// package.json
{
  "dev": "cross-env NODE_ENV=development MOCKS=true node index.ts",
  "start:mocks": "cross-env NODE_ENV=production MOCKS=true node index.ts"
}
```

```typescript
// playwright.config.ts
webServer: {
  // CI: start:mocks (MOCKS=true)
  // 开发: dev (MOCKS=true)
  command: process.env.CI ? 'npm run start:mocks' : 'npm run dev',
  env: {
    PORT,
    NODE_ENV: 'test',
    // 注意：没有 MOCKS 变量，因为它在 npm script 中设置
  },
}
```

#### 3.3.2 生效范围

**Playwright MSW 拦截**：
- 应用服务器进程中发出的所有 HTTP 请求
- 包括：
  - 邮件服务 (Resend)
  - OAuth 服务 (GitHub)
  - 对象存储 (Tigris)
  - 密码泄露检查 (HaveIBeenPwned)

**Playwright MSW 不拦截**：
- Playwright 测试进程中的请求
- Vitest 进程中的请求
- 浏览器直接发起的请求（除非使用 MSW 浏览器版）

### 3.4 MSW 关键实现

#### 3.4.1 GitHub OAuth Mock

```typescript
// tests/mocks/github.ts

// Mock 用户数据存储（按测试池隔离）
const githubUserFixturePath = path.join(
  here(
    '..',
    'fixtures',
    'github',
    `users.${process.env.VITEST_POOL_ID || 0}.local.json`,
  ),
)

// 条件直通机制
const passthroughGitHub =
  !process.env.GITHUB_CLIENT_ID?.startsWith('MOCK_') &&
  process.env.NODE_ENV !== 'test'

export const handlers: Array<HttpHandler> = [
  // 1. OAuth 令牌交换
  http.post(
    'https://github.com/login/oauth/access_token',
    async ({ request }) => {
      if (passthroughGitHub) return passthrough()
      
      const params = new URLSearchParams(await request.text())
      const code = params.get('code')
      const githubUsers = await getGitHubUsers()
      let user = githubUsers.find((u) => u.code === code)
      if (!user) {
        user = await insertGitHubUser(code)
      }

      return json({
        access_token: user.accessToken,
        token_type: '__MOCK_TOKEN_TYPE__',
      })
    },
  ),

  // 2. 获取用户资料
  http.get('https://api.github.com/user', async ({ request }) => {
    if (passthroughGitHub) return passthrough()
    const user = await getUser(request)
    if (user instanceof Response) return user
    return json(user.profile)
  }),

  // 3. Mock 头像图片
  http.get('https://github.com/ghost.png', async () => {
    if (passthroughGitHub) return passthrough()
    const buffer = await fsExtra.readFile('./tests/fixtures/github/ghost.jpg')
    return new Response(buffer, {
      headers: { 'content-type': 'image/jpg' },
    })
  }),
]
```

#### 3.4.2 Resend 邮件 Mock

```typescript
// tests/mocks/resend.ts
export const handlers: Array<HttpHandler> = [
  http.post(`https://api.resend.com/emails`, async ({ request }) => {
    requireHeader(request.headers, 'Authorization')
    const body = await request.json()
    console.info('🔶 mocked email contents:', body)

    // 将邮件写入文件系统，供测试读取
    const email = await writeEmail(body)

    return json({
      id: faker.string.uuid(),
      from: email.from,
      to: email.to,
      created_at: new Date().toISOString(),
    })
  }),
]
```

```typescript
// tests/mocks/utils.ts
export async function writeEmail(rawEmail: unknown) {
  const email = EmailSchema.parse(rawEmail)
  // 保存到 tests/fixtures/email/${to}.json
  await createFixture('email', email.to, email)
  return email
}

export async function readEmail(recipient: string) {
  try {
    const email = await readFixture('email', recipient)
    return EmailSchema.parse(email)
  } catch (error) {
    return null
  }
}
```

### 3.5 MSW 生效范围总结

| 场景 | MSW 实例 | 加载条件 | 拦截范围 |
|------|----------|----------|----------|
| **Vitest 测试** | MSW Server (A) | `setupFiles` 自动加载 | Vitest 进程内的 HTTP 请求 |
| **Playwright E2E** | MSW Server (B) | `MOCKS=true` 环境变量 | 应用服务器进程内的 HTTP 请求 |
| **开发环境** | MSW Server (B) | `npm run dev` (MOCKS=true) | 应用服务器进程内的 HTTP 请求 |
| **生产环境** | 无 | `npm run start` (无 MOCKS) | 不拦截，使用真实服务 |

---

## 4. Playwright 夹具体系

### 4.1 夹具架构

Playwright 夹具在 `tests/playwright-utils.ts` 中定义，采用**生命周期钩子**模式。

```typescript
// tests/playwright-utils.ts
export const test = base.extend<{
  navigate: <Path extends AppPages>(...) => Promise<Response | null>
  insertNewUser(options?: GetOrInsertUserOptions): Promise<User>
  login(options?: GetOrInsertUserOptions): Promise<User>
  prepareGitHubUser(): Promise<GitHubUser>
}>({
  // 夹具定义...
})
```

### 4.2 核心夹具详解

#### 4.2.1 `login` 夹具（最常用）

```typescript
login: async ({ page }, use) => {
  let userId: string | undefined = undefined
  
  // setup 阶段：在测试运行前执行
  await use(async (options) => {
    // 1. 创建或获取用户
    const user = await getOrInsertUser(options)
    userId = user.id
    
    // 2. 在数据库中创建会话
    const session = await prisma.session.create({
      data: {
        expirationDate: getSessionExpirationDate(),
        userId: user.id,
      },
      select: { id: true },
    })

    // 3. 生成认证 Cookie
    const authSession = await authSessionStorage.getSession()
    authSession.set(sessionKey, session.id)
    const cookieConfig = setCookieParser.parseString(
      await authSessionStorage.commitSession(authSession),
    )
    
    // 4. 将 Cookie 注入 Playwright 浏览器上下文
    await page.context().addCookies([{
      ...cookieConfig,
      domain: 'localhost',
      expires: cookieConfig.expires?.getTime(),
      sameSite: cookieConfig.sameSite as 'Strict' | 'Lax' | 'None',
    }])
    
    // 5. 将 user 对象传递给测试
    return user
  })
  
  // cleanup 阶段：在测试运行后执行
  await prisma.user.deleteMany({ where: { id: userId } })
},
```

**工作流程**：
```
测试开始
    ↓
login 夹具 setup
    ├─ 创建用户 (prisma.user.create)
    ├─ 创建会话 (prisma.session.create)
    └─ 注入 Cookie 到浏览器
    ↓
测试运行
    └─ 浏览器已登录，可访问受保护路由
    ↓
login 夹具 cleanup
    └─ 删除用户 (prisma.user.deleteMany)
```

#### 4.2.2 `insertNewUser` 夹具

```typescript
insertNewUser: async ({}, use) => {
  let userId: string | undefined = undefined
  
  await use(async (options) => {
    const user = await getOrInsertUser(options)
    userId = user.id
    return user
  })
  
  await prisma.user.delete({ where: { id: userId } }).catch(() => {})
},
```

**与 `login` 的区别**：
- `insertNewUser`：只创建用户，不注入 Cookie（未登录状态）
- `login`：创建用户 + 创建会话 + 注入 Cookie（已登录状态）

#### 4.2.3 `prepareGitHubUser` 夹具

```typescript
prepareGitHubUser: async ({ page }, use, testInfo) => {
  // 1. 拦截 GitHub OAuth 请求，添加测试 ID 头
  await page.route(/\/auth\/github(?!\/callback)/, async (route, request) => {
    const headers = {
      ...request.headers(),
      [MOCK_CODE_GITHUB_HEADER]: testInfo.testId,  // 关键关联
    }
    await route.continue({ headers })
  })

  let ghUser: GitHubUser | null = null
  
  // 2. 在 MSW 中创建 Mock GitHub 用户
  await use(async () => {
    const newGitHubUser = await insertGitHubUser(testInfo.testId)!
    ghUser = newGitHubUser
    return newGitHubUser
  })

  // 3. 清理：删除关联的应用用户和 Mock 用户
  const user = await prisma.user.findUnique({
    select: { id: true, name: true },
    where: { email: normalizeEmail(ghUser!.primaryEmail) },
  })
  if (user) {
    await prisma.user.delete({ where: { id: user.id } })
    await prisma.session.deleteMany({ where: { userId: user.id } })
  }
  await deleteGitHubUser(ghUser!.primaryEmail)
},
```

**关键配合点**：
```
Playwright 测试进程                    应用服务器进程 (MSW)
        │                                    │
        │  prepareGitHubUser()               │
        │  ├─ 拦截 /auth/github 请求          │
        │  │   添加 MOCK_CODE_GITHUB_HEADER  │
        │  │                                │
        │  └─ insertGitHubUser(testId)       │
        │      │                             │
        │      └─────────────────────────────>│
        │                                    │  在 MSW 中创建用户
        │                                    │  (code = testId)
        │                                    │
        │  测试: 点击 "Sign up with GitHub"   │
        │      │                             │
        │      └─────────────────────────────>│
        │                                    │  OAuth 请求被拦截
        │                                    │  根据 code=testId 查找用户
        │                                    │  返回 Mock 响应
```

### 4.3 夹具使用示例

#### 4.3.1 基础登录测试

```typescript
// tests/e2e/notes.test.ts
test('Users can create notes', async ({ page, navigate, login }) => {
  // login 夹具自动完成：创建用户 → 创建会话 → 注入 Cookie
  const user = await login()
  
  // 浏览器已登录，直接访问受保护路由
  await navigate('/users/:username/notes', { username: user.username })

  const newNote = createNote()
  await page.getByRole('link', { name: /New Note/i }).click()

  await page.getByRole('textbox', { name: /title/i }).fill(newNote.title)
  await page.getByRole('textbox', { name: /content/i }).fill(newNote.content)

  await page.getByRole('button', { name: /submit/i }).click()
  await expect(page).toHaveURL(new RegExp(`/users/${user.username}/notes/.*`))
})
```

#### 4.3.2 邮件验证测试

```typescript
// tests/e2e/onboarding.test.ts
test('onboarding with link', async ({ page, navigate, getOnboardingData }) => {
  const onboardingData = getOnboardingData()

  await navigate('/signup')
  await page.getByRole('textbox', { name: /email/i }).fill(onboardingData.email)
  await page.getByRole('button', { name: /submit/i }).click()

  // MSW 拦截邮件请求，保存到文件系统
  // 测试从文件系统读取邮件
  const email = await readEmail(onboardingData.email)
  invariant(email, 'Email not found')
  
  // 提取验证链接
  const onboardingUrl = extractUrl(email.text) as AppPages
  await navigate(onboardingUrl)
  
  // ... 继续测试
})
```

---

## 5. 完整调用时序与衔接点

### 5.1 Vitest 测试完整时序

```
┌─────────────────────────────────────────────────────────────────┐
│                    Vitest 测试执行时序                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  T0: Vitest 启动                                                │
│      │                                                          │
│      ▼                                                          │
│  global-setup.ts 执行 (一次)                                    │
│      │                                                          │
│      ├─ 检查 base.db 是否存在                                    │
│      └─ 如有需要，创建 base.db                                   │
│      │                                                          │
│      ▼                                                          │
│  T1: 测试池 0 启动 (独立进程)                                    │
│      │                                                          │
│      ▼                                                          │
│  setup-test-env.ts 执行                                         │
│      │                                                          │
│      ├─ 导入 db-setup.ts                                        │
│      │   ├─ 设置 DATABASE_URL = data.0.db                      │
│      │   └─ 注册 beforeEach/afterAll                            │
│      │                                                          │
│      ├─ 导入 MSW Server                                         │
│      │   └─ server.listen() 启动                                │
│      │                                                          │
│      └─ 注册 afterEach (resetHandlers, cleanup)                 │
│      │                                                          │
│      ▼                                                          │
│  T2: 测试用例 A 开始                                             │
│      │                                                          │
│      ├─ beforeEach (db-setup.ts)                                │
│      │   └─ 复制 base.db → data.0.db                            │
│      │                                                          │
│      ├─ 测试代码执行                                             │
│      │   ├─ 导入 prisma (连接 data.0.db)                        │
│      │   ├─ 数据库操作                                           │
│      │   └─ HTTP 请求 (被 MSW 拦截)                              │
│      │                                                          │
│      └─ afterEach                                               │
│          ├─ server.resetHandlers()                              │
│          └─ cleanup() (Testing Library)                         │
│      │                                                          │
│      ▼                                                          │
│  T3: 测试用例 B 开始                                             │
│      │                                                          │
│      ├─ beforeEach                                              │
│      │   └─ 再次复制 base.db → data.0.db (重置数据库)           │
│      │                                                          │
│      └─ ... (同用例 A)                                          │
│      │                                                          │
│      ▼                                                          │
│  T4: 测试池 0 结束                                               │
│      │                                                          │
│      └─ afterAll (db-setup.ts)                                  │
│          ├─ prisma.$disconnect()                                │
│          └─ 删除 data.0.db                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Playwright E2E 测试完整时序

```
┌─────────────────────────────────────────────────────────────────┐
│                 Playwright E2E 测试执行时序                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  T0: Playwright 启动                                            │
│      │                                                          │
│      ▼                                                          │
│  playwright.config.ts                                           │
│      │                                                          │
│      └─ 启动 webServer                                          │
│          │                                                      │
│          └─ npm run dev / npm run start:mocks                   │
│              │                                                  │
│              ▼                                                  │
│  T1: 应用服务器启动 (独立 Node.js 进程)                          │
│      │                                                          │
│      ├─ 加载 .env                                               │
│      │   └─ DATABASE_URL = file:./prisma/data.db (默认)        │
│      │                                                          │
│      ├─ 检查 MOCKS=true                                         │
│      │   └─ 导入 MSW Server                                     │
│      │       └─ server.listen() 启动                            │
│      │                                                          │
│      └─ 启动 Express + React Router                             │
│          │                                                      │
│          └─ Prisma 连接默认数据库                                │
│              │                                                  │
│              ▼                                                  │
│  T2: 应用服务器就绪，Playwright 开始执行测试                      │
│      │                                                          │
│      ▼                                                          │
│  T3: 测试用例开始                                                │
│      │                                                          │
│      ├─ 夹具 setup 阶段                                          │
│      │   │                                                      │
│      │   ├─ login 夹具:                                         │
│      │   │   ├─ 导入 prisma (连接默认数据库)                     │
│      │   │   ├─ 创建用户                                        │
│      │   │   ├─ 创建会话                                        │
│      │   │   └─ 注入 Cookie 到浏览器                            │
│      │   │                                                      │
│      │   └─ (其他夹具 setup)                                    │
│      │                                                          │
│      ├─ 测试代码执行                                             │
│      │   │                                                      │
│      │   ├─ Playwright 浏览器操作                                │
│      │   │   └─ HTTP 请求 → 应用服务器                          │
│      │   │       │                                              │
│      │   │       └─ 应用服务器处理                               │
│      │   │           ├─ 数据库操作 (默认数据库)                 │
│      │   │           └─ 外部 API 调用 (被 MSW 拦截)             │
│      │   │               │                                      │
│      │   │               └─ MSW 响应                            │
│      │   │                   └─ (如邮件写入 fixtures/)          │
│      │   │                                                      │
│      │   └─ 断言                                                │
│      │       ├─ 页面内容检查                                     │
│      │       ├─ (可选) 直接查询数据库验证                        │
│      │       └─ (可选) 从 fixtures/ 读取邮件验证                │
│      │                                                          │
│      └─ 夹具 cleanup 阶段                                        │
│          │                                                      │
│          ├─ login 夹具:                                         │
│          │   └─ 删除用户                                        │
│          │                                                      │
│          └─ (其他夹具 cleanup)                                  │
│      │                                                          │
│      ▼                                                          │
│  T4: 下一个测试用例开始                                          │
│      │                                                          │
│      └─ ... (重复 T3，数据库未重置，依赖夹具清理)                │
│      │                                                          │
│      ▼                                                          │
│  T5: 所有测试完成                                                │
│      │                                                          │
│      └─ Playwright 关闭应用服务器                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 关键衔接点分析

#### 5.3.1 衔接点 1：Playwright 测试 ↔ 应用服务器数据库

```
Playwright 测试进程                    应用服务器进程
        │                                    │
        │  夹具: prisma.note.create()        │
        │  (使用 .env 中的 DATABASE_URL)     │
        │      │                             │
        │      └─────────────────────────────>│
        │                                    │  写入默认数据库
        │                                    │
        │  浏览器: 访问 /notes/:id            │
        │      │                             │
        │      └─────────────────────────────>│
        │                                    │  读取默认数据库
        │                                    │  (读取到刚才写入的数据)
        │                                    │
        │      <─────────────────────────────│
        │                                    │
```

**关键**：Playwright 测试和应用服务器**共享同一个数据库**（`.env` 中配置的），这是它们之间的数据传递通道。

#### 5.3.2 衔接点 2：MSW ↔ 测试断言（邮件示例）

```
应用服务器进程                        Playwright 测试进程
        │                                    │
        │  resend.emails.send()              │
        │      │                             │
        │      ▼                             │
        │  MSW 拦截                          │
        │      │                             │
        │      ├─ writeEmail(body)           │
        │      │   └─ 保存到                  │
        │      │      tests/fixtures/email/  │
        │      │      ${recipient}.json      │
        │      │                             │
        │      └─ 返回 Mock 响应              │
        │                                    │
        │                                    │  readEmail(recipient)
        │                                    │      │
        │                                    │      └─ 读取
        │                                    │         tests/fixtures/email/
        │                                    │         ${recipient}.json
        │                                    │
        │                                    │  断言: email.subject = 'Welcome'
```

**关键**：MSW 和测试通过**文件系统**（`tests/fixtures/`）传递数据。

#### 5.3.3 衔接点 3：Playwright 夹具 ↔ MSW（GitHub OAuth 示例）

```
Playwright 测试进程                    应用服务器进程 (MSW)
        │                                    │
        │  prepareGitHubUser()               │
        │      │                             │
        │      ├─ 拦截 /auth/github          │
        │      │   添加 testId 到请求头      │
        │      │                             │
        │      └─ insertGitHubUser(testId)   │
        │          │                         │
        │          └─────────────────────────>│
        │                                    │  MSW 存储:
        │                                    │  code=testId → 用户数据
        │                                    │
        │  浏览器: 点击 GitHub 登录           │
        │      │                             │
        │      └─────────────────────────────>│
        │                                    │  OAuth 请求被拦截
        │                                    │  code=testId (从请求头)
        │                                    │      │
        │                                    │      └─ 查找对应的 Mock 用户
        │                                    │          │
        │                                    │          └─ 返回 Mock 响应
```

**关键**：通过 `testInfo.testId` 关联 Playwright 测试和 MSW 中的 Mock 数据。

---

## 6. 配置汇总

### 6.1 环境变量

```
# .env.example
DATABASE_URL="file:./prisma/data.db"     # 默认数据库 (Playwright 使用)
DATABASE_PATH="./prisma/data.db"          # 数据库文件路径

# 测试相关 (在代码中设置，不在 .env 中)
VITEST_POOL_ID                            # Vitest 测试池 ID (自动设置)
MOCKS=true                                # 启用 MSW (npm script 中设置)
NODE_ENV=test                             # 测试环境标识
```

### 6.2 NPM Scripts

```json
{
  "scripts": {
    // Vitest 单元测试
    "test": "vitest",
    "coverage": "vitest run --coverage",
    
    // Playwright E2E 测试
    "test:e2e": "npm run test:e2e:dev --silent",
    "test:e2e:dev": "playwright test --ui",
    "pretest:e2e:run": "npm run build",
    "test:e2e:run": "cross-env CI=true playwright test",
    
    // 开发服务器 (带 MSW)
    "dev": "cross-env NODE_ENV=development MOCKS=true node index.ts",
    "dev:no-mocks": "cross-env NODE_ENV=development node index.ts",
    
    // 生产服务器
    "start": "cross-env NODE_ENV=production node index.ts",
    "start:mocks": "cross-env NODE_ENV=production MOCKS=true node index.ts",
    
    // 完整验证
    "validate": "run-p \"test -- --run\" lint typecheck test:e2e:run"
  }
}
```

### 6.3 Vite 配置

```typescript
// vite.config.ts
export default defineConfig({
  test: {
    include: ['./app/**/*.test.{ts,tsx}'],
    
    // Vitest 独有的数据库隔离配置
    setupFiles: ['./tests/setup/setup-test-env.ts'],
    globalSetup: ['./tests/setup/global-setup.ts'],
    
    restoreMocks: true,
    coverage: {
      include: ['app/**/*.{ts,tsx}'],
      all: true,
    },
  },
  
  // Vitest 独有的缓存 Mock
  plugins: [
    {
      name: 'vitest-cache-server-stub',
      enforce: 'pre' as const,
      resolveId(source: string) {
        if (!process.env.VITEST) return null
        if (source.endsWith('cache.server.ts')) {
          return path.resolve('tests/mocks/cache-server.ts')
        }
        return null
      },
    },
  ],
})
```

### 6.4 Playwright 配置

```typescript
// playwright.config.ts
export default defineConfig({
  testDir: './tests/e2e',
  timeout: 15 * 1000,
  fullyParallel: true,
  
  use: {
    baseURL: `http://localhost:${PORT}/`,
    trace: 'on-first-retry',
  },

  // 启动应用服务器 (带 MSW)
  webServer: {
    command: process.env.CI ? 'npm run start:mocks' : 'npm run dev',
    port: Number(PORT),
    timeout: 60 * 1000,
    reuseExistingServer: true,
    env: {
      PORT,
      NODE_ENV: 'test',
      // 注意：MOCKS 在 npm script 中设置
    },
  },
})
```

---

## 7. 核心架构总结

### 7.1 两套独立系统

```
┌─────────────────────────────────────────────────────────────────┐
│                        架构总览                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Vitest 系统                           │   │
│  │                                                         │   │
│  │  进程: Vitest Worker 进程 (每个 pool 一个)               │   │
│  │  数据库: tests/prisma/data.${poolId}.db                 │   │
│  │  MSW: 通过 setupFiles 加载                               │   │
│  │  隔离: beforeEach 复制 base.db                          │   │
│  │                                                         │   │
│  │  适用: 单元测试、工具函数测试、loader/action 测试        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  Playwright 系统                         │   │
│  │                                                         │   │
│  │  进程 1: Playwright 测试进程                             │   │
│  │  ├─ 运行测试代码                                          │   │
│  │  ├─ 操作夹具 (创建用户、注入 Cookie)                      │   │
│  │  └─ 直接操作数据库 (使用 .env DATABASE_URL)              │   │
│  │                                                         │   │
│  │  进程 2: 应用服务器 (Node.js)                            │   │
│  │  ├─ 启动 Express + React Router                         │   │
│  │  ├─ 加载 MSW (如果 MOCKS=true)                          │   │
│  │  └─ 操作数据库 (使用 .env DATABASE_URL)                 │   │
│  │                                                         │   │
│  │  数据库: .env 中配置的默认数据库 (共享)                  │   │
│  │  MSW: 在应用服务器进程中加载                             │   │
│  │  隔离: 依赖夹具 cleanup 手动删除数据                     │   │
│  │                                                         │   │
│  │  适用: E2E 测试、用户流程测试                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ⚠️ 重要: Vitest 和 Playwright 不共享任何资源！                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 关键决策点

| 决策 | Vitest 方案 | Playwright 方案 |
|------|-------------|-----------------|
| **数据库隔离** | 文件复制 + 测试池 | 共享数据库 + 夹具清理 |
| **MSW 加载** | setupFiles 自动导入 | `MOCKS=true` 条件导入 |
| **进程模型** | 单测试进程 (多 worker) | 双进程 (测试 + 应用服务器) |
| **数据传递** | 内存/函数调用 | 数据库 + 文件系统 |
| **测试速度** | 快 (无浏览器) | 慢 (有浏览器 + 服务器) |
| **测试真实性** | 较低 (单元/集成) | 高 (真实浏览器) |

### 7.3 最佳实践

1. **单元测试用 Vitest**：
   - 工具函数、纯函数
   - 不需要浏览器的逻辑
   - 需要快速反馈的测试

2. **E2E 测试用 Playwright**：
   - 用户交互流程
   - 认证、授权
   - 跨组件/跨页面的流程

3. **数据库操作**：
   - Vitest：依赖 `beforeEach` 的自动重置
   - Playwright：依赖夹具的 `cleanup` 阶段手动删除

4. **MSW 使用**：
   - 不要假设 Vitest 和 Playwright 共享 MSW 状态
   - Playwright 测试中读取邮件使用 `readEmail()`
   - GitHub OAuth 测试使用 `prepareGitHubUser()` 夹具

---

## 8. 常见问题解答

### Q1: Vitest 和 Playwright 能共享数据库吗？

**不能**。它们是独立的进程，使用不同的数据库：
- Vitest 使用 `tests/prisma/data.${poolId}.db`
- Playwright 使用 `.env` 中配置的数据库

### Q2: 为什么 Playwright 不使用 `db-setup.ts`？

`db-setup.ts` 依赖 Vitest 的 `beforeEach`/`afterAll` 钩子，这些在 Playwright 中不存在。Playwright 使用夹具的生命周期来管理数据。

### Q3: MSW 在 Playwright 测试中如何工作？

MSW 运行在**应用服务器进程**中，不是 Playwright 测试进程中。当应用服务器发出 HTTP 请求时，MSW 拦截这些请求。

### Q4: 如何在 Playwright 测试中验证邮件发送？

MSW 拦截邮件请求后，将邮件内容写入 `tests/fixtures/email/${recipient}.json`。测试使用 `readEmail(recipient)` 读取该文件进行断言。

### Q5: 为什么 `prepareGitHubUser` 要设置 `MOCK_CODE_GITHUB_HEADER`？

Playwright 测试和 MSW 是独立的进程，无法直接共享内存。通过在请求头中传递 `testId`，MSW 可以找到对应的 Mock 用户数据。
