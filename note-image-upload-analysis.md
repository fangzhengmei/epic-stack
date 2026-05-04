# 笔记图片上传流程分析

## 概述

本文档详细分析 Epic Stack 中笔记编辑时上传图片的完整流程，包括对象存储签名生成位置、客户端直传（实际为服务端中转）的协调机制。

## 架构概览

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────────────┐
│   客户端     │────▶│  应用服务器      │────▶│  对象存储 (Tigris/S3)│
│  (浏览器)    │     │  (签名+中转)     │     │                     │
└─────────────┘     └─────────────────┘     └─────────────────────┘
```

## 完整流程图

### 1. 前端选择与预览阶段

**文件位置**: `app/routes/users/$username/notes/+shared/note-editor.tsx`

#### 1.1 ImageChooser 组件

```
┌─────────────────────────────────────────────────────────────┐
│                    ImageChooser 组件流程                      │
├─────────────────────────────────────────────────────────────┤
│  1. 用户点击图片选择区域 (label + input type="file")          │
│     └─> line 194-255: 隐藏的 file input，opacity: 0         │
│                                                              │
│  2. 选择图片后的 onChange 事件                                │
│     └─> line 239-251:                                        │
│         - 获取 event.target.files[0]                        │
│         - 使用 FileReader.readAsDataURL() 读取文件           │
│         - onloadend 回调中 setPreviewImage(reader.result)   │
│         - 显示 data URL 格式的预览图                          │
└─────────────────────────────────────────────────────────────┘
```

#### 1.2 关键代码片段

```tsx
// note-editor.tsx:239-251
<input
  aria-label="Image"
  className="absolute top-0 left-0 z-0 size-32 cursor-pointer opacity-0"
  onChange={(event) => {
    const file = event.target.files?.[0]
    if (file) {
      const reader = new FileReader()
      reader.onloadend = () => {
        setPreviewImage(reader.result as string)
      }
      reader.readAsDataURL(file)
    } else {
      setPreviewImage(null)
    }
  }}
  accept="image/*"
  {...getInputProps(fields.file, { type: 'file' })}
  key={fields.file.key}
/>
```

### 2. 表单提交阶段

**文件位置**: `app/routes/users/$username/notes/+shared/note-editor.tsx:80-84`

```tsx
<Form
  method="POST"
  className="flex h-full flex-col gap-y-4 overflow-x-hidden overflow-y-auto px-10 pt-12 pb-28"
  {...getFormProps(form)}
  encType="multipart/form-data"  // 关键：multipart 编码支持文件上传
>
```

### 3. 服务端处理阶段

**文件位置**: `app/routes/users/$username/notes/+shared/note-editor.server.tsx`

#### 3.1 整体流程

```
┌────────────────────────────────────────────────────────────────────┐
│                    服务端 Action 处理流程                            │
├────────────────────────────────────────────────────────────────────┤
│  1. 解析表单数据                                                    │
│     └─> line 30-32: parseFormData(request, { maxFileSize })      │
│                                                              │
│  2. Zod Schema 验证与转换                                           │
│     └─> line 34-83: parseWithZod(formData, { schema })           │
│         - superRefine: 验证笔记是否存在且属于当前用户               │
│         - transform: 处理图片上传逻辑（核心）                       │
│                                                              │
│  3. 数据库操作                                                       │
│     └─> line 100-126: prisma.note.upsert                         │
│         - create: 创建新笔记，包含 newImages                       │
│         - update: 更新现有笔记，处理 imageUpdates 和 deleteMany    │
│                                                              │
│  4. 重定向                                                          │
│     └─> line 128-130: redirect 到笔记详情页                        │
└────────────────────────────────────────────────────────────────────┘
```

#### 3.2 图片上传处理逻辑（transform 阶段）

这是整个上传流程的核心，区分三种图片类型：

```tsx
// note-editor.server.tsx:48-81
.transform(async ({ images = [], ...data }) => {
  const noteId = data.id ?? cuid()
  return {
    ...data,
    id: noteId,
    // 已有图片的更新（有 id）
    imageUpdates: await Promise.all(
      images.filter(imageHasId).map(async (i) => {
        if (imageHasFile(i)) {
          // 已有图片且有新文件：需要重新上传
          return {
            id: i.id,
            altText: i.altText,
            objectKey: await uploadNoteImage(userId, noteId, i.file),
          }
        } else {
          // 已有图片但无新文件：只更新 altText
          return {
            id: i.id,
            altText: i.altText,
          }
        }
      }),
    ),
    // 新图片（无 id 但有文件）
    newImages: await Promise.all(
      images
        .filter(imageHasFile)
        .filter((i) => !i.id)
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

### 4. 存储层：签名生成与上传

**文件位置**: `app/utils/storage.server.ts`

#### 4.1 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                      storage.server.ts 架构                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  │ uploadNoteImage │───▶│ uploadToStorage │───▶│getSignedPutRequ │
│  │  (业务层)       │    │  (上传执行层)    │    │  estInfo        │
│  └─────────────────┘    └─────────────────┘    │  (签名生成层)    │
│                                                  └────────┬────────┘
│                                                           │
│                                                  ┌────────▼────────┐
│                                                  │getBaseSignedRequ│
│                                                  │  estInfo        │
│                                                  │ (AWS4 签名核心)  │
│                                                  └─────────────────┘
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 4.2 签名生成详细分析

**核心函数**: `getBaseSignedRequestInfo` (line 77-148)

这是 AWS Signature Version 4 的完整实现：

```
┌────────────────────────────────────────────────────────────────────┐
│                    AWS Signature Version 4 签名流程                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Step 1: 准备日期字符串 (line 92-93)                               │
│  ─────────────────────────────────                                  │
│  const amzDate = new Date().toISOString()                         │
│                  .replace(/[:-]|\.\d{3}/g, '')   // 如: 20260504T123456Z │
│  const dateStamp = amzDate.slice(0, 8)          // 如: 20260504   │
│                                                                     │
│  Step 2: 构建请求头 (line 96-102)                                  │
│  ──────────────────────────────                                    │
│  headers = [                                                        │
│    `content-type:${contentType}`,    // 可选                        │
│    `host:${endpoint.host}`,          // 必须                        │
│    `x-amz-content-sha256:UNSIGNED-PAYLOAD`,  // 不计算 payload hash │
│    `x-amz-date:${amzDate}`,          // 必须                        │
│    `x-amz-meta-upload-date:${uploadDate}`,  // 元数据               │
│  ]                                                                  │
│                                                                     │
│  Step 3: 构建 Canonical Request (line 107-114)                    │
│  ─────────────────────────────────────────────                     │
│  canonicalRequest = [                                               │
│    method,                         // PUT/GET                       │
│    `/${STORAGE_BUCKET}/${key}`,    // 资源路径                      │
│    '',                              // query string (空)             │
│    canonicalHeaders,                // 规范请求头 + \n               │
│    signedHeaders,                   // 已签名头列表                   │
│    'UNSIGNED-PAYLOAD',              // payload 标识                  │
│  ].join('\n')                                                       │
│                                                                     │
│  Step 4: 构建 String to Sign (line 117-124)                       │
│  ───────────────────────────────────────────                       │
│  algorithm = 'AWS4-HMAC-SHA256'                                    │
│  credentialScope = `${dateStamp}/${STORAGE_REGION}/s3/aws4_request` │
│  stringToSign = [                                                   │
│    algorithm,                                                        │
│    amzDate,                                                          │
│    credentialScope,                                                  │
│    sha256(canonicalRequest),        // Canonical Request 的 SHA256  │
│  ].join('\n')                                                        │
│                                                                     │
│  Step 5: 生成签名密钥 (line 127-135)                               │
│  ────────────────────────────────────────                           │
│  // getSignatureKey 函数 (line 64-75)                              │
│  kDate    = HMAC-SHA256("AWS4" + secretKey, dateStamp)           │
│  kRegion  = HMAC-SHA256(kDate, regionName)                        │
│  kService = HMAC-SHA256(kRegion, serviceName)                     │
│  kSigning = HMAC-SHA256(kService, "aws4_request")                 │
│                                                                     │
│  signature = HMAC-SHA256(kSigning, stringToSign).hex()            │
│                                                                     │
│  Step 6: 构建 Authorization 头 (line 137-145)                     │
│  ─────────────────────────────────────────────                      │
│  Authorization =                                                    │
│    "AWS4-HMAC-SHA256 " +                                           │
│    "Credential=${accessKey}/${credentialScope}, " +               │
│    "SignedHeaders=${signedHeaders}, " +                            │
│    "Signature=${signature}"                                         │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

#### 4.3 关键代码：签名生成

```typescript
// storage.server.ts:64-75 - 签名密钥生成
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

// storage.server.ts:127-135 - 最终签名计算
const signingKey = getSignatureKey(
  STORAGE_SECRET_KEY,
  dateStamp,
  STORAGE_REGION,
  's3',
)
const signature = createHmac('sha256', signingKey)
  .update(stringToSign)
  .digest('hex')
```

#### 4.4 上传执行流程

```typescript
// storage.server.ts:11-27 - uploadToStorage
async function uploadToStorage(file: File | FileUpload, key: string) {
  // Step 1: 获取签名后的请求信息
  const { url, headers } = getSignedPutRequestInfo(file, key)

  // Step 2: 直接从服务端发起 PUT 请求到对象存储
  const uploadResponse = await fetch(url, {
    method: 'PUT',
    headers,
    body: file instanceof File ? file : (file as FileUpload).stream(),
  })

  if (!uploadResponse.ok) {
    throw new Error(`Failed to upload object: ${key}`)
  }

  return key  // 返回 objectKey，用于存储到数据库
}
```

#### 4.5 存储路径结构

```typescript
// storage.server.ts:40-50
export async function uploadNoteImage(
  userId: string,
  noteId: string,
  file: File | FileUpload,
) {
  const fileId = createId()  // CUID: 冲突概率极低的唯一ID
  const fileExtension = file.name.split('.').pop() || ''
  const timestamp = Date.now()
  
  // 最终路径: users/{userId}/notes/{noteId}/images/{timestamp}-{fileId}.{ext}
  const key = `users/${userId}/notes/${noteId}/images/${timestamp}-${fileId}.${fileExtension}`
  
  return uploadToStorage(file, key)
}
```

**路径示例**:
```
users/clx123abc/notes/not123/images/1714838400000-abc456def.jpg
```

### 5. 数据库持久化

**文件位置**: `app/routes/users/$username/notes/+shared/note-editor.server.tsx:100-126`

```typescript
const updatedNote = await prisma.note.upsert({
  select: { id: true, owner: { select: { username: true } } },
  where: { id: noteId },
  
  create: {
    id: noteId,
    ownerId: userId,
    title,
    content,
    images: { create: newImages },  // 创建新图片记录
  },
  
  update: {
    title,
    content,
    images: {
      // 删除不再需要的图片
      deleteMany: { id: { notIn: imageUpdates.map((i) => i.id) } },
      
      // 更新已有图片（如果有新文件则重新生成 ID 以 bust cache）
      updateMany: imageUpdates.map((updates) => ({
        where: { id: updates.id },
        data: {
          ...updates,
          id: updates.objectKey ? cuid() : updates.id,  // 缓存击穿策略
        },
      })),
      
      create: newImages,  // 新增图片
    },
  },
})
```

### 6. 图片访问流程

**文件位置**: `app/routes/resources/images.tsx`

```
┌────────────────────────────────────────────────────────────────────┐
│                        图片访问流程                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  客户端请求: /resources/images?objectKey=users/.../image.jpg      │
│       │                                                             │
│       ▼                                                             │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  images.tsx loader (line 28-80)                               │  │
│  │                                                                │  │
│  │  1. 解析 objectKey 参数                                        │  │
│  │  2. 调用 getSignedGetRequestInfo(objectKey)                   │  │
│  │     - 生成 GET 请求的 AWS4 签名                                │  │
│  │  3. 返回 { type: 'fetch', url: signedUrl, headers: ... }    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│       │                                                             │
│       ▼                                                             │
│  getImgResponse (openimg/node)                                      │
│       │                                                             │
│       ├─> 本地缓存检查 (./tests/fixtures/openimg 或 /data/images) │
│       ├─> 若命中缓存：直接返回缓存文件                              │
│       └─> 若未命中：使用 signedUrl fetch Tigris，缓存后返回       │
│                                                                     │
│  客户端收到: 处理后的图片（支持 resize、format 转换）               │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

#### 6.1 前端图片 URL 构建

```typescript
// app/utils/misc.tsx:14-16
export function getNoteImgSrc(objectKey: string) {
  return `/resources/images?objectKey=${encodeURIComponent(objectKey)}`
}
```

#### 6.2 GET 请求签名生成

```typescript
// storage.server.ts:169-179
export function getSignedGetRequestInfo(key: string) {
  const { url, baseHeaders } = getBaseSignedRequestInfo({
    method: 'GET',  // 使用 GET 方法
    key,
  })

  return {
    url,
    headers: baseHeaders,
  }
}
```

## 关键问题解答

### Q1: 签名是在哪一层生成的？

**答案**: **服务端 (Server-only)**

**具体位置**: `app/utils/storage.server.ts`

**原因分析**:
1. **文件名后缀**: `.server.ts` 表示这是 Remix/Epic Stack 中的服务端专用文件，不会打包到客户端
2. **环境变量依赖**: 使用了 `AWS_SECRET_ACCESS_KEY` 等敏感环境变量，这些变量只存在于服务端
3. **Node.js 内置模块**: 使用了 `crypto` 模块的 `createHmac`、`createHash`，这些是 Node.js 特有的

**签名生成函数**:
| 函数名 | 职责 | 行号 |
|--------|------|------|
| `getBaseSignedRequestInfo` | 核心签名逻辑（AWS4 完整实现） | 77-148 |
| `getSignedPutRequestInfo` | 封装 PUT 请求签名 | 150-167 |
| `getSignedGetRequestInfo` | 封装 GET 请求签名 | 169-179 |

### Q2: 客户端直传是怎么协调的？

**答案**: **这不是真正的客户端直传，而是服务端中转上传**

#### 实际架构 vs 客户端直传架构对比

```
【当前实现：服务端中转】
┌──────────┐      ┌──────────┐      ┌──────────┐
│  浏览器   │─────▶│  应用服务器 │─────▶│  对象存储  │
│          │ POST │  (签名+   │ PUT  │  (Tigris) │
│          │ 文件  │   中转)   │      │          │
└──────────┘      └──────────┘      └──────────┘
     │                 │                  │
     │  1. 选择文件     │  2. 接收文件      │  3. 验证签名     │
     │  2. 本地预览     │  3. 生成 AWS4 签名│  4. 存储文件     │
     │  3. 提交表单     │  4. PUT 到存储    │  5. 返回响应     │
     │                 │  5. 保存 objectKey │                  │

【真正的客户端直传架构（当前未实现）】
┌──────────┐      ┌──────────┐      ┌──────────┐
│  浏览器   │─────▶│  应用服务器 │─────▶│  对象存储  │
│          │      │          │      │          │
│  1. 请求 ├─────▶│  2. 生成  │      │          │
│     签名  │      │     签名  │      │          │
│          │◀─────┤  3. 返回  │      │          │
│  4. 使用  │      │  signedUrl│      │          │
│  signedUrl│      │          │      │          │
│  直接上传 ├─────────────────────▶│  5. 接收   │
│  PUT     │                      │     文件    │
└──────────┘                      └──────────┘
```

#### 为什么说这是服务端中转？

关键证据来自 `uploadToStorage` 函数 (line 11-27):

```typescript
async function uploadToStorage(file: File | FileUpload, key: string) {
  const { url, headers } = getSignedPutRequestInfo(file, key)

  // 👇 关键：是从服务端发起的 fetch，不是返回给客户端
  const uploadResponse = await fetch(url, {
    method: 'PUT',
    headers,
    body: file instanceof File ? file : (file as FileUpload).stream(),
  })
  // ...
}
```

**数据流**:
1. `客户端` → (multipart/form-data POST) → `应用服务器`
2. `应用服务器` 解析表单，得到 `File | FileUpload` 对象
3. `应用服务器` 调用 `getSignedPutRequestInfo()` 生成签名
4. `应用服务器` 使用 `fetch()` **自己** 将文件 PUT 到对象存储
5. `应用服务器` 将 `objectKey` 保存到数据库
6. `应用服务器` 返回重定向响应给客户端

#### 服务端中转的优缺点

**优点**:
| 优势 | 说明 |
|------|------|
| 密钥安全 | `AWS_SECRET_ACCESS_KEY` 永远不会暴露给客户端 |
| 统一验证 | 可以在上传到对象存储前进行额外的业务验证 |
| 简化客户端 | 客户端不需要处理复杂的签名逻辑和直接的 S3 API 调用 |
| 易于调试 | 所有上传逻辑集中在服务端，便于日志和监控 |

**缺点**:
| 劣势 | 说明 |
|------|------|
| 带宽压力 | 文件数据经过应用服务器中转，增加服务器带宽消耗 |
| 延迟增加 | 多了一次网络跳变（客户端→服务器→存储） |
| 可扩展性 | 高并发上传时，应用服务器可能成为瓶颈 |

#### 如果要实现真正的客户端直传，需要修改什么？

如果想改为真正的客户端直传架构，需要：

1. **新增一个 API 端点** 用于客户端预获取签名：
```typescript
// 示例：新增 app/routes/resources/upload-sign.ts
export async function loader({ request }: LoaderArgs) {
  const userId = await requireUserId(request)
  const url = new URL(request.url)
  const fileName = url.searchParams.get('fileName')
  const contentType = url.searchParams.get('contentType')
  
  const key = `users/${userId}/temp/${Date.now()}-${fileName}`
  const { url: signedUrl, headers } = getSignedPutRequestInfo({
    type: contentType,
    name: fileName,
  }, key)
  
  return json({ signedUrl, headers, key })
}
```

2. **修改客户端** 直接使用 signedUrl 上传：
```typescript
// 客户端代码示例
const { signedUrl, headers, key } = await fetch('/resources/upload-sign?...')
  .then(r => r.json())

// 直接 PUT 到对象存储，不经过应用服务器
await fetch(signedUrl, {
  method: 'PUT',
  headers,
  body: file,
})
```

3. **修改服务端** 只接收 objectKey，不接收文件：
```typescript
// note-editor.server.tsx 修改
// 不再需要 parseFormData 解析文件
// 只需要接收客户端上传成功后的 objectKey
```

## 完整时序图

```
┌──────────┐         ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│  客户端   │         │  应用服务器   │         │  对象存储     │         │   数据库     │
└────┬─────┘         └──────┬───────┘         └──────┬───────┘         └──────┬───────┘
     │                       │                         │                         │
     │  1. 选择图片文件       │                         │                         │
     │  (FileReader 预览)    │                         │                         │
     │◀─────────────────────▶│                         │                         │
     │                       │                         │                         │
     │  2. 提交表单           │                         │                         │
     │  POST (multipart)     │                         │                         │
     │──────────────────────▶│                         │                         │
     │                       │                         │                         │
     │                       │  3. parseFormData       │                         │
     │                       │  解析文件和表单字段      │                         │
     │                       │                         │                         │
     │                       │  4. Zod Schema 验证      │                         │
     │                       │  - superRefine          │                         │
     │                       │  - transform 开始处理图片 │                         │
     │                       │                         │                         │
     │                       │  5. uploadNoteImage     │                         │
     │                       │  生成 objectKey 路径     │                         │
     │                       │                         │                         │
     │                       │  6. uploadToStorage     │                         │
     │                       │  ┌───────────────────┐  │                         │
     │                       │  │ getSignedPutRequ- │  │                         │
     │                       │  │ estInfo           │  │                         │
     │                       │  │ - 构建 Canonical  │  │                         │
     │                       │  │   Request         │  │                         │
     │                       │  │ - StringToSign    │  │                         │
     │                       │  │ - AWS4 签名计算    │  │                         │
     │                       │  │ - 返回 url+headers │  │                         │
     │                       │  └───────────────────┘  │                         │
     │                       │                         │                         │
     │                       │  7. fetch(PUT)          │                         │
     │                       │  携带签名和文件内容      │                         │
     │                       │────────────────────────▶│                         │
     │                       │                         │                         │
     │                       │                         │  8. 验证 AWS4 签名       │
     │                       │                         │  9. 存储文件             │
     │                       │◀────────────────────────│                         │
     │                       │                         │                         │
     │                       │ 10. 返回 objectKey      │                         │
     │                       │     给 transform 回调    │                         │
     │                       │                         │                         │
     │                       │ 11. prisma.note.upsert │                         │
     │                       │     保存图片元数据       │────────────────────────▶│
     │                       │                         │                         │
     │                       │                         │◀────────────────────────│
     │                       │                         │                         │
     │                       │ 12. redirect 到笔记详情  │                         │
     │◀──────────────────────│                         │                         │
     │                       │                         │                         │
     │ 13. 访问笔记详情页     │                         │                         │
     │     图片通过           │                         │                         │
     │     /resources/images │                         │                         │
     │     代理访问           │────────────────────────▶│                         │
     │                       │  getSignedGetRequestInfo│                         │
     │                       │  生成 GET 签名后 fetch   │────────────────────────▶│
     │                       │                         │                         │
     │◀──────────────────────│◀────────────────────────│                         │
     │                       │                         │                         │
┌────┴─────┐         ┌──────┴───────┐         ┌──────┴───────┐         ┌──────┴───────┐
│  客户端   │         │  应用服务器   │         │  对象存储     │         │   数据库     │
└──────────┘         └──────────────┘         └──────────────┘         └──────────────┘
```

## 涉及文件清单

| 文件路径 | 职责 | 关键行号 |
|---------|------|---------|
| `app/routes/users/$username/notes/+shared/note-editor.tsx` | 前端图片选择、预览、表单提交 | 175-282 (ImageChooser), 80-84 (Form) |
| `app/routes/users/$username/notes/+shared/note-editor.server.tsx` | 服务端表单解析、图片上传协调、数据库操作 | 48-81 (transform), 100-126 (upsert) |
| `app/utils/storage.server.ts` | AWS4 签名生成、服务端中转上传 | 77-148 (签名核心), 11-27 (上传执行) |
| `app/routes/resources/images.tsx` | 图片访问代理、缓存、GET 签名 | 28-80 (loader), 44-53 (getImgSource) |
| `app/utils/misc.tsx` | 图片 URL 构建辅助函数 | 8-16 (getUserImgSrc, getNoteImgSrc) |

## 环境变量依赖

```bash
# .env 或部署环境中需要配置
AWS_ACCESS_KEY_ID="your-access-key"
AWS_SECRET_ACCESS_KEY="your-secret-key"  # 关键：用于签名，绝不暴露给客户端
AWS_REGION="auto"                         # Tigris 使用 auto
AWS_ENDPOINT_URL_S3="https://fly.storage.tigris.dev"
BUCKET_NAME="your-bucket-name"
```

## 总结

### 核心要点

1. **签名位置**: 完全在服务端 (`storage.server.ts`)，使用 AWS Signature Version 4 算法

2. **上传模式**: 服务端中转，而非客户端直传
   - 文件先传到应用服务器
   - 应用服务器生成签名后 PUT 到对象存储
   - objectKey 保存到 SQLite 数据库

3. **图片访问**: 通过 `/resources/images` 端点代理
   - 服务端生成 GET 签名
   - 支持本地缓存
   - 支持图片处理 (resize, format)

4. **缓存策略**: 更新图片时重新生成 `id` (line 120)，实现缓存击穿

### 设计考量

这种服务端中转的设计在 Epic Stack 中是**有意为之**的，因为：

1. **简化开发体验**: 不需要客户端处理复杂的 AWS SDK 和签名逻辑
2. **统一权限控制**: 所有文件操作都经过应用服务器的权限验证
3. **本地开发友好**: 可以使用 MSW (Mock Service Worker) 模拟对象存储
4. **符合 Epic Stack 哲学**: "Convention over Configuration"，提供开箱即用的方案

对于大多数中小型应用，这种设计是完全合适的。只有当上传量非常大（每天数百万次上传）时，才需要考虑改为真正的客户端直传架构以减轻应用服务器的压力。
