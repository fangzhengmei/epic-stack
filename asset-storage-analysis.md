# Epic Stack 资产存储链路分析报告

## 1. 概述

Epic Stack 采用了 **Tigris**（一个与 S3 兼容的对象存储服务）作为头像和附件的存储解决方案。该架构实现了从浏览器提交、服务端签名到外部对象存储直传的完整链路，并通过服务端代理的方式处理图像访问，为后续实现私有与公开资产的鉴权机制奠定了基础。

---

## 2. 技术架构

### 2.1 存储服务配置

项目使用以下环境变量配置 S3 兼容的对象存储服务：

| 环境变量 | 说明 | 示例值 |
|---------|------|--------|
| `AWS_ENDPOINT_URL_S3` | S3 兼容服务端点 | `https://fly.storage.tigris.dev` |
| `BUCKET_NAME` | 存储桶名称 | 项目特定 |
| `AWS_ACCESS_KEY_ID` | 访问密钥 ID | 认证凭证 |
| `AWS_SECRET_ACCESS_KEY` | 秘密访问密钥 | 认证凭证 |
| `AWS_REGION` | 区域 | `auto` |

**核心文件**：[app/utils/storage.server.ts](app/utils/storage.server.ts:5-9)

### 2.2 数据库模型设计

项目采用**混合存储策略**：
- **元数据**（所有权、引用关系等）存储在 SQLite 数据库
- **二进制数据**（实际文件内容）存储在 Tigris 对象存储

#### 用户头像模型
```prisma
model UserImage {
  id        String  @id @default(cuid())
  altText   String?
  objectKey String   // Tigris 中的对象引用键
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  user      User    @relation(fields: [userId], references: [id])
  userId    String  @unique
}
```

#### 笔记图片模型
```prisma
model NoteImage {
  id        String  @id @default(cuid())
  altText   String?
  objectKey String   // Tigris 中的对象引用键
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  note      Note    @relation(fields: [noteId], references: [id])
  noteId    String
}
```

**核心文件**：[prisma/schema.prisma](prisma/schema.prisma:66-76)

---

## 3. 完整上传链路分析

### 3.1 链路总览

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   浏览器    │ ──▶ │  Remix Action│ ──▶ │   服务端    │ ──▶ │   Tigris    │
│  (文件选择) │     │  (表单处理)  │     │  (签名上传) │     │  (对象存储) │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │                   │
       │  1. 选择文件       │  2. 提交表单      │  3. 生成签名请求   │  4. 接收文件    │
       │  2. 预览图片       │  3. 验证文件      │  5. 直传 Tigris   │  5. 返回成功    │
       │  3. 提交表单       │  4. 调用上传函数  │  6. 存储 objectKey │
       └───────────────────┘                   │
       ▼                                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                      7. 重定向到用户资料页面                       │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 详细步骤分析

#### 步骤 1：浏览器文件选择与提交

**文件**：[app/routes/settings/profile/photo.tsx](app/routes/settings/profile/photo.tsx:109-236)

用户在浏览器中选择头像文件：

1. **文件选择**：使用隐藏的 `<input type="file">` 元素
2. **本地预览**：通过 `FileReader.readAsDataURL()` 在上传前预览图片
3. **表单提交**：使用 `POST` 请求，`multipart/form-data` 编码

```tsx
<input
  {...getInputProps(fields.photoFile, { type: 'file' })}
  accept="image/*"
  className="peer sr-only"
  required
  onChange={(e) => {
    const file = e.currentTarget.files?.[0]
    if (file) {
      const reader = new FileReader()
      reader.onload = (event) => {
        setNewImageSrc(event.target?.result?.toString() ?? null)
      }
      reader.readAsDataURL(file)
    }
  }}
/>
```

**验证规则**：
- 最大文件大小：3MB (`MAX_SIZE = 1024 * 1024 * 3`)
- 文件类型：仅接受图片 (`accept="image/*"`)

#### 步骤 2：服务端 Action 处理

**文件**：[app/routes/settings/profile/photo.tsx](app/routes/settings/profile/photo.tsx:66-107)

服务端接收并处理上传请求：

1. **用户认证**：通过 `requireUserId(request)` 确保用户已登录
2. **表单解析**：使用 `parseFormData()` 解析 multipart 表单
3. **Zod 验证**：使用 Zod  schema 验证文件

```typescript
export async function action({ request }: Route.ActionArgs) {
  const userId = await requireUserId(request)  // 1. 认证用户

  const formData = await parseFormData(request, { maxFileSize: MAX_SIZE })
  const submission = await parseWithZod(formData, {
    schema: PhotoFormSchema.transform(async (data) => {
      if (data.intent === 'delete') return { intent: 'delete' }
      return {
        intent: data.intent,
        image: {
          objectKey: await uploadProfileImage(userId, data.photoFile),  // 2. 调用上传
        },
      }
    }),
    async: true,
  })

  // 3. 数据库操作
  await prisma.$transaction(async ($prisma) => {
    await $prisma.userImage.deleteMany({ where: { userId } })
    await $prisma.user.update({
      where: { id: userId },
      data: { image: { create: image } },
    })
  })

  return redirect('/settings/profile')
}
```

#### 步骤 3：服务端签名与上传

**文件**：[app/utils/storage.server.ts](app/utils/storage.server.ts:11-38)

核心上传流程：

```typescript
async function uploadToStorage(file: File | FileUpload, key: string) {
  const { url, headers } = getSignedPutRequestInfo(file, key)  // 1. 生成签名请求

  const uploadResponse = await fetch(url, {
    method: 'PUT',
    headers,
    body: file instanceof File ? file : (file as FileUpload).stream(),  // 2. 流式上传
  })

  if (!uploadResponse.ok) {
    throw new Error(`Failed to upload object: ${key}`)
  }

  return key  // 3. 返回 objectKey
}
```

#### 步骤 4：AWS4 签名算法实现

**文件**：[app/utils/storage.server.ts](app/utils/storage.server.ts:52-148)

项目**没有使用 S3 SDK**，而是**手动实现了 AWS4-HMAC-SHA256 签名算法**：

```typescript
// 1. HMAC-SHA256 哈希函数
function hmacSha256(key: string | Buffer, message: string) {
  const hmac = createHmac('sha256', key)
  hmac.update(message)
  return hmac.digest()
}

// 2. SHA256 哈希函数
function sha256(message: string) {
  const hash = createHash('sha256')
  hash.update(message)
  return hash.digest('hex')
}

// 3. 生成签名密钥（4层派生）
function getSignatureKey(
  key: string,
  dateStamp: string,
  regionName: string,
  serviceName: string,
) {
  const kDate = hmacSha256(`AWS4${key}`, dateStamp)
  const kRegion = hmacSha256(kDate, regionName)
  const kService = hmacSha256(kRegion, serviceName)
  const kSigning = hmacSha256(kService, 'aws4_request')
  return kSigning
}
```

**签名请求生成流程**：

1. **构建规范请求 (Canonical Request)**
   - HTTP 方法 (PUT/GET)
   - 规范 URI (`/${bucket}/${key}`)
   - 规范查询字符串（空）
   - 规范请求头
   - 签名请求头列表
   - 负载哈希 (`UNSIGNED-PAYLOAD`)

2. **构建待签名字符串 (String to Sign)**
   - 算法标识 (`AWS4-HMAC-SHA256`)
   - 请求时间戳 (ISO 8601)
   - 凭证范围 (`date/region/service/aws4_request`)
   - 规范请求的 SHA256 哈希

3. **计算签名**
   - 使用 4 层派生的签名密钥
   - HMAC-SHA256 哈希待签名字符串

4. **构建 Authorization 头**
   ```
   AWS4-HMAC-SHA256 Credential=accessKey/date/region/service/aws4_request, 
   SignedHeaders=content-type;host;x-amz-content-sha256;x-amz-date;..., 
   Signature=calculated_signature
   ```

#### 步骤 5：对象键命名策略

**文件**：[app/utils/storage.server.ts](app/utils/storage.server.ts:29-50)

项目采用**层级化命名策略**，便于后续权限控制：

##### 头像图片路径
```typescript
export async function uploadProfileImage(
  userId: string,
  file: File | FileUpload,
) {
  const fileId = createId()
  const fileExtension = file.name.split('.').pop() || ''
  const timestamp = Date.now()
  const key = `users/${userId}/profile-images/${timestamp}-${fileId}.${fileExtension}`
  // 示例: users/clg23.../profile-images/1714900000000-abc123.jpg
  return uploadToStorage(file, key)
}
```

##### 笔记图片路径
```typescript
export async function uploadNoteImage(
  userId: string,
  noteId: string,
  file: File | FileUpload,
) {
  const fileId = createId()
  const fileExtension = file.name.split('.').pop() || ''
  const timestamp = Date.now()
  const key = `users/${userId}/notes/${noteId}/images/${timestamp}-${fileId}.${fileExtension}`
  // 示例: users/clg23.../notes/note456/images/1714900000000-def456.png
  return uploadToStorage(file, key)
}
```

**命名策略优势**：
- **可追溯性**：从路径可直接识别所有者和用途
- **权限控制**：路径结构天然支持按用户/笔记隔离
- **唯一性**：使用 `createId()` (CUID) 和时间戳确保无冲突

---

## 4. 访问链路分析

### 4.1 访问总览

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────┐
│   浏览器    │ ──▶ │  资源路由 (SSR)  │ ──▶ │  签名 GET 请求  │ ──▶ │   Tigris    │
│  (显示图片) │     │ /resources/images│     │   (服务端代理)  │     │  (返回文件) │
└─────────────┘     └─────────────────┘     └─────────────────┘     └─────────────┘
       │                   │                        │                        │
       │  1. 请求图片 URL   │  2. 解析 objectKey     │  3. 生成签名 GET 请求   │  4. 返回文件内容 │
       │                   │  4. 本地缓存检查        │  5. 从 Tigris 获取      │
       │                   │  6. 返回响应（带缓存头） │
       ▼                   ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│  URL 格式: /resources/images?objectKey=users%2FuserId%2Fprofile-images%2F... │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 详细步骤分析

#### 步骤 1：前端 URL 生成

**文件**：[app/utils/misc.tsx](app/utils/misc.tsx:8-12)

前端通过辅助函数生成访问 URL：

```typescript
export function getUserImgSrc(objectKey?: string | null) {
  return objectKey
    ? `/resources/images?objectKey=${encodeURIComponent(objectKey)}`
    : '/img/user.png'
}
```

#### 步骤 2：服务端资源路由处理

**文件**：[app/routes/resources/images.tsx](app/routes/resources/images.tsx:28-80)

资源路由作为**服务端代理**处理所有图片请求：

```typescript
export async function loader({ request }: Route.LoaderArgs) {
  const url = new URL(request.url)
  const searchParams = url.searchParams

  const headers = new Headers()
  headers.set('Cache-Control', 'public, max-age=31536000, immutable')  // 1 年缓存

  const objectKey = searchParams.get('objectKey')

  return getImgResponse(request, {
    headers,
    allowlistedOrigins: [
      getDomainUrl(request),
      process.env.AWS_ENDPOINT_URL_S3,
    ].filter(Boolean),
    cacheFolder: await getCacheDir(),  // 本地缓存目录
    getImgSource: () => {
      if (objectKey) {
        // 为 objectKey 生成签名 GET 请求
        const { url: signedUrl, headers: signedHeaders } =
          getSignedGetRequestInfo(objectKey)
        return {
          type: 'fetch',
          url: signedUrl,
          headers: signedHeaders,
        }
      }
      // ... 其他来源（本地文件系统等）
    },
  })
}
```

#### 步骤 3：签名 GET 请求生成

**文件**：[app/utils/storage.server.ts](app/utils/storage.server.ts:169-179)

与 PUT 请求类似，GET 请求也使用 AWS4 签名：

```typescript
export function getSignedGetRequestInfo(key: string) {
  const { url, baseHeaders } = getBaseSignedRequestInfo({
    method: 'GET',
    key,
  })

  return {
    url,
    headers: baseHeaders,
  }
}
```

### 4.3 缓存策略

#### 服务端缓存
**文件**：[app/routes/resources/images.tsx](app/routes/resources/images.tsx:10-26)

项目使用 `openimg` 库实现**本地文件缓存**：

```typescript
let cacheDir: string | null = null

async function getCacheDir() {
  if (cacheDir) return cacheDir

  let dir = './tests/fixtures/openimg'  // 开发环境
  if (process.env.NODE_ENV === 'production') {
    const isAccessible = await fs
      .access('/data', constants.W_OK)
      .then(() => true)
      .catch(() => false)

    if (isAccessible) {
      dir = '/data/images'  // 生产环境
    }
  }

  return (cacheDir = dir)
}
```

#### 客户端缓存
**文件**：[app/routes/resources/images.tsx](app/routes/resources/images.tsx:33)

响应头设置**强缓存**：
```typescript
headers.set('Cache-Control', 'public, max-age=31536000, immutable')
```

- `max-age=31536000`：缓存 1 年
- `immutable`：不验证直接使用缓存
- `public`：允许 CDN 和中间代理缓存

---

## 5. 鉴权方式设计分析

### 5.1 当前实现状态

#### 架构特点

**服务端代理模式**：所有图片访问都经过 Remix 的资源路由，这为鉴权提供了**天然的控制点**。

```
直接访问（当前不支持）:
浏览器 ──▶ https://fly.storage.tigris.dev/bucket/users/xxx/... ❌

实际访问路径:
浏览器 ──▶ /resources/images?objectKey=... ──▶ 服务端签名 ──▶ Tigris ✅
```

#### 当前鉴权机制

| 层面 | 机制 | 说明 |
|-----|------|------|
| **存储层** | AWS4 签名 | 只有持有 `AWS_SECRET_ACCESS_KEY` 的服务端能直接访问 Tigris |
| **应用层** | （暂无） | 当前资源路由未实现基于 objectKey 的权限检查 |
| **数据层** | 所有权关联 | 数据库中 `UserImage.userId` 和 `NoteImage.noteId` 建立了所有权关系 |

### 5.2 潜在的鉴权扩展能力

基于当前架构，项目可以轻松扩展私有/公开资产鉴权：

#### 方案 A：基于路径的权限控制

利用 objectKey 的层级结构：

```typescript
// users/{userId}/profile-images/...   - 公开（头像）
// users/{userId}/notes/{noteId}/images/... - 需检查笔记权限

async function checkImageAccess(objectKey: string, userId: string | null) {
  const parts = objectKey.split('/')
  
  if (parts[0] === 'users') {
    const ownerId = parts[1]
    
    if (parts[2] === 'profile-images') {
      // 头像：公开访问
      return { allowed: true, visibility: 'public' }
    }
    
    if (parts[2] === 'notes') {
      const noteId = parts[3]
      // 检查笔记权限
      const note = await prisma.note.findUnique({
        where: { id: noteId },
        select: { ownerId: true, /* visibility: true */ },
      })
      
      if (!note) return { allowed: false }
      
      // 示例：如果笔记是公开的，或请求者是所有者
      if (userId === note.ownerId /* || note.visibility === 'public' */) {
        return { allowed: true, visibility: 'private' }
      }
    }
  }
  
  return { allowed: false }
}
```

#### 方案 B：基于数据库元数据的权限控制

扩展数据库模型添加可见性字段：

```prisma
// 扩展后的模型示例
model NoteImage {
  id         String  @id @default(cuid())
  altText    String?
  objectKey  String
  visibility String  @default("private")  // "private" | "public" | "unlisted"
  
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt
  
  note       Note    @relation(fields: [noteId], references: [id])
  noteId     String
}
```

#### 方案 C：预签名 URL 模式（客户端直传）

当前实现是**服务端直传**，也可以改为**客户端直传**模式：

```
当前模式：
浏览器 ──▶ 服务端 ──▶ Tigris
           (签名+上传)

客户端直传模式：
浏览器 ──▶ 服务端（获取签名URL）
   │
   └───────▶ Tigris（使用签名URL直传）
```

### 5.3 鉴权设计建议

#### 推荐实现步骤

1. **第一阶段：基础权限检查**
   ```typescript
   // 在 resources/images.tsx 中添加权限检查
   export async function loader({ request }: Route.LoaderArgs) {
     const userId = await optionalUserId(request)  // 获取当前用户（可选）
     const objectKey = searchParams.get('objectKey')
     
     if (objectKey) {
       // 检查权限
       const access = await checkImageAccess(objectKey, userId)
       if (!access.allowed) {
         throw new Response(null, { status: 403 })
       }
       
       // 继续处理...
     }
   }
   ```

2. **第二阶段：区分公开/私有资产**
   - 头像：默认公开（便于社交展示）
   - 笔记图片：继承笔记的可见性设置

3. **第三阶段：缓存策略优化**
   - 公开资产：使用 `public` 缓存指令
   - 私有资产：使用 `private` 缓存指令，或不缓存

---

## 6. 关键文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `app/utils/storage.server.ts` | 核心存储服务：签名算法、上传函数 |
| `app/routes/settings/profile/photo.tsx` | 头像上传页面与服务端 Action |
| `app/routes/resources/images.tsx` | 图片资源路由（服务端代理） |
| `app/utils/misc.tsx` | 前端 URL 生成辅助函数 |
| `app/routes/users/$username/notes/+shared/note-editor.server.tsx` | 笔记图片上传处理 |
| `prisma/schema.prisma` | 数据库模型定义 |
| `docs/decisions/040-tigris-image-storage.md` | 存储架构决策文档 |
| `docs/image-storage.md` | 官方存储说明文档 |

---

## 7. 总结

### 架构优势

1. **安全性**：
   - AWS4 签名算法确保只有服务端能直接访问存储
   - 服务端代理模式为鉴权提供了统一控制点
   - 密钥仅存储在服务端环境变量中

2. **可扩展性**：
   - 层级化的 objectKey 命名天然支持权限隔离
   - 服务端代理架构便于添加任意鉴权逻辑
   - 数据库元数据可以轻松扩展可见性字段

3. **性能**：
   - 服务端直传避免了浏览器的跨域限制
   - 多层缓存策略（服务端文件缓存 + 客户端 HTTP 缓存）
   - 流式上传减少内存占用

### 当前局限

1. **缺少明确的公开/私有区分**：当前所有图片通过同一代理访问，没有权限检查
2. **服务端直传**：文件经过服务端中转，在高并发场景可能成为瓶颈
3. **缺少 URL 过期机制**：当前的签名是即时生成的，没有实现预签名 URL 的过期时间

### 扩展建议

1. **实现基于路径的权限检查**：利用 objectKey 的层级结构快速判断访问权限
2. **考虑客户端直传模式**：对于大文件上传，可实现服务端生成预签名 URL，客户端直接上传
3. **添加可见性字段**：在数据库模型中添加 `visibility` 字段支持细粒度控制
4. **实现 CDN 集成**：公开资产可通过 CDN 分发，进一步提升访问速度

---

## 附录：完整流程图

### 上传流程图
```
用户操作                    浏览器                     服务端                     Tigris/DB
   │                         │                          │                          │
   │  1. 选择图片文件         │                          │                          │
   │────────────────────────▶│                          │                          │
   │                         │  FileReader 预览         │                          │
   │                         │─────────────────────▶    │                          │
   │                         │                          │                          │
   │  2. 点击保存             │                          │                          │
   │────────────────────────▶│                          │                          │
   │                         │  POST /settings/profile  │                          │
   │                         │  multipart/form-data     │                          │
   │                         │─────────────────────────▶│                          │
   │                         │                          │  requireUserId() 认证    │
   │                         │                          │─────────────────────▶    │
   │                         │                          │                          │
   │                         │                          │  parseFormData() 解析    │
   │                         │                          │  Zod schema 验证         │
   │                         │                          │─────────────────────▶    │
   │                         │                          │                          │
   │                         │                          │  uploadProfileImage()    │
   │                         │                          │  ├─ 生成 objectKey       │
   │                         │                          │  ├─ AWS4 签名 PUT 请求   │
   │                         │                          │  └─ fetch 直传 Tigris   │
   │                         │                          │─────────────────────────▶│
   │                         │                          │                          │  接收文件
   │                         │                          │                          │──────────▶
   │                         │                          │                          │
   │                         │                          │◀─────────────────────────│
   │                         │                          │  返回 objectKey          │
   │                         │                          │                          │
   │                         │                          │  Prisma 事务             │
   │                         │                          │  ├─ 删除旧图片记录       │
   │                         │                          │  └─ 创建新图片记录       │
   │                         │                          │─────────────────────────▶│
   │                         │                          │                          │
   │                         │  302 Redirect            │                          │
   │                         │◀─────────────────────────│                          │
   │◀────────────────────────│                          │                          │
   │  3. 跳转到资料页面       │                          │                          │
   ▼                         ▼                          ▼                          ▼
```

### 访问流程图
```
用户操作                    浏览器                     服务端代理                  Tigris/缓存
   │                         │                          │                          │
   │  1. 访问页面             │                          │                          │
   │────────────────────────▶│                          │                          │
   │                         │  渲染 <img src="...">    │                          │
   │                         │─────────────────────▶    │                          │
   │                         │                          │                          │
   │                         │  GET /resources/images    │                          │
   │                         │  ?objectKey=users/...     │                          │
   │                         │─────────────────────────▶│                          │
   │                         │                          │  检查本地缓存             │
   │                         │                          │─────────────────────▶    │
   │                         │                          │                          │
   │                         │                          │  缓存未命中               │
   │                         │                          │─────────────────────▶    │
   │                         │                          │                          │
   │                         │                          │  getSignedGetRequestInfo()│
   │                         │                          │  ├─ 构建规范请求          │
   │                         │                          │  ├─ 计算 AWS4 签名        │
   │                         │                          │  └─ 生成 Authorization 头 │
   │                         │                          │                          │
   │                         │                          │  fetch(signedUrl)         │
   │                         │                          │─────────────────────────▶│
   │                         │                          │                          │  验证签名
   │                         │                          │                          │──────────▶
   │                         │                          │                          │
   │                         │                          │◀─────────────────────────│
   │                         │                          │  返回图片数据             │
   │                         │                          │                          │
   │                         │                          │  写入本地缓存             │
   │                         │                          │─────────────────────▶    │
   │                         │                          │                          │
   │                         │  200 OK                   │                          │
   │                         │  Cache-Control: public    │                          │
   │                         │  Content-Type: image/jpeg │                          │
   │                         │◀─────────────────────────│                          │
   │◀────────────────────────│                          │                          │
   │  2. 显示图片             │                          │                          │
   ▼                         ▼                          ▼                          ▼
```
