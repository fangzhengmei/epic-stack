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

## 5. 笔记图片完整 CRUD 流程分析

### 5.1 前端表单处理

**文件**：[app/routes/users/$username/notes/+shared/note-editor.tsx](app/routes/users/$username/notes/+shared/note-editor.tsx:1-294)

笔记编辑器支持**动态图片列表**，用户可以添加、删除、替换图片：

#### 表单结构设计

```typescript
export const NoteEditorSchema = z.object({
  id: z.string().optional(),
  title: z.string().min(1).max(100),
  content: z.string().min(1).max(10000),
  images: z.array(ImageFieldsetSchema).max(5).optional(),  // 最多 5 张图片
})

const ImageFieldsetSchema = z.object({
  id: z.string().optional(),        // 已有图片的数据库 ID
  file: z.instanceof(File).optional(),  // 新上传的文件
  altText: z.string().optional(),        // 替代文本
})
```

#### 前端交互流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      笔记图片前端交互流程                          │
└─────────────────────────────────────────────────────────────────┘

  初始状态：
  ┌─────────┐     ┌─────────┐     ┌─────────┐
  │ 图片 1  │     │ 图片 2  │     │ [添加]  │
  │(已有ID) │     │(已有ID) │     │  按钮   │
  └─────────┘     └─────────┘     └─────────┘
       ▲               ▲
       │               │
   点击图片区域      点击 [×] 按钮
   选择新文件        移除该图片
       │               │
       ▼               ▼
  ┌─────────┐     ┌─────────┐
  │ 替换为  │     │  消失   │
  │ 新文件  │     │         │
  └─────────┘     └─────────┘
```

#### 关键交互代码

```tsx
// 动态图片列表管理
const imageList = fields.images.getFieldList()

{imageList.map((imageMeta, index) => {
  const imageMetaId = imageMeta.getFieldset().id.value
  const image = note?.images.find(({ id }) => id === imageMetaId)
  
  return (
    <li key={imageMeta.key}>
      {/* 删除按钮 */}
      <button {...form.remove.getButtonProps({
        name: fields.images.name,
        index,
      })}>
        <Icon name="cross-1" />
      </button>
      
      {/* 图片选择器 */}
      <ImageChooser meta={imageMeta} objectKey={image?.objectKey} />
    </li>
  )
})}

// 添加新图片按钮
<Button {...form.insert.getButtonProps({ name: fields.images.name })}>
  <Icon name="plus">Image</Icon>
</Button>
```

### 5.2 服务端完整处理逻辑

**文件**：[app/routes/users/$username/notes/+shared/note-editor.server.tsx](app/routes/users/$username/notes/+shared/note-editor.server.tsx:1-131)

服务端处理图片的**新增、替换、删除**三种场景：

#### 类型谓词函数

```typescript
// 判断是否有文件上传（新增或替换）
function imageHasFile(
  image: ImageFieldset,
): image is ImageFieldset & { file: NonNullable<ImageFieldset['file']> } {
  return Boolean(image.file?.size && image.file?.size > 0)
}

// 判断是否是已有图片（有数据库 ID）
function imageHasId(
  image: ImageFieldset,
): image is ImageFieldset & { id: string } {
  return Boolean(image.id)
}
```

#### 数据转换逻辑

```typescript
transform(async ({ images = [], ...data }) => {
  const noteId = data.id ?? cuid()
  return {
    ...data,
    id: noteId,
    
    // 已有图片的更新（可能包含替换）
    imageUpdates: await Promise.all(
      images.filter(imageHasId).map(async (i) => {
        if (imageHasFile(i)) {
          // 有新文件：替换场景 → 上传新文件，生成新 objectKey
          return {
            id: i.id,
            altText: i.altText,
            objectKey: await uploadNoteImage(userId, noteId, i.file),
          }
        } else {
          // 无新文件：仅更新 altText
          return {
            id: i.id,
            altText: i.altText,
          }
        }
      }),
    ),
    
    // 新增图片（无 ID 但有文件）
    newImages: await Promise.all(
      images
        .filter(imageHasFile)
        .filter((i) => !i.id)  // 无 ID = 新增
        .map(async (image) => {
          return {
            altText: image.altText,
            objectKey: await uploadNoteImage(userId, noteId, image.file),
          }
        }),
    ),
  }
})
```

#### 数据库操作（Prisma 事务）

```typescript
const updatedNote = await prisma.note.upsert({
  where: { id: noteId },
  create: {
    id: noteId,
    ownerId: userId,
    title,
    content,
    images: { create: newImages },  // 新增图片
  },
  update: {
    title,
    content,
    images: {
      // 删除：不在提交列表中的图片
      deleteMany: { id: { notIn: imageUpdates.map((i) => i.id) } },
      
      // 更新：已有图片的更新（可能包含替换）
      updateMany: imageUpdates.map((updates) => ({
        where: { id: updates.id },
        data: {
          ...updates,
          // 关键：如果有新文件（新 objectKey），生成新的数据库 ID
          // 用于缓存失效（cache busting）
          id: updates.objectKey ? cuid() : updates.id,
        },
      })),
      
      // 新增：新添加的图片
      create: newImages,
    },
  },
})
```

### 5.3 三种场景的完整流程

#### 场景 1：新增图片

```
┌─────────────────────────────────────────────────────────────────────┐
│                        新增图片流程                                   │
└─────────────────────────────────────────────────────────────────────┘

  前端                              服务端                            数据库
   │                                  │                                 │
   │  点击 [Add Image]               │                                 │
   │────────────────────────────────▶│                                 │
   │                                  │                                 │
   │  选择文件                         │                                 │
   │  本地预览                         │                                 │
   │  提交表单                         │                                 │
   │────────────────────────────────▶│                                 │
   │                                  │                                 │
   │                                  │  检测：无 id，有 file           │
   │                                  │  ├─ uploadNoteImage()          │
   │                                  │  │  ├─ 生成 objectKey          │
   │                                  │  │  └─ 上传到 Tigris          │
   │                                  │  └─ 加入 newImages 数组       │
   │                                  │────────────────────────────────▶│
   │                                  │                                 │  prisma.note.upsert()
   │                                  │                                 │  images: { create: newImages }
   │                                  │                                 │  └─ 插入新 NoteImage 记录
   │                                  │◀────────────────────────────────│
   │  重定向到笔记详情                │                                 │
   │◀────────────────────────────────│                                 │
   ▼                                  ▼                                 ▼
```

#### 场景 2：替换图片（更新）

```
┌─────────────────────────────────────────────────────────────────────┐
│                        替换图片流程                                   │
└─────────────────────────────────────────────────────────────────────┘

  前端                              服务端                            数据库
   │                                  │                                 │
   │  点击已有图片区域                 │                                 │
   │  选择新文件                       │                                 │
   │  提交表单                         │                                 │
   │────────────────────────────────▶│                                 │
   │                                  │                                 │
   │                                  │  检测：有 id，有 file           │
   │                                  │  ├─ uploadNoteImage()          │
   │                                  │  │  ├─ 生成 新的 objectKey     │
   │                                  │  │  └─ 上传到 Tigris          │
   │                                  │  └─ 加入 imageUpdates 数组    │
   │                                  │                                 │
   │                                  │  关键：有新 objectKey →        │
   │                                  │  生成 新的数据库 ID            │
   │                                  │                                 │
   │                                  │────────────────────────────────▶│
   │                                  │                                 │  prisma.note.upsert()
   │                                  │                                 │  updateMany: [
   │                                  │                                 │    {
   │                                  │                                 │      where: { id: 旧ID },
   │                                  │                                 │      data: {
   │                                  │                                 │        objectKey: 新Key,
   │                                  │                                 │        id: 新ID,  // ← 缓存失效
   │                                  │                                 │      }
   │                                  │                                 │    }
   │                                  │                                 │  ]
   │                                  │◀────────────────────────────────│
   │  重定向到笔记详情                │                                 │
   │◀────────────────────────────────│                                 │
   │                                  │                                 │
   │  关键点：                         │                                 │
   │  1. URL 变化：新 objectKey       │                                 │
   │  2. 数据库 ID 变化：新 cuid()    │                                 │
   │  两者共同确保缓存失效             │                                 │
   ▼                                  ▼                                 ▼
```

#### 场景 3：删除图片

```
┌─────────────────────────────────────────────────────────────────────┐
│                        删除图片流程                                   │
└─────────────────────────────────────────────────────────────────────┘

  前端                              服务端                            数据库
   │                                  │                                 │
   │  点击 [×] 移除按钮               │                                 │
   │────────────────────────────────▶│                                 │
   │                                  │                                 │
   │  提交表单                         │                                 │
   │────────────────────────────────▶│                                 │
   │                                  │                                 │
   │                                  │  检测：该图片不在 imageUpdates  │
   │                                  │  列表中（因为被移除了）          │
   │                                  │                                 │
   │                                  │────────────────────────────────▶│
   │                                  │                                 │  prisma.note.upsert()
   │                                  │                                 │  deleteMany: {
   │                                  │                                 │    id: { 
   │                                  │                                 │      notIn: imageUpdates.map(
   │                                  │                                 │        (i) => i.id
   │                                  │                                 │      ) 
   │                                  │                                 │    }
   │                                  │                                 │  }
   │                                  │                                 │
   │                                  │                                 │  注意：Tigris 中的旧文件
   │                                  │                                 │  不会被自动删除！
   │                                  │◀────────────────────────────────│
   │  重定向到笔记详情                │                                 │
   │◀────────────────────────────────│                                 │
   ▼                                  ▼                                 ▼
```

### 5.4 潜在问题：存储泄漏

**重要发现**：当前实现中，**删除和替换操作不会删除 Tigris 中的旧文件**。

```typescript
// 替换场景：只更新了数据库记录，旧的 objectKey 对应的文件仍然存在
// 删除场景：只删除了数据库记录，Tigris 中的文件仍然存在

// 这可能导致：
// 1. 存储成本浪费：无用文件继续占用存储空间
// 2. 安全风险：如果旧文件的 URL 泄露，仍可被访问
// 3. 合规问题：用户删除的数据可能未真正删除
```

**建议的修复方案**：
```typescript
// 在更新或删除前，获取旧的 objectKey 并调用删除 API
async function cleanupOldImages(oldObjectKeys: string[]) {
  for (const key of oldObjectKeys) {
    // 调用 Tigris/S3 的删除 API
    const { url, headers } = getSignedDeleteRequestInfo(key)
    await fetch(url, { method: 'DELETE', headers })
  }
}
```

---

## 6. 缓存失效机制深度分析

### 6.1 缓存策略回顾

**文件**：[app/routes/resources/images.tsx](app/routes/resources/images.tsx:33)

当前使用的是**强缓存 + immutable** 策略：

```typescript
headers.set('Cache-Control', 'public, max-age=31536000, immutable')
```

| 指令 | 含义 | 影响 |
|-----|------|------|
| `public` | 允许所有中间代理（CDN、浏览器）缓存 | 可被多级缓存 |
| `max-age=31536000` | 缓存有效期 1 年 | 长期缓存 |
| `immutable` | 缓存内容不会改变，无需重新验证 | **关键特性** |

### 6.2 Immutable 缓存的特性

**immutable** 是一个特殊的缓存指令，它告诉浏览器：

> "这个资源在 max-age 期间绝对不会改变，不要发送 304 验证请求，直接使用缓存。"

这意味着：

```
传统缓存（无 immutable）：
浏览器 ── If-None-Match/If-Modified-Since ──▶ 服务器
         ◀────── 304 Not Modified（或 200）──────
         （每次都需要验证）

immutable 缓存：
浏览器 ──────────────────▶ 直接使用缓存
         （不发送任何请求，直到 max-age 过期）
```

### 6.3 唯一的缓存失效方式：改变 URL

由于使用了 `immutable`，**唯一的缓存失效方式是改变资源的 URL**。

这就是为什么 Epic Stack 采用了以下设计：

#### 1. ObjectKey 包含时间戳和随机 ID

**文件**：[app/utils/storage.server.ts](app/utils/storage.server.ts:40-50)

```typescript
export async function uploadNoteImage(
  userId: string,
  noteId: string,
  file: File | FileUpload,
) {
  const fileId = createId()        // 随机 CUID
  const timestamp = Date.now()     // 时间戳
  const key = `users/${userId}/notes/${noteId}/images/${timestamp}-${fileId}.${fileExtension}`
  // 示例: users/xxx/notes/yyy/images/1714900000000-abc123.png
  //       每次上传都是新的 key
}
```

**效果**：
- 每次上传都是**全新的 URL**
- 新 URL 没有被缓存过，浏览器会重新请求
- 旧 URL 的缓存仍然存在，但已不再被使用

#### 2. 数据库 ID 也会更新（双重保险）

**文件**：[app/routes/users/$username/notes/+shared/note-editor.server.tsx](app/routes/users/$username/notes/+shared/note-editor.server.tsx:119-120)

```typescript
updateMany: imageUpdates.map((updates) => ({
  where: { id: updates.id },
  data: {
    ...updates,
    // If the image is new, we need to generate a new ID to bust the cache.
    id: updates.objectKey ? cuid() : updates.id,
  },
})),
```

注释明确说明了目的：**"bust the cache"（破坏缓存）**。

**为什么需要更新数据库 ID？**

查看前端渲染逻辑：

**文件**：[app/routes/users/$username/notes/$noteId.tsx](app/routes/users/$username/notes/$noteId.tsx:126-138)

```tsx
{loaderData.note.images.map((image) => (
  <li key={image.id}>
    <a href={getNoteImgSrc(image.objectKey)}>
      <Img
        src={getNoteImgSrc(image.objectKey)}
        alt={image.altText ?? ''}
        // ...
      />
    </a>
  </li>
))}
```

- **React `key`**：使用 `image.id`
- **图片 `src`**：使用 `image.objectKey`

当图片被替换时：
1. `objectKey` 改变 → `src` 改变 → **HTTP 缓存失效**
2. `id` 改变 → React `key` 改变 → **组件强制重新渲染**

这是**双重保险**机制，确保：
- 浏览器层面：获取新图片（因为 URL 变了）
- React 层面：重新渲染组件（因为 key 变了）

### 6.4 缓存失效流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          缓存失效完整流程                                  │
└─────────────────────────────────────────────────────────────────────────┘

  场景：用户替换笔记中的一张图片

  ┌─────────────────────────────────────────────────────────────────────┐
  │ 第 1 次访问（原图）                                                    │
  └─────────────────────────────────────────────────────────────────────┘
  
  浏览器                          资源路由                    Tigris/缓存
     │                               │                           │
     │  GET /resources/images?       │                           │
     │      objectKey=old-key.png    │                           │
     │──────────────────────────────▶│                           │
     │                               │  检查本地缓存               │
     │                               │──────────────────────────▶│
     │                               │                           │  缓存未命中
     │                               │◀──────────────────────────│
     │                               │                           │
     │                               │  getSignedGetRequestInfo() │
     │                               │  fetch(signedUrl)          │
     │                               │──────────────────────────▶│
     │                               │                           │
     │                               │◀──────────────────────────│
     │                               │                           │
     │  200 OK                       │                           │
     │  Cache-Control: public,      │                           │
     │      max-age=31536000,       │                           │
     │      immutable                │                           │
     │◀──────────────────────────────│                           │
     │                               │                           │
     │  浏览器缓存：                  │                           │
     │  URL: old-key.png             │                           │
     │  有效期：1 年                  │                           │
     │  immutable：永不验证          │                           │
     ▼                               ▼                           ▼


  ┌─────────────────────────────────────────────────────────────────────┐
  │ 第 2 次访问（图片已替换，但使用旧 URL）                                 │
  └─────────────────────────────────────────────────────────────────────┘
  
  浏览器                          资源路由                    Tigris/缓存
     │                               │                           │
     │  GET /resources/images?       │                           │
     │      objectKey=old-key.png    │                           │
     │                               │                           │
     │  ┌─────────────────────┐      │                           │
     │  │ immutable 缓存生效   │      │                           │
     │  │ 直接使用本地缓存     │      │                           │
     │  │ 不发送任何请求       │      │                           │
     │  └─────────────────────┘      │                           │
     │                               │                           │
     │  结果：显示的还是旧图片！      │                           │
     │  （因为 immutable 缓存不会验证）│                           │
     ▼                               ▼                           ▼


  ┌─────────────────────────────────────────────────────────────────────┐
  │ 正确的缓存失效方式：使用新 URL                                        │
  └─────────────────────────────────────────────────────────────────────┘
  
  前端渲染逻辑：
  - 替换图片后，数据库记录有新的 objectKey
  - getNoteImgSrc(newObjectKey) 生成新 URL
  
  浏览器                          资源路由                    Tigris/缓存
     │                               │                           │
     │  GET /resources/images?       │                           │
     │      objectKey=new-key.png    │  ← 新的 URL！            │
     │──────────────────────────────▶│                           │
     │                               │                           │
     │  ┌─────────────────────┐      │                           │
     │  │ 这个 URL 从未缓存过  │      │                           │
     │  │ 必须重新请求         │      │                           │
     │  └─────────────────────┘      │                           │
     │                               │                           │
     │                               │  fetch(signedUrl)          │
     │                               │──────────────────────────▶│
     │                               │                           │
     │                               │◀──────────────────────────│
     │                               │                           │
     │  200 OK（新图片）             │                           │
     │  同样带 immutable 缓存头      │                           │
     │◀──────────────────────────────│                           │
     │                               │                           │
     │  结果：显示新图片！            │                           │
     │  （因为 URL 变了，缓存失效）   │                           │
     ▼                               ▼                           ▼
```

### 6.5 缓存层架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            多层缓存架构                                   │
└─────────────────────────────────────────────────────────────────────────┘

  请求顺序（从快到慢）：

  1. 浏览器内存缓存
     └─ 最快，直接从内存读取
     └─ 受 immutable 策略影响

  2. 浏览器磁盘缓存
     └─ 次快，从本地磁盘读取
     └─ 同样受 immutable 策略影响

  3. CDN 缓存（如果有）
     └─ 从边缘节点获取
     └─ 受 public 指令影响

  4. 服务端文件缓存（openimg）
     └─ [app/routes/resources/images.tsx]
     └─ cacheFolder: ./tests/fixtures/openimg 或 /data/images
     └─ 基于 objectKey 的文件缓存

  5. Tigris 原始存储
     └─ 最慢，但总是最新的
     └─ 需要 AWS4 签名认证

  缓存失效点：
  ─────────────────────────────────────────────────────────────────────
  改变 URL（objectKey）→ 所有层缓存同时失效
  因为新 URL 在任何缓存层都不存在
```

---

## 7. 公开访问与私有资源的边界设计

### 7.1 当前状态：所有图片都是公开可访问的

#### 证据 1：资源路由无权限检查

**文件**：[app/routes/resources/images.tsx](app/routes/resources/images.tsx:28-80)

```typescript
export async function loader({ request }: Route.LoaderArgs) {
  const url = new URL(request.url)
  const searchParams = url.searchParams
  
  // 没有任何用户认证检查！
  // 没有 requireUserId()
  // 没有权限验证
  
  const objectKey = searchParams.get('objectKey')
  
  return getImgResponse(request, {
    // ...
    getImgSource: () => {
      if (objectKey) {
        // 直接生成签名请求，不检查权限
        const { url: signedUrl, headers: signedHeaders } =
          getSignedGetRequestInfo(objectKey)
        return {
          type: 'fetch',
          url: signedUrl,
          headers: signedHeaders,
        }
      }
      // ...
    },
  })
}
```

#### 证据 2：笔记页面也无图片权限检查

**文件**：[app/routes/users/$username/notes/$noteId.tsx](app/routes/users/$username/notes/$noteId.tsx:24-49)

```typescript
export async function loader({ params }: Route.LoaderArgs) {
  // 直接查询笔记，不检查当前用户是否有权访问
  const note = await prisma.note.findUnique({
    where: { id: params.noteId },
    select: {
      id: true,
      title: true,
      content: true,
      ownerId: true,
      updatedAt: true,
      images: {
        select: {
          id: true,
          altText: true,
          objectKey: true,  // 直接返回图片引用
        },
      },
    },
  })
  
  invariantResponse(note, 'Not found', { status: 404 })
  // 直接返回，不检查权限
  return { note, timeAgo }
}
```

#### 结论

**当前架构中，所有图片实际上都是公开的**：

```
任何人只要知道 objectKey，就可以访问图片：

https://your-app.com/resources/images?objectKey=users%2Fxxx%2Fprofile-images%2F...

不需要登录
不需要权限
不需要知道笔记 ID
只需要知道 objectKey 的值
```

### 7.2 路径结构的权限潜力

虽然当前没有实现权限控制，但 **objectKey 的层级结构天然支持权限隔离**：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ObjectKey 路径结构                                │
└─────────────────────────────────────────────────────────────────────────┘

  路径格式：
  users/{userId}/{resourceType}/{resourceId}/.../{timestamp}-{fileId}.{ext}

  示例分解：
  ┌─────────────────────────────────────────────────────────────────────┐
  │ users/clg23456.../profile-images/1714900000000-abc123.jpg          │
  ├───────────────┬──────────────┬───────────────────────────────────────┤
  │    用户       │   资源类型    │           具体文件                     │
  │  (ownerId)    │              │                                      │
  └───────────────┴──────────────┴───────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────┐
  │ users/clg23456.../notes/note789.../images/1714900000000-def456.png │
  ├───────────────┬─────────┬─────────────┬──────────────────────────────┤
  │    用户       │  类型   │   笔记 ID   │         具体文件              │
  │               │         │             │                              │
  └───────────────┴─────────┴─────────────┴──────────────────────────────┘
```

**可提取的权限信息**：

| 路径部分 | 含义 | 权限用途 |
|---------|------|---------|
| `users/{userId}` | 资源所有者 ID | 确定谁拥有这个资源 |
| `profile-images` | 头像类型 | 头像通常是公开的 |
| `notes/{noteId}` | 笔记 ID | 可检查笔记的可见性设置 |
| `images` | 笔记图片类型 | 继承笔记的权限 |

### 7.3 数据库中的权限关联

数据库模型已经建立了**所有权关联**：

```prisma
model UserImage {
  id        String  @id @default(cuid())
  objectKey String
  
  userId    String  @unique  // ← 直接关联到用户
  user      User    @relation(fields: [userId], references: [id])
}

model NoteImage {
  id        String  @id @default(cuid())
  objectKey String
  
  noteId    String          // ← 关联到笔记
  note      Note    @relation(fields: [noteId], references: [id])
  // 通过 note 间接关联到 user: note.ownerId
}

model Note {
  id        String  @id @default(cuid())
  ownerId   String          // ← 笔记所有者
  owner     User    @relation(fields: [ownerId], references: [id])
}
```

**权限查询路径**：
- `UserImage.objectKey` → `UserImage.userId` → `User.id`（直接）
- `NoteImage.objectKey` → `NoteImage.noteId` → `Note.ownerId` → `User.id`（间接）

### 7.4 权限系统的扩展能力

项目已经有一套 **RBAC（基于角色的访问控制）** 系统：

**文件**：[app/utils/user.ts](app/utils/user.ts:28-62)

```typescript
// 权限字符串格式
type Action = 'create' | 'read' | 'update' | 'delete'
type Entity = 'user' | 'note'
type Access = 'own' | 'any'
export type PermissionString =
  | `${Action}:${Entity}`
  | `${Action}:${Entity}:${Access}`

// 示例权限
// 'read:note:own'  - 读取自己的笔记
// 'read:note:any'  - 读取任意笔记
// 'delete:note:own' - 删除自己的笔记
```

**当前权限检查示例**：

**文件**：[app/utils/permissions.server.ts](app/utils/permissions.server.ts:6-41)

```typescript
export async function requireUserWithPermission(
  request: Request,
  permission: PermissionString,
) {
  const userId = await requireUserId(request)
  const permissionData = parsePermissionString(permission)
  
  const user = await prisma.user.findFirst({
    select: { id: true },
    where: {
      id: userId,
      roles: {
        some: {
          permissions: {
            some: {
              ...permissionData,
              access: permissionData.access
                ? { in: permissionData.access }
                : undefined,
            },
          },
        },
      },
    },
  })
  
  if (!user) {
    throw data({ error: 'Unauthorized' }, { status: 403 })
  }
  return user.id
}
```

### 7.5 建议的权限实现方案

基于现有架构，建议采用**三层权限检查**：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      建议的权限检查流程                                   │
└─────────────────────────────────────────────────────────────────────────┘

  请求：GET /resources/images?objectKey=users/xxx/notes/yyy/images/...
  
                              │
                              ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ 第 1 层：路径解析（快速判断资源类型）                                   │
  └─────────────────────────────────────────────────────────────────────┘
                              │
                              ├── 路径以 users/{userId}/profile-images/ 开头
                              │   └─ 头像 → 公开访问，无需检查
                              │
                              ├── 路径以 users/{userId}/notes/{noteId}/ 开头
                              │   └─ 笔记图片 → 继续检查
                              │
                              └── 其他路径
                                  └─ 拒绝访问（或根据业务规则）
                              │
                              ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ 第 2 层：所有权验证                                                    │
  └─────────────────────────────────────────────────────────────────────┘
                              │
                              ├── 从路径提取 ownerId 和 noteId
                              │
                              ├── 查询数据库确认资源存在
                              │   └─ SELECT * FROM NoteImage WHERE noteId = ?
                              │
                              └── 确认路径中的 ownerId 与数据库一致
                                  （防止路径欺骗攻击）
                              │
                              ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ 第 3 层：业务规则判断                                                  │
  └─────────────────────────────────────────────────────────────────────┘
                              │
                              ├── 方案 A：笔记可见性字段（推荐扩展）
                              │   ├─ 扩展 Note 模型添加 visibility 字段
                              │   ├─ 'public' → 所有人可访问
                              │   ├─ 'private' → 仅所有者可访问
                              │   └─ 'unlisted' → 知道链接即可访问
                              │
                              ├── 方案 B：继承笔记权限（使用现有 RBAC）
                              │   ├─ 当前用户是所有者？
                              │   ├─ 或者用户有 'read:note:any' 权限？
                              │   └─ 或者其他业务规则？
                              │
                              └── 方案 C：资源级别权限
                                  ├─ 扩展 NoteImage 添加 visibility 字段
                                  └─ 独立控制每张图片的可见性
                              │
                              ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ 权限决策                                                               │
  └─────────────────────────────────────────────────────────────────────┘
                              │
                              ├── 允许访问 → 继续处理，返回图片
                              │
                              └── 拒绝访问 → 返回 403 Forbidden
```

### 7.6 具体实现示例

#### 示例 1：基于路径的快速权限检查

```typescript
// app/routes/resources/images.tsx 扩展

type ResourceType = 'profile-image' | 'note-image' | 'unknown'

interface ParsedObjectKey {
  type: ResourceType
  ownerId?: string
  noteId?: string
  original: string
}

function parseObjectKey(objectKey: string): ParsedObjectKey {
  const parts = objectKey.split('/')
  
  if (parts.length >= 4 && 
      parts[0] === 'users' && 
      parts[2] === 'profile-images') {
    return {
      type: 'profile-image',
      ownerId: parts[1],
      original: objectKey,
    }
  }
  
  if (parts.length >= 6 && 
      parts[0] === 'users' && 
      parts[2] === 'notes' &&
      parts[4] === 'images') {
    return {
      type: 'note-image',
      ownerId: parts[1],
      noteId: parts[3],
      original: objectKey,
    }
  }
  
  return { type: 'unknown', original: objectKey }
}
```

#### 示例 2：笔记图片权限检查

```typescript
// app/routes/resources/images.tsx 扩展

import { optionalUserId } from '#app/utils/auth.server.ts'
import { prisma } from '#app/utils/db.server.ts'

async function checkNoteImageAccess(
  noteId: string,
  pathOwnerId: string,
  currentUserId: string | null,
): Promise<{ allowed: boolean; cacheControl: string }> {
  // 1. 查询笔记及其图片
  const note = await prisma.note.findUnique({
    where: { id: noteId },
    select: {
      ownerId: true,
      // visibility: true, // 未来可扩展
    },
  })
  
  if (!note) {
    return { allowed: false, cacheControl: '' }
  }
  
  // 2. 防止路径欺骗：确认路径中的 ownerId 与数据库一致
  if (note.ownerId !== pathOwnerId) {
    return { allowed: false, cacheControl: '' }
  }
  
  // 3. 判断权限
  const isOwner = currentUserId === note.ownerId
  
  // 方案：当前实现为"所有人可读"（与笔记页面一致）
  // 未来可扩展为：
  // if (note.visibility === 'private' && !isOwner) {
  //   return { allowed: false, cacheControl: '' }
  // }
  
  // 4. 返回缓存策略建议
  if (isOwner) {
    // 所有者：使用 private 缓存
    return { allowed: true, cacheControl: 'private, max-age=86400' }
  } else {
    // 非所有者：如果是公开笔记，使用 public 缓存
    return { allowed: true, cacheControl: 'public, max-age=31536000, immutable' }
  }
}
```

#### 示例 3：完整的权限检查集成

```typescript
// 修改后的 loader
export async function loader({ request }: Route.LoaderArgs) {
  const url = new URL(request.url)
  const searchParams = url.searchParams
  const objectKey = searchParams.get('objectKey')
  
  if (!objectKey) {
    // 处理其他来源...
  }
  
  // 解析路径
  const parsed = parseObjectKey(objectKey)
  
  // 获取当前用户（可选）
  const currentUserId = await optionalUserId(request)
  
  // 根据资源类型判断权限
  let accessResult: { allowed: boolean; cacheControl: string }
  
  switch (parsed.type) {
    case 'profile-image':
      // 头像：公开访问
      accessResult = { 
        allowed: true, 
        cacheControl: 'public, max-age=31536000, immutable' 
      }
      break
      
    case 'note-image':
      // 笔记图片：检查权限
      accessResult = await checkNoteImageAccess(
        parsed.noteId!,
        parsed.ownerId!,
        currentUserId,
      )
      break
      
    default:
      // 未知类型：拒绝访问
      throw new Response(null, { status: 403 })
  }
  
  if (!accessResult.allowed) {
    throw new Response(null, { status: 403 })
  }
  
  // 使用权限检查返回的缓存策略
  const headers = new Headers()
  headers.set('Cache-Control', accessResult.cacheControl)
  
  // 继续处理图片获取...
  return getImgResponse(request, {
    headers,
    // ... 其他配置
  })
}
```

### 7.7 公开与私有边界总结

| 维度 | 当前状态 | 建议扩展 |
|-----|---------|---------|
| **访问控制** | 无，所有图片公开 | 基于路径解析 + 数据库验证 |
| **缓存策略** | 全部 `public, immutable` | 公开资产 `public`，私有资产 `private` |
| **资源类型** | 未区分 | 头像（公开）、笔记图片（继承笔记权限） |
| **路径结构** | 已包含权限信息 | 充分利用 `users/{id}/...` 结构 |
| **数据库关联** | 已有所有权字段 | 添加 `visibility` 字段支持细粒度控制 |

**关键设计决策**：
1. **服务端代理架构**是实现权限控制的基础
2. **层级化 objectKey** 天然支持权限隔离
3. **URL 改变 = 缓存失效** 策略与权限控制正交
4. **现有 RBAC 系统**可复用于资源权限检查

---

## 8. 鉴权方式设计分析

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
