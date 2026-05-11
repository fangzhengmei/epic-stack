# Epic Stack 测试基础设施分析报告

## 1. 概述

Epic Stack 采用了一套完整、分层的测试体系，包括单元测试、集成测试和端到端（E2E）测试。核心组成部分包括：

- **数据库快照机制**：基于文件系统的数据库隔离和快速重置
- **网络请求 Mock (MSW)**：Mock Service Worker 实现的服务端网络拦截
- **Playwright E2E 测试夹具**：扩展的测试工具和自定义夹具

本报告深入分析这三个核心组件如何协同工作，构建可靠、可复现的测试环境。

---

## 2. 数据库快照机制

### 2.1 核心原理

数据库快照机制的核心思想是：**创建一个基础数据库模板，每个测试运行时复制该模板，测试完成后删除副本**。

### 2.2 实现细节

#### 2.2.1 全局设置 (`tests/setup/global-setup.ts`)

```typescript
export const BASE_DATABASE_PATH = path.join(
  process.cwd(),
  `./tests/prisma/base.db`,
)

export async function setup() {
  const databaseExists = await fsExtra.pathExists(BASE_DATABASE_PATH)

  if (databaseExists) {
    const databaseLastModifiedAt = (await fsExtra.stat(BASE_DATABASE_PATH)).mtime
    const prismaSchemaLastModifiedAt = (
      await fsExtra.stat('./prisma/schema.prisma')
    ).mtime

    if (prismaSchemaLastModifiedAt < databaseLastModifiedAt) {
      return
    }
  }

  await execaCommand(
    'npx prisma migrate reset --force --skip-seed --skip-generate',
    {
      stdio: 'inherit',
      env: {
        ...process.env,
        DATABASE_URL: `file:${BASE_DATABASE_PATH}`,
        PRISMA_USER_CONSENT_FOR_DANGEROUS_AI_ACTION: 'true',
      },
    },
  )
}
```

**关键特性**：
- 检查 `base.db` 是否已存在
- 比较 schema 和数据库的修改时间，仅在 schema 更新时重建
- 使用 `prisma migrate reset` 创建干净的基础数据库

#### 2.2.2 每个测试池的数据库隔离 (`tests/setup/db-setup.ts`)

```typescript
const poolId = process.env.VITEST_POOL_ID || '0'
const databaseFile = `./tests/prisma/data.${poolId}.db`
const databasePath = path.join(process.cwd(), databaseFile)
process.env.DATABASE_URL = `file:${databasePath}`

beforeEach(async () => {
  await fsExtra.copyFile(BASE_DATABASE_PATH, databasePath)
})

afterAll(async () => {
  const { prisma } = await import('#app/utils/db.server.ts')
  await prisma.$disconnect()
  await fsExtra.remove(databasePath)
})
```

**关键特性**：
- 为每个 Vitest 测试池创建独立的数据库文件（`data.${poolId}.db`）
- 每个测试用例前复制基础数据库，确保测试隔离
- 测试完成后断开 Prisma 连接并删除数据库文件
- 同样支持缓存数据库的隔离

#### 2.2.3 缓存数据库隔离

```typescript
const cacheDatabasePath = process.env.CACHE_DATABASE_PATH
if (cacheDatabasePath && cacheDatabasePath !== ':memory:') {
  const parsed = path.parse(cacheDatabasePath)
  const cacheFileName = parsed.ext
    ? `${parsed.name}.${poolId}${parsed.ext}`
    : `${parsed.name}.${poolId}`
  const cacheDir = parsed.dir || '.'
  process.env.CACHE_DATABASE_PATH = path.join(cacheDir, cacheFileName)
}
```

### 2.3 工作流程

```
测试启动
    ↓
global-setup.ts: 创建/验证 base.db（基础模板）
    ↓
db-setup.ts: 为每个测试池设置独立数据库路径
    ↓
每个测试用例开始前:
    ↓
beforeEach: 复制 base.db → data.${poolId}.db
    ↓
测试运行（操作独立数据库）
    ↓
测试结束
    ↓
afterAll: 断开连接，删除 data.${poolId}.db
```

### 2.4 优势

1. **测试隔离**：每个测试池有独立数据库，避免测试间干扰
2. **快速重置**：文件复制比数据库迁移快得多
3. **一致性**：所有测试从相同的基础状态开始
4. **并行安全**：支持 Vitest 的并行测试执行

---

## 3. 网络请求 Mock (MSW)

### 3.1 核心原理

使用 **Mock Service Worker (MSW)** 实现服务端网络请求拦截，模拟外部 API 响应。

### 3.2 实现架构

#### 3.2.1 Mock 服务器入口 (`tests/mocks/index.ts`)

```typescript
import closeWithGrace from 'close-with-grace'
import { setupServer } from 'msw/node'
import { handlers as githubHandlers } from './github.ts'
import { handlers as pwnedPasswordApiHandlers } from './pwned-passwords.ts'
import { handlers as resendHandlers } from './resend.ts'
import { handlers as tigrisHandlers } from './tigris.ts'

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

if (process.env.NODE_ENV !== 'test') {
  console.info('🔶 Mock server installed')
  closeWithGrace(() => {
    server.close()
  })
}
```

**关键特性**：
- 整合多个服务的 handlers
- 智能忽略 Sentry 和 React Router DevTools 请求
- 开发环境时输出提示信息
- 优雅关闭机制

#### 3.2.2 条件加载 (`index.ts`)

```typescript
if (process.env.MOCKS === 'true') {
  await import('./tests/mocks/index.ts')
}
```

通过 `MOCKS=true` 环境变量控制是否启用 Mock。

### 3.3 Mock 服务详解

#### 3.3.1 GitHub OAuth Mock (`tests/mocks/github.ts`)

这是最复杂的 Mock 实现，展示了如何模拟完整的 OAuth 流程：

```typescript
// 存储 Mock 用户数据到文件系统
const githubUserFixturePath = path.join(
  here(
    '..',
    'fixtures',
    'github',
    `users.${process.env.VITEST_POOL_ID || 0}.local.json`,
  ),
)

// Mock handlers
export const handlers: Array<HttpHandler> = [
  // 1. OAuth 令牌交换
  http.post(
    'https://github.com/login/oauth/access_token',
    async ({ request }) => {
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
  
  // 2. 获取用户邮箱
  http.get('https://api.github.com/user/emails', async ({ request }) => {
    const user = await getUser(request)
    if (user instanceof Response) return user
    return json(user.emails)
  }),
  
  // 3. 获取用户资料
  http.get('https://api.github.com/user', async ({ request }) => {
    const user = await getUser(request)
    if (user instanceof Response) return user
    return json(user.profile)
  }),
  
  // 4. Mock 头像图片
  http.get('https://github.com/ghost.png', async () => {
    const buffer = await fsExtra.readFile('./tests/fixtures/github/ghost.jpg')
    return new Response(buffer, {
      headers: { 'content-type': 'image/jpg' },
    })
  }),
]
```

**关键特性**：
- 基于测试池隔离的用户数据存储
- 支持动态创建用户或查找已有用户
- 完整模拟 OAuth 流程的各个端点
- 支持图片资源的 Mock
- 条件直通（passthrough）机制

```typescript
const passthroughGitHub =
  !process.env.GITHUB_CLIENT_ID?.startsWith('MOCK_') &&
  process.env.NODE_ENV !== 'test'
```

当使用真实 GitHub 凭证且非测试环境时，请求会直通到真实 API。

#### 3.3.2 Resend 邮件服务 Mock (`tests/mocks/resend.ts`)

```typescript
export const handlers: Array<HttpHandler> = [
  http.post(`https://api.resend.com/emails`, async ({ request }) => {
    requireHeader(request.headers, 'Authorization')
    const body = await request.json()
    console.info('🔶 mocked email contents:', body)

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

配合 `tests/mocks/utils.ts` 中的工具函数：

```typescript
export async function writeEmail(rawEmail: unknown) {
  const email = EmailSchema.parse(rawEmail)
  await createFixture('email', email.to, email)
  return email
}

export async function readEmail(recipient: string) {
  try {
    const email = await readFixture('email', recipient)
    return EmailSchema.parse(email)
  } catch (error) {
    console.error(`Error reading email`, error)
    return null
  }
}
```

**工作流程**：
1. 拦截 `https://api.resend.com/emails` 请求
2. 验证 Authorization 头
3. 将邮件内容写入 `tests/fixtures/email/${recipient}.json`
4. 返回模拟的成功响应
5. 测试中通过 `readEmail()` 读取邮件内容进行断言

### 3.4 测试环境集成 (`tests/setup/setup-test-env.ts`)

```typescript
import 'dotenv/config'
import './db-setup.ts'
import '#app/utils/env.server.ts'

import { cleanup } from '@testing-library/react'
import { afterEach, beforeEach, vi, type MockInstance } from 'vitest'
import { server } from '#tests/mocks/index.ts'
import './custom-matchers.ts'

afterEach(() => server.resetHandlers())
afterEach(() => cleanup())

// Console 错误/警告监控
beforeEach(() => {
  const originalConsoleError = console.error
  consoleError = vi.spyOn(console, 'error')
  consoleError.mockImplementation(
    (...args: Parameters<typeof console.error>) => {
      originalConsoleError(...args)
      throw new Error(
        'Console error was called. Call consoleError.mockImplementation(() => {}) if this is expected.',
      )
    },
  )
  // ... 类似的 console.warn 监控
})
```

**关键特性**：
- 每次测试后重置 MSW handlers
- 监控 console.error 和 console.warn，防止静默失败
- 集成 Testing Library cleanup

### 3.5 Vite 插件中的额外 Mock

在 `vite.config.ts` 中，还有一个特殊的缓存服务器 Stub：

```typescript
const cacheServerStubPlugin = {
  name: 'vitest-cache-server-stub',
  enforce: 'pre' as const,
  resolveId(source: string) {
    if (!process.env.VITEST) return null
    if (source.endsWith('cache.server.ts')) {
      return path.resolve('tests/mocks/cache-server.ts')
    }
    return null
  },
}
```

这个插件在 Vitest 环境下，将所有对 `cache.server.ts` 的导入重定向到 `tests/mocks/cache-server.ts`，实现缓存层的 Mock。

---

## 4. Playwright E2E 测试夹具

### 4.1 核心原理

Playwright 夹具（Fixtures）扩展了基础测试功能，提供：
- 预配置的测试数据
- 自动的资源管理
- 简化的测试编写

### 4.2 自定义夹具定义 (`tests/playwright-utils.ts`)

#### 4.2.1 扩展的测试类型

```typescript
export const test = base.extend<{
  navigate: <Path extends AppPages>(
    ...args: Parameters<typeof href<Path>>
  ) => Promise<null | Response>
  insertNewUser(options?: GetOrInsertUserOptions): Promise<User>
  login(options?: GetOrInsertUserOptions): Promise<User>
  prepareGitHubUser(): Promise<GitHubUser>
}>({
  // ... 夹具实现
})
```

#### 4.2.2 关键夹具详解

##### 4.2.2.1 `navigate` 夹具

```typescript
navigate: async ({ page }, use) => {
  await use((...args) => {
    return page.goto(href(...args))
  })
},
```

**用途**：类型安全的路由导航，使用 React Router 的 `href` 函数。

**示例**：
```typescript
await navigate('/users/:username/notes', { username: user.username })
```

##### 4.2.2.2 `insertNewUser` 夹具

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

**生命周期**：
1. 测试开始前：创建用户（可自定义属性）
2. 测试运行中：提供用户对象
3. 测试结束后：自动删除用户

##### 4.2.2.3 `login` 夹具（核心夹具）

```typescript
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
      select: { id: true },
    })

    // 创建认证 Cookie
    const authSession = await authSessionStorage.getSession()
    authSession.set(sessionKey, session.id)
    const cookieConfig = setCookieParser.parseString(
      await authSessionStorage.commitSession(authSession),
    )
    const newConfig = {
      ...cookieConfig,
      domain: 'localhost',
      expires: cookieConfig.expires?.getTime(),
      sameSite: cookieConfig.sameSite as 'Strict' | 'Lax' | 'None',
    }
    
    // 注入 Cookie 到浏览器上下文
    await page.context().addCookies([newConfig])
    return user
  })
  await prisma.user.deleteMany({ where: { id: userId } })
},
```

**工作流程**：
1. 创建或获取用户
2. 在数据库中创建会话记录
3. 生成认证 Cookie
4. 将 Cookie 注入 Playwright 浏览器上下文
5. 测试完成后清理用户

**使用示例**：
```typescript
test('Users can create notes', async ({ page, navigate, login }) => {
  const user = await login()  // 一步完成用户创建和登录
  await navigate('/users/:username/notes', { username: user.username })
  // ... 测试逻辑
})
```

##### 4.2.2.4 `prepareGitHubUser` 夹具

```typescript
prepareGitHubUser: async ({ page }, use, testInfo) => {
  // 拦截 GitHub OAuth 请求，添加测试 ID 头
  await page.route(/\/auth\/github(?!\/callback)/, async (route, request) => {
    const headers = {
      ...request.headers(),
      [MOCK_CODE_GITHUB_HEADER]: testInfo.testId,
    }
    await route.continue({ headers })
  })

  let ghUser: GitHubUser | null = null
  await use(async () => {
    const newGitHubUser = await insertGitHubUser(testInfo.testId)!
    ghUser = newGitHubUser
    return newGitHubUser
  })

  // 清理：删除关联的应用用户和 GitHub Mock 用户
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

**关键特性**：
- 使用 `testInfo.testId` 关联测试和 Mock 用户
- 通过请求头传递测试 ID，让 MSW 能识别对应的 Mock 数据
- 自动清理应用用户和 Mock 用户

### 4.3 测试中使用夹具的示例

#### 4.3.1 基础功能测试 (`tests/e2e/notes.test.ts`)

```typescript
test('Users can create notes', async ({ page, navigate, login }) => {
  const user = await login()  // 创建用户并登录
  await navigate('/users/:username/notes', { username: user.username })

  const newNote = createNote()
  await page.getByRole('link', { name: /New Note/i }).click()

  await page.getByRole('textbox', { name: /title/i }).fill(newNote.title)
  await page.getByRole('textbox', { name: /content/i }).fill(newNote.content)

  await page.getByRole('button', { name: /submit/i }).click()
  await expect(page).toHaveURL(new RegExp(`/users/${user.username}/notes/.*`))
})
```

#### 4.3.2 邮件验证流程测试 (`tests/e2e/onboarding.test.ts`)

```typescript
test('onboarding with link', async ({ page, navigate, getOnboardingData }) => {
  const onboardingData = getOnboardingData()

  await navigate('/')
  await page.getByRole('link', { name: /log in/i }).click()
  
  // ... 填写注册表单
  const emailTextbox = page.getByRole('textbox', { name: /email/i })
  await emailTextbox.fill(onboardingData.email)
  await page.getByRole('button', { name: /submit/i }).click()
  
  // 从 Mock 中读取邮件
  const email = await readEmail(onboardingData.email)
  invariant(email, 'Email not found')
  
  // 提取验证链接
  const onboardingUrl = extractUrl(email.text) as AppPages
  await navigate(onboardingUrl)
  
  // ... 完成注册流程
})
```

#### 4.3.3 GitHub OAuth 测试

```typescript
test('completes onboarding after GitHub OAuth given valid user details', async ({
  page,
  navigate,
  prepareGitHubUser,
}) => {
  const ghUser = await prepareGitHubUser()  // 准备 Mock GitHub 用户

  // 验证用户不存在
  expect(
    await prisma.user.findUnique({
      where: { email: normalizeEmail(ghUser.primaryEmail) },
    }),
  ).toBeNull()

  // 触发 GitHub 登录流程
  await navigate('/signup')
  await page.getByRole('button', { name: /signup with github/i }).click()

  // 验证跳转到 onboarding 页面
  await expect(page).toHaveURL(/\/onboarding\/github/)
  
  // ... 完成注册
})
```

---

## 5. 三者配合机制

### 5.1 启动流程

```
Playwright 测试启动
    ↓
playwright.config.ts 配置 webServer
    ↓
┌─────────────────────────────────────────┐
│  webServer 启动命令:                      │
│  - CI: npm run start:mocks              │
│  - 开发: npm run dev                    │
│                                         │
│  设置环境变量:                            │
│  - NODE_ENV=test                        │
│  - MOCKS=true (CI 模式)                  │
└─────────────────────────────────────────┘
    ↓
index.ts 入口文件
    ↓
┌─────────────────────────────────────────┐
│  if (process.env.MOCKS === 'true') {    │
│    await import('./tests/mocks/index.ts')│
│  }                                      │
└─────────────────────────────────────────┘
    ↓
MSW Server 启动，拦截所有外部 API 请求
    ↓
服务器就绪，Playwright 开始执行测试
```

### 5.2 测试执行流程

以 `onboarding.test.ts` 中的 `onboarding with link` 测试为例：

```
测试开始
    ↓
getOnboardingData 夹具
    ├─ 使用 faker 生成随机用户数据
    └─ 测试结束后: prisma.user.deleteMany()
    ↓
navigate('/') 夹具
    └─ 使用类型安全的路由导航
    ↓
用户操作: 点击登录 → 填写邮箱 → 提交
    ↓
应用发送邮件请求到 resend.com
    ↓
MSW 拦截请求
    ├─ resend.ts handler 处理
    ├─ 调用 writeEmail() 保存到 fixtures/email/
    └─ 返回模拟的成功响应
    ↓
测试代码调用 readEmail(onboardingData.email)
    └─ 从 fixtures/email/ 读取邮件内容
    ↓
提取验证链接，继续测试流程
    ↓
测试结束
    ↓
夹具自动清理: 删除测试用户
```

### 5.3 数据流向图

```
┌─────────────────────────────────────────────────────────────┐
│                    测试执行环境                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Playwright 浏览器                                            │
│       │                                                     │
│       │ 页面操作 / 表单提交                                    │
│       ▼                                                     │
│  应用服务器 (Express + React Router)                          │
│       │                                                     │
│       ├──────────────────────┐                              │
│       │ 内部数据库操作          │ 外部 API 调用                  │
│       ▼                      ▼                              │
│  ┌─────────────┐    ┌──────────────┐                        │
│  │ 隔离数据库    │    │   MSW Server │                        │
│  │ (SQLite)    │    │              │                        │
│  └─────────────┘    └──────────────┘                        │
│       │                      │                              │
│       │ 读写                  │ 拦截                        │
│       ▼                      ▼                              │
│  data.${poolId}.db    ┌──────────────┐                      │
│       ^               │ Mock Handlers│                      │
│       │               └──────────────┘                      │
│       │ 复制                     │                          │
│       │                          ▼                          │
│  base.db (基础模板)      ┌──────────────┐                    │
│                          │ Fixtures     │                    │
│                          │ (文件系统)    │                    │
│                          │ - email/     │                    │
│                          │ - github/    │                    │
│                          │ - images/    │                    │
│                          └──────────────┘                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.4 关键配合点

#### 5.4.1 数据库 + 夹具

```typescript
// 夹具直接操作隔离的数据库
login: async ({ page }, use) => {
  const user = await getOrInsertUser(options)  // 写入 data.${poolId}.db
  const session = await prisma.session.create(...)  // 写入隔离数据库
  // ...
})
```

夹具使用的 Prisma 实例会自动连接到当前测试池的隔离数据库。

#### 5.4.2 MSW + 夹具

```typescript
// GitHub 夹具和 MSW 通过 testId 关联
prepareGitHubUser: async ({ page }, use, testInfo) => {
  // 1. 拦截请求，添加测试 ID 头
  await page.route(/\/auth\/github(?!\/callback)/, async (route, request) => {
    const headers = {
      ...request.headers(),
      [MOCK_CODE_GITHUB_HEADER]: testInfo.testId,  // 关键关联
    }
    await route.continue({ headers })
  })

  // 2. 在 MSW 中创建对应用户
  await use(async () => {
    const newGitHubUser = await insertGitHubUser(testInfo.testId)!
    return newGitHubUser
  })
}

// MSW 中根据 code（包含 testId）查找用户
http.post('https://github.com/login/oauth/access_token', async ({ request }) => {
  const params = new URLSearchParams(await request.text())
  const code = params.get('code')  // 这个 code 关联到 testId
  const githubUsers = await getGitHubUsers()
  let user = githubUsers.find((u) => u.code === code)
  // ...
})
```

#### 5.4.3 邮件 Mock + 测试断言

```typescript
// 应用发送邮件（被 MSW 拦截）
await resend.emails.send({
  to: user.email,
  from: 'hello@epicstack.dev',
  subject: 'Welcome!',
  // ...
})

// MSW 保存到 fixtures
await writeEmail(body)  // 保存到 tests/fixtures/email/${to}.json

// 测试代码读取并断言
const email = await readEmail(user.email)
expect(email.subject).toMatch(/welcome/i)
expect(email.from).toBe('hello@epicstack.dev')
```

---

## 6. 配置文件汇总

### 6.1 package.json 脚本

```json
{
  "scripts": {
    "dev": "cross-env NODE_ENV=development MOCKS=true node index.ts",
    "dev:no-mocks": "cross-env NODE_ENV=development node index.ts",
    "start:mocks": "cross-env NODE_ENV=production MOCKS=true node index.ts",
    "test": "vitest",
    "test:e2e": "npm run test:e2e:dev --silent",
    "test:e2e:dev": "playwright test --ui",
    "test:e2e:run": "cross-env CI=true playwright test",
    "validate": "run-p \"test -- --run\" lint typecheck test:e2e:run"
  }
}
```

### 6.2 playwright.config.ts

```typescript
export default defineConfig({
  testDir: './tests/e2e',
  timeout: 15 * 1000,
  expect: {
    timeout: 5 * 1000,
  },
  fullyParallel: true,
  workers: process.env.CI ? 1 : undefined,
  
  use: {
    baseURL: `http://localhost:${PORT}/`,
    trace: 'on-first-retry',
  },

  webServer: {
    command: process.env.CI ? 'npm run start:mocks' : 'npm run dev',
    port: Number(PORT),
    timeout: 60 * 1000,
    reuseExistingServer: true,
    env: {
      PORT,
      NODE_ENV: 'test',
    },
  },
})
```

### 6.3 vite.config.ts 测试配置

```typescript
test: {
  include: ['./app/**/*.test.{ts,tsx}'],
  setupFiles: ['./tests/setup/setup-test-env.ts'],
  globalSetup: ['./tests/setup/global-setup.ts'],
  restoreMocks: true,
  coverage: {
    include: ['app/**/*.{ts,tsx}'],
    all: true,
  },
}
```

---

## 7. 测试目录结构

```
tests/
├── e2e/                      # E2E 测试文件
│   ├── 2fa.test.ts
│   ├── error-boundary.test.ts
│   ├── note-images.test.ts
│   ├── notes.test.ts
│   ├── onboarding.test.ts
│   ├── passkey.test.ts
│   ├── search.test.ts
│   └── settings-profile.test.ts
│
├── fixtures/                 # 静态夹具数据
│   ├── github/
│   │   └── ghost.jpg
│   ├── images/
│   │   ├── kody-notes/
│   │   ├── notes/
│   │   └── user/
│   └── email/               # 运行时生成的邮件夹具
│
├── mocks/                    # MSW Mock 定义
│   ├── index.ts             # Mock Server 入口
│   ├── github.ts            # GitHub OAuth Mock
│   ├── resend.ts            # 邮件服务 Mock
│   ├── pwned-passwords.ts   # 密码泄露检查 Mock
│   ├── tigris.ts            # 对象存储 Mock
│   ├── cache-server.ts      # 缓存服务 Stub
│   └── utils.ts             # Mock 工具函数
│
├── setup/                    # 测试环境设置
│   ├── global-setup.ts      # 全局设置（数据库模板）
│   ├── db-setup.ts          # 每个测试池的数据库隔离
│   ├── setup-test-env.ts    # Vitest 测试环境
│   └── custom-matchers.ts   # 自定义断言
│
├── db-utils.ts              # 数据库工具（生成测试数据）
├── playwright-utils.ts      # Playwright 扩展夹具
└── utils.ts                 # 通用测试工具
```

---

## 8. 设计亮点与最佳实践

### 8.1 设计亮点

1. **分层隔离**
   - 测试池级别的数据库隔离
   - 测试用例级别的数据重置
   - 并行执行安全

2. **真实环境模拟**
   - MSW 在真实网络层面拦截，而非代码层面替换
   - 支持条件直通，可切换真实/模拟 API
   - 完整的 OAuth 流程模拟

3. **夹具驱动**
   - 自动资源管理（创建/清理）
   - 测试代码简洁，聚焦业务逻辑
   - 可组合的夹具系统

4. **可调试性**
   - 邮件内容保存到文件系统，便于调试
   - Playwright trace 功能
   - Console 错误/警告监控

### 8.2 最佳实践

1. **测试数据生成**
   - 使用 `@faker-js/faker` 生成随机测试数据
   - `enforce-unique` 确保唯一性约束

2. **断言策略**
   - 优先使用角色选择器（`getByRole`）
   - 避免脆弱的选择器
   - 使用自定义匹配器扩展断言能力

3. **清理策略**
   - 夹具自动清理（`after` 阶段）
   - 数据库文件删除而非 truncate
   - MSW handlers 每次测试后重置

4. **环境变量控制**
   - `MOCKS=true/false` 控制 Mock 开关
   - `NODE_ENV=test` 标识测试环境
   - `VITEST_POOL_ID` 实现并行隔离

---

## 9. 总结

Epic Stack 的测试基础设施是一个精心设计的系统，三个核心组件各司其职又紧密配合：

| 组件 | 职责 | 关键技术 |
|------|------|----------|
| **数据库快照** | 提供隔离、一致的数据库环境 | 文件复制、测试池隔离 |
| **MSW Mock** | 拦截外部 API，模拟第三方服务 | 服务端 Worker、条件直通 |
| **Playwright 夹具** | 简化测试编写，自动资源管理 | 扩展测试、生命周期钩子 |

这套体系的核心价值在于：
- **可靠性**：隔离的环境确保测试可复现
- **效率**：快照复制比数据库迁移快，夹具减少样板代码
- **灵活性**：可切换真实/模拟 API，支持并行执行
- **可维护性**：清晰的分层和职责分离

这种设计不仅适用于 Epic Stack 本身，也可以作为现代 Web 应用测试基础设施的参考架构。
