# Epic Stack 资产存储深度分析报告（R2）

## 概述

本报告在第一版分析的基础上，深入分析三个关键领域：
1. **前端文件选择限制与后端真实校验能力的差异**
2. **对象回收机制缺失导致的生命周期管理与成本风险**
3. **图片读取链路的权限边界分析及改进建议**

---

## 一、前端文件选择限制与后端真实校验能力分析

### 1.1 前端限制机制

前端文件选择主要通过 HTML5 文件输入的属性进行限制：

#### 用户头像上传 (`photo.tsx`)

```typescript
// app/routes/settings/profile/photo.tsx:160-176
<input
	{...getInputProps(fields.photoFile, { type: 'file' })}
	accept="image/*"
	className="peer sr-only"
	required
	tabIndex={newImageSrc ? -1 : 0}
	onChange={(e) => {
		// ... 预览逻辑
	}}
/>
```

**前端限制：**
| 限制类型 | 实现方式 | 限制值 |
|---------|---------|--------|
| 文件类型 | `accept="image/*"` | 仅图片类型 |
| 必填 | `required` 属性 | 必须选择文件 |

#### 笔记图片上传 (`note-editor.tsx`)

```typescript
// app/routes/users/$username/notes/+shared/note-editor.tsx:236-255
<input
	aria-label="Image"
	className="absolute top-0 left-0 z-0 size-32 cursor-pointer opacity-0"
	onChange={(event) => {
		// 预览逻辑
	}}
	accept="image/*"
	{...getInputProps(fields.file, { type: 'file' })}
	key={fields.file.key}
/>
```

**前端限制：**
| 限制类型 | 实现方式 | 限制值 |
|---------|---------|--------|
| 文件类型 | `accept="image/*"` | 仅图片类型 |
| 数量限制 | `z.array(...).max(5)` | 最多 5 张图片 |

### 1.2 前端限制的可绕过性分析

**⚠️ 重要安全提示：前端限制可被轻易绕过**

| 限制类型 | 绕过方式 | 风险等级 |
|---------|---------|---------|
| `accept="image/*"` | 修改文件扩展名、使用浏览器开发者工具删除属性 | **高** |
| `required` | 使用开发者工具移除属性、提交空表单 | 中 |
| 客户端预览检查 | 绕过 JavaScript 执行 | 中 |

**攻击场景示例：**

1. **文件类型绕过：**
   ```bash
   # 将恶意 PHP 文件重命名为 .jpg
   mv malware.php malware.jpg
   # 前端 accept="image/*" 会接受此文件
   ```

2. **通过开发者工具绕过：**
   - 按 F12 打开开发者工具
   - 定位到文件输入元素
   - 删除 `accept="image/*"` 属性
   - 现在可以选择任何类型的文件

### 1.3 后端真实校验能力

让我们深入分析后端的实际校验机制。

#### Schema 校验层

**用户头像 Schema：**
```typescript
// app/routes/settings/profile/photo.tsx:29-44
const MAX_SIZE = 1024 * 1024 * 3 // 3MB

const NewImageSchema = z.object({
	intent: z.literal('submit'),
	photoFile: z
		.instanceof(File)
		.refine((file) => file.size > 0, 'Image is required')
		.refine(
			(file) => file.size <= MAX_SIZE,
			'Image size must be less than 3MB',
		),
})
```

**笔记图片 Schema：**
```typescript
// app/routes/users/$username/notes/+shared/note-editor.tsx:31-42
export const MAX_UPLOAD_SIZE = 1024 * 1024 * 3 // 3MB

const ImageFieldsetSchema = z.object({
	id: z.string().optional(),
	file: z
		.instanceof(File)
		.optional()
		.refine((file) => {
			return !file || file.size <= MAX_UPLOAD_SIZE
		}, 'File size must be less than 3MB'),
	altText: z.string().optional(),
})
```

**后端校验能力分析：**

| 校验项 | 实现位置 | 是否真实验证 | 说明 |
|--------|---------|-------------|------|
| 文件大小 | `z.refine(file.size <= MAX_SIZE)` | ✅ 是 | 检查实际字节数 |
| 文件非空 | `z.refine(file.size > 0)` | ✅ 是 | 检查实际大小 |
| **文件类型** | ❌ 无 | ❌ 否 | **仅依赖 `file.type`，未校验实际内容** |
| 图片数量 | `z.array(...).max(5)` | ✅ 是 | 限制最多 5 张 |

#### 文件类型校验的严重缺失

**关键问题：依赖 `file.type` 而非实际文件内容**

```typescript
// app/utils/storage.server.ts:150-167
function getSignedPutRequestInfo(file: File | FileUpload, key: string) {
	const uploadDate = new Date().toISOString()
	const { url, baseHeaders } = getBaseSignedRequestInfo({
		method: 'PUT',
		key,
		contentType: file.type,  // ⚠️ 依赖客户端提供的 file.type
		uploadDate,
	})

	return {
		url,
		headers: {
			...baseHeaders,
			'Content-Type': file.type,  // ⚠️ 直接使用客户端提供的值
			'X-Amz-Meta-Upload-Date': uploadDate,
		},
	}
}
```

**问题分析：**

1. **`file.type` 来自客户端**：
   - `File.type` 属性由浏览器根据文件扩展名或 MIME 类型嗅探设置
   - 攻击者可以伪造请求，设置任意 `Content-Type`
   - 例如：上传 PHP 文件但设置 `Content-Type: image/jpeg`

2. **无实际文件内容校验**：
   - 没有检查文件魔数（Magic Number）
   - 没有解析文件头验证实际格式
   - 没有进行病毒扫描或恶意内容检测

3. **存储服务直接信任**：
   - 存储服务（S3/Tigris）会存储上传的任何内容
   - 仅根据 `Content-Type` 设置元数据，不验证内容匹配

### 1.4 前端限制 vs 后端校验对比表

| 检查项 | 前端限制 | 后端校验 | 是否足够安全 |
|--------|---------|---------|-------------|
| 文件大小 | ❌ 无（仅预览时可能检查） | ✅ `file.size <= 3MB` | ✅ 安全 |
| 非空检查 | ⚠️ `required` 属性（可绕过） | ✅ `file.size > 0` | ✅ 安全 |
| 文件类型 | ⚠️ `accept="image/*"`（可绕过） | ❌ 无（仅依赖 `file.type`） | ❌ **不安全** |
| 图片数量 | ⚠️ 前端动态限制 | ✅ `z.array(...).max(5)` | ✅ 安全 |
| 实际内容验证 | ❌ 无 | ❌ 无 | ❌ **高风险** |

### 1.5 安全风险与改进建议

#### 当前风险

1. **任意文件上传风险**：
   - 攻击者可以上传脚本文件（PHP、ASPX、JS 等）
   - 虽然文件存储在对象存储中，但如果存在文件包含漏洞或被不当提供执行权限，可能导致 RCE

2. **内容欺骗风险**：
   - 上传非图片内容但伪装为图片
   - 可能用于存储恶意内容、钓鱼页面等

3. **存储滥用风险**：
   - 虽然有大小限制，但没有类型限制
   - 可能被用于存储非授权内容（视频、压缩包等）

#### 改进建议

**方案一：添加文件类型校验（推荐）**

```typescript
// 建议新增的校验函数
import { fileTypeFromBuffer } from 'file-type'
import { Readable } from 'node:stream'
import { buffer } from 'node:stream/consumers'

const ALLOWED_IMAGE_TYPES = new Set([
	'image/jpeg',
	'image/png',
	'image/gif',
	'image/webp',
	'image/svg+xml',
])

async function validateImageFile(file: File | FileUpload): Promise<{ 
	isValid: boolean
	detectedType?: string
}> {
	let fileBuffer: Buffer
	
	if (file instanceof File) {
		const arrayBuffer = await file.arrayBuffer()
		fileBuffer = Buffer.from(arrayBuffer)
	} else {
		// 对于 FileUpload，需要先读取流
		fileBuffer = await buffer(file.stream() as unknown as Readable)
	}
	
	// 1. 检查魔数（Magic Number）
	const type = await fileTypeFromBuffer(fileBuffer)
	
	// 2. 验证类型是否在允许列表中
	if (type && ALLOWED_IMAGE_TYPES.has(type.mime)) {
		return { isValid: true, detectedType: type.mime }
	}
	
	// 3. 额外检查 SVG（因为 file-type 可能无法检测所有 SVG）
	if (file.type === 'image/svg+xml') {
		// 简单的 SVG 检查
		const content = fileBuffer.toString('utf-8').trim()
		if (content.startsWith('<svg') || content.startsWith('<?xml')) {
			return { isValid: true, detectedType: 'image/svg+xml' }
		}
	}
	
	return { isValid: false, detectedType: type?.mime }
}
```

**方案二：使用存储服务策略**

在存储服务端配置策略，限制可上传的文件类型：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlyImages",
      "Effect": "Allow",
      "Principal": {"AWS": ["arn:aws:iam::123456789012:user/your-user"]},
      "Action": ["s3:PutObject"],
      "Resource": ["arn:aws:s3:::your-bucket/*"],
      "Condition": {
        "StringLike": {"s3:content-type": ["image/*"]}
      }
    }
  ]
}
```

**方案三：图片处理管道（最安全）**

上传后使用图片处理库重新编码，彻底清除可能的恶意内容：

```typescript
import sharp from 'sharp'

async function processImage(file: File | FileUpload, key: string) {
	// 1. 读取原始文件
	const fileBuffer = await readFileToBuffer(file)
	
	// 2. 使用 sharp 重新编码（清除所有元数据和潜在恶意内容）
	const processedBuffer = await sharp(fileBuffer)
		.resize({ width: 2000, height: 2000, fit: 'inside' })
		.toFormat('jpeg', { quality: 85 })
		.toBuffer()
	
	// 3. 上传处理后的图片
	return uploadToStorage(
		new File([processedBuffer], key, { type: 'image/jpeg' }),
		key
	)
}
```

---

## 二、对象回收机制缺失分析

### 2.1 当前删除行为分析

让我们深入分析系统中的各种删除操作及其对存储对象的影响。

#### 场景一：用户头像更新/删除

**代码位置：** `app/routes/settings/profile/photo.tsx:91-106`

```typescript
const { image, intent } = submission.value

if (intent === 'delete') {
	// 仅删除数据库记录
	await prisma.userImage.deleteMany({ where: { userId } })
	return redirect('/settings/profile')
}

// 更新头像时的事务
await prisma.$transaction(async ($prisma) => {
	await $prisma.userImage.deleteMany({ where: { userId } })  // ⚠️ 仅删除数据库
	await $prisma.user.update({
		where: { id: userId },
		data: { image: { create: image } },
	})
})
```

**问题：**
- ✅ 删除数据库中的 `UserImage` 记录
- ❌ **未删除存储服务中的旧对象**
- ❌ 每次更新头像都会产生新的 objectKey，旧对象被遗弃

**对象键命名策略：**
```
users/{userId}/profile-images/{timestamp}-{fileId}.{extension}
```

每次上传都会生成新的 `timestamp` 和 `fileId`，因此：
- 旧对象：`users/user123/profile-images/1715000000-abc123.jpg`
- 新对象：`users/user123/profile-images/1715000100-def456.jpg`
- 旧对象永远保留，不会被清理

#### 场景二：笔记图片删除

**代码位置：** `app/routes/users/$username/notes/+shared/note-editor.server.tsx:113-124`

```typescript
images: {
	// 删除不在更新列表中的图片记录
	deleteMany: { id: { notIn: imageUpdates.map((i) => i.id) } },
	// 更新现有图片
	updateMany: imageUpdates.map((updates) => ({
		where: { id: updates.id },
		data: {
			...updates,
			// 如果有新文件，生成新 ID 以 bust 缓存
			id: updates.objectKey ? cuid() : updates.id,
		},
	})),
	// 创建新图片
	create: newImages,
}
```

**问题分析：**

1. **`deleteMany` 操作：**
   - 删除数据库中的 `NoteImage` 记录
   - ❌ **存储服务中的对象未被删除**

2. **更新图片时：**
   ```typescript
   // note-editor.server.tsx:54-67
   imageUpdates: await Promise.all(
       images.filter(imageHasId).map(async (i) => {
           if (imageHasFile(i)) {
               return {
                   id: i.id,
                   altText: i.altText,
                   objectKey: await uploadNoteImage(userId, noteId, i.file),  // ⚠️ 新 objectKey
               }
           } else {
               return {
                   id: i.id,
                   altText: i.altText,
               }
           }
       }),
   ),
   ```
   - 上传新文件时生成**新的 objectKey**
   - 旧的 objectKey 对应的对象被遗弃

#### 场景三：笔记删除

**代码位置：** `app/routes/users/$username/notes/$noteId.tsx:83`

```typescript
await prisma.note.delete({ where: { id: note.id } })
```

**问题分析：**

1. **数据库级联删除：**
   - `NoteImage` 模型定义了 `onDelete: Cascade`
   - 删除笔记时，相关 `NoteImage` 记录会被自动删除
   - ✅ 数据库层面是完整的

2. **存储服务：**
   - ❌ **对象存储中的图片文件完全未被处理**
   - 所有关联的 objectKey 对应的文件成为孤立文件

#### 场景四：用户账户删除

**代码位置：** `app/routes/settings/profile/index.tsx:340`

```typescript
async function deleteDataAction({ userId }: ProfileActionArgs) {
	await prisma.user.delete({ where: { id: userId } })
	// ...
}
```

**问题分析：**

1. **数据库级联删除链：**
   ```
   User.delete()
     → UserImage (onDelete: Cascade)
     → Note (onDelete: Cascade)
       → NoteImage (onDelete: Cascade)
   ```
   - ✅ 所有数据库记录被正确删除

2. **存储服务：**
   - ❌ **用户目录下的所有对象被遗弃**
   - 路径模式：`users/{userId}/**/*` 下的所有文件
   - 包括：头像、所有笔记图片

### 2.2 对象遗弃场景汇总表

| 操作场景 | 数据库记录 | 存储对象 | 遗弃对象数量 |
|---------|-----------|---------|-------------|
| 更新用户头像 | ✅ 替换旧记录 | ❌ 旧对象保留 | 每次更新 +1 |
| 删除用户头像 | ✅ 删除记录 | ❌ 对象保留 | +1 |
| 更新笔记图片 | ✅ 更新记录 | ❌ 旧对象保留 | 每个更新 +1 |
| 删除笔记图片 | ✅ 删除记录 | ❌ 对象保留 | 每个删除 +1 |
| 删除笔记 | ✅ 级联删除 | ❌ 所有关联对象保留 | N（笔记图片数） |
| 删除用户账户 | ✅ 级联删除 | ❌ 用户目录所有对象 | 大量 |

### 2.3 生命周期与成本风险分析

#### 风险一：存储成本无限增长

**计算示例：**

假设一个活跃用户：
- 每月更新 2 次头像
- 每月创建 10 篇笔记，每篇 3 张图片
- 每月编辑 5 篇笔记，更新 2 张图片
- 平均图片大小：500KB

**月度遗弃对象：**
- 头像更新：2 个 × 500KB = 1MB
- 笔记创建：10 × 3 = 30 个 × 500KB = 15MB
- 笔记更新：5 × 2 = 10 个 × 500KB = 5MB
- **月度总计：42 个对象，21MB**

**年度成本（假设 1000 活跃用户）：**
- 年度遗弃数据：1000 用户 × 21MB × 12 月 = 252,000 MB = **246 GB**
- 存储成本（按 $0.02/GB/月）：246 GB × $0.02 × 12 = **约 $59/年**
- **长期（3年）：约 $738，累计 738 GB**

**风险放大场景：**
- 如果用户上传高清图片（5MB/张）
- 月度遗弃：42 × 5MB = 210MB/用户
- 1000 用户年度：2.46 TB
- 成本：约 $590/年

#### 风险二：合规与隐私风险

| 风险类型 | 说明 |
|---------|------|
| GDPR/CCPA 合规 | 用户请求删除数据时，存储服务中的数据未被真正删除 |
| 数据泄露风险 | 遗弃的敏感图片可能被意外访问（如果存储桶配置不当） |
| 审计困难 | 无法准确说明哪些用户数据已被删除 |
| 保留政策违规 | 如果有数据保留期限要求，遗弃对象可能违反政策 |

#### 风险三：管理与运维风险

| 风险类型 | 说明 |
|---------|------|
| 存储桶膨胀 | 对象数量无限制增长，影响存储服务性能 |
| 备份成本增加 | 遗弃对象也会被备份，增加备份存储成本 |
| 灾难恢复复杂 | 恢复时需要处理大量无用数据 |
| 成本估算困难 | 无法准确预测存储成本增长 |

#### 风险四：安全风险

| 风险类型 | 说明 |
|---------|------|
| 敏感数据残留 | 用户已删除的头像/笔记图片仍在存储中 |
| 无法彻底清除 | 恶意内容上传后，即使删除数据库记录，文件仍存在 |
| 取证困难 | 安全事件调查时，无法确定哪些文件是有效的 |

### 2.4 改进建议

#### 方案一：软删除 + 定时清理任务（推荐）

**步骤 1：添加软删除字段**

```prisma
// 修改 schema.prisma
model UserImage {
  // ... 现有字段
  deletedAt   DateTime?
  
  @@index([userId])
  @@index([deletedAt])
}

model NoteImage {
  // ... 现有字段
  deletedAt   DateTime?
  
  @@index([noteId])
  @@index([deletedAt])
}
```

**步骤 2：修改删除操作**

```typescript
// 在删除时记录 objectKey，而不是直接删除
interface DeletedObject {
	id: string
	objectKey: string
	deletedAt: DateTime
	modelType: 'UserImage' | 'NoteImage'
}

// 创建专门的清理队列表
model DeletedStorageObject {
	id          String   @id @default(cuid())
	objectKey   String   @unique
	modelType   String   // 'UserImage' | 'NoteImage'
	originalId  String   // 原记录 ID
	deletedAt   DateTime @default(now())
	
	@@index([deletedAt])
}
```

**步骤 3：删除时记录清理任务**

```typescript
// 修改 photo.tsx 的删除逻辑
if (intent === 'delete') {
	// 1. 先获取要删除的记录
	const oldImages = await prisma.userImage.findMany({
		where: { userId },
		select: { id: true, objectKey: true },
	})
	
	// 2. 记录到清理队列
	await prisma.$transaction([
		...oldImages.map(img => prisma.deletedStorageObject.create({
			data: {
				objectKey: img.objectKey,
				modelType: 'UserImage',
				originalId: img.id,
			},
		})),
		prisma.userImage.deleteMany({ where: { userId } }),
	])
	
	return redirect('/settings/profile')
}
```

**步骤 4：创建定时清理任务**

```typescript
// app/utils/storage-cleanup.server.ts
import { prisma } from './db.server.ts'
import { getSignedDeleteRequestInfo } from './storage.server.ts'

const CLEANUP_DELAY_HOURS = 24 // 延迟 24 小时删除，允许恢复

export async function cleanupDeletedObjects() {
	const cutoffTime = new Date(Date.now() - CLEANUP_DELAY_HOURS * 60 * 60 * 1000)
	
	const objectsToDelete = await prisma.deletedStorageObject.findMany({
		where: {
			deletedAt: { lte: cutoffTime },
		},
		take: 100, // 批量处理
	})
	
	if (objectsToDelete.length === 0) return { deleted: 0 }
	
	let deletedCount = 0
	const errors: string[] = []
	
	for (const obj of objectsToDelete) {
		try {
			// 需要实现删除函数
			await deleteFromStorage(obj.objectKey)
			await prisma.deletedStorageObject.delete({
				where: { id: obj.id },
			})
			deletedCount++
		} catch (error) {
			errors.push(`Failed to delete ${obj.objectKey}: ${error}`)
			// 可以记录重试次数，超过一定次数后标记为失败
		}
	}
	
	return { deleted: deletedCount, errors, total: objectsToDelete.length }
}

// 需要在 storage.server.ts 添加删除功能
export async function deleteFromStorage(key: string) {
	const { url, headers } = getSignedDeleteRequestInfo(key)
	
	const response = await fetch(url, {
		method: 'DELETE',
		headers,
	})
	
	if (!response.ok) {
		throw new Error(`Failed to delete object ${key}: ${response.status}`)
	}
}

function getSignedDeleteRequestInfo(key: string) {
	// 类似于 getSignedGetRequestInfo，但使用 DELETE 方法
	return getBaseSignedRequestInfo({
		method: 'DELETE' as any, // 需要扩展类型
		key,
	})
}
```

**步骤 5：触发清理任务**

可以使用以下方式触发：

```typescript
// 方式一：每次上传后触发（简单）
export async function uploadProfileImage(userId: string, file: File | FileUpload) {
	// ... 现有上传逻辑
	
	// 异步触发清理（不等待）
	cleanupDeletedObjects().catch(console.error)
	
	return key
}

// 方式二：使用 cron 定时任务（推荐用于生产）
// 可以使用 node-cron 或平台提供的定时任务
```

#### 方案二：存储服务生命周期规则（成本较低）

在存储服务端配置生命周期规则，自动清理旧对象：

```json
{
  "Rules": [
    {
      "ID": "AutoCleanupAbandonedObjects",
      "Prefix": "users/",
      "Status": "Enabled",
      "Tags": [
        {
          "Key": "cleanup-policy",
          "Value": "abandoned"
        }
      ],
      "Expiration": {
        "Days": 90  // 90 天后自动删除
      },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 30  // 旧版本 30 天后删除
      }
    }
  ]
}
```

**优点：**
- 无需修改应用代码
- 存储服务自动管理

**缺点：**
- 无法区分"有效"和"遗弃"对象
- 可能误删仍在使用的对象
- 不够精确

#### 方案三：引用计数 + 垃圾回收

**实现思路：**
1. 记录每个 objectKey 的引用计数
2. 删除操作时减少计数
3. 计数为 0 时真正删除

```prisma
model StorageObjectReference {
  id          String   @id @default(cuid())
  objectKey   String
  refType     String   // 'UserImage' | 'NoteImage'
  refId       String   // 引用记录的 ID
  createdAt   DateTime @default(now())
  
  @@unique([objectKey, refType, refId])
  @@index([objectKey])
}
```

**优点：**
- 精确追踪每个对象的引用
- 可以安全处理共享对象（如果未来支持）

**缺点：**
- 实现复杂度高
- 需要维护额外的表

### 2.5 实施方案建议

| 实施阶段 | 方案 | 工作量 | 风险降低 |
|---------|------|--------|---------|
| **短期（1-2 周）** | 方案二：存储生命周期规则 | 低 | 部分缓解 |
| **中期（1-2 月）** | 方案一：软删除 + 定时清理 | 中 | 完全解决 |
| **长期（按需）** | 方案三：引用计数 | 高 | 企业级方案 |

**立即行动建议：**

1. **评估当前遗弃数据量**：
   ```bash
   # 使用 AWS CLI 或存储服务工具检查
   aws s3 ls s3://your-bucket/users/ --recursive --summarize
   ```

2. **配置存储生命周期规则**作为临时措施

3. **规划实施软删除方案**

---

## 三、图片读取链路权限边界分析

### 3.1 当前读取流程分析

#### 步骤 1：URL 生成

```typescript
// app/utils/misc.tsx:8-16
export function getUserImgSrc(objectKey?: string | null) {
	return objectKey
		? `/resources/images?objectKey=${encodeURIComponent(objectKey)}`
		: '/img/user.png'
}

export function getNoteImgSrc(objectKey: string) {
	return `/resources/images?objectKey=${encodeURIComponent(objectKey)}`
}
```

**URL 格式：**
```
/resources/images?objectKey=users%2Fuser123%2Fprofile-images%2Fxxx.jpg
```

#### 步骤 2：图片资源路由处理

```typescript
// app/routes/resources/images.tsx:28-80
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
			if (objectKey) {
				// ⚠️ 直接使用 objectKey，无权限检查
				const { url: signedUrl, headers: signedHeaders } =
					getSignedGetRequestInfo(objectKey)
				return {
					type: 'fetch',
					url: signedUrl,
					headers: signedHeaders,
				}
			}
			// ... 其他来源处理
		},
	})
}
```

**关键问题：无任何权限验证！**

### 3.2 权限边界深度分析

#### 当前权限模型

```
┌─────────────────────────────────────────────────────────────┐
│                        权限检查点                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  页面/API 层（有检查）                                        │
│  ├── 笔记详情页：检查用户是否有权查看笔记                        │
│  ├── 用户资料页：公开访问                                      │
│  └── 设置页面：需要登录                                        │
│                                                             │
│  图片资源层（无检查）                                          │
│  └── /resources/images?objectKey=xxx：任何人可访问            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 问题一：对象键可枚举性分析

**对象键命名模式：**

```
头像: users/{userId}/profile-images/{timestamp}-{fileId}.{extension}
笔记: users/{userId}/notes/{noteId}/images/{timestamp}-{fileId}.{extension}
```

**可枚举性风险：**

| 组件 | 是否可预测 | 风险 |
|------|-----------|------|
| `userId` | ⚠️ 可能可预测（cuid2 有一定随机性） | 中 |
| `noteId` | ⚠️ 同样使用 cuid2 | 中 |
| `timestamp` | ✅ 可预测（Unix 时间戳） | 高 |
| `fileId` | ✅ cuid2，理论上不可预测 | 低 |
| `extension` | ✅ 可预测（jpg、png 等） | 低 |

**暴力枚举可行性：**

- 知道 `userId` 和大致时间范围的攻击者
- 可以尝试枚举 `timestamp`（分钟级精度就有 1440 种可能/天）
- 但 `fileId` 是 cuid2（24+字符），暴力枚举不可行

**实际风险场景：**

1. **对象键泄露**：
   - 日志中记录 URL
   - 浏览器历史记录
   - 页面源码中暴露（即使是其他用户的页面）

2. **示例：用户列表页面**
   ```tsx
   // app/routes/users/index.tsx:50-54
   <Img
       alt={user.name ?? user.username}
       src={getUserImgSrc(user.imageObjectKey)}  // ⚠️ 所有用户头像 objectKey 都暴露
       className="size-16 rounded-full"
       width={256}
       height={256}
   />
   ```
   - 未登录用户也可以访问用户列表
   - 所有用户的头像 objectKey 都被暴露

#### 问题二：横向越权访问

**场景描述：**

假设系统存在以下资源：

| 资源 | objectKey | 访问权限（页面级） |
|------|-----------|-------------------|
| 用户 A 的头像 | `users/a/profile-images/1.jpg` | 公开（用户资料页） |
| 用户 A 的公开笔记图片 | `users/a/notes/public-1/images/1.jpg` | 公开 |
| 用户 A 的私有笔记图片 | `users/a/notes/private-1/images/1.jpg` | ❌ 仅所有者可见 |
| 用户 B 的头像 | `users/b/profile-images/1.jpg` | 公开 |
| 用户 B 的私有笔记图片 | `users/b/notes/private-2/images/1.jpg` | ❌ 仅 B 可见 |

**当前行为：**

```typescript
// 任何用户，甚至未登录用户，可以直接访问：

// ✅ 应该允许：公开头像
GET /resources/images?objectKey=users%2Fa%2Fprofile-images%2F1.jpg

// ✅ 应该允许：公开笔记图片
GET /resources/images?objectKey=users%2Fa%2Fnotes%2Fpublic-1%2Fimages%2F1.jpg

// ⚠️ 但实际也允许：私有笔记图片（越权！）
GET /resources/images?objectKey=users%2Fa%2Fnotes%2Fprivate-1%2Fimages%2F1.jpg

// ⚠️ 甚至其他用户的私有图片
GET /resources/images?objectKey=users%2Fb%2Fnotes%2Fprivate-2%2Fimages%2F1.jpg
```

**越权访问的完整路径：**

```
攻击者构造 URL
    ↓
GET /resources/images?objectKey=users%2Fvictim%2Fnotes%2Fprivate-note%2Fimages%2Fsecret.jpg
    ↓
images.tsx loader 接收请求
    ↓
⚠️ 无权限检查，直接解析 objectKey
    ↓
getSignedGetRequestInfo(objectKey) 生成签名
    ↓
从存储服务获取图片
    ↓
openimg 优化后返回
    ↓
✅ 攻击者成功获取私有图片
```

#### 问题三：缓存导致的权限持续失效

```typescript
// app/routes/resources/images.tsx:32-33
const headers = new Headers()
headers.set('Cache-Control', 'public, max-age=31536000, immutable')
```

**问题分析：**

1. **`public`**：允许任何缓存（CDN、浏览器、代理）缓存
2. **`max-age=31536000`**：缓存 1 年
3. **`immutable`**：表示资源不会改变，缓存可以无限期使用

**权限泄露场景：**

1. 用户 A 的私有笔记图片被攻击者访问
2. CDN 缓存了此图片（因为 Cache-Control: public）
3. 用户 A 后来修改了笔记权限，或者删除了笔记
4. ❌ 缓存中仍然存在此图片
5. ❌ 任何知道 URL 的人都可以从 CDN 获取
6. ❌ 此状态持续 **1 年** 或缓存被手动清除

**更严重的场景：**

- 如果使用了公共 CDN（如 Cloudflare）
- 缓存的内容可能被其他用户访问
- 即使原始权限已被撤销

#### 问题四：路径遍历风险

**代码分析：**

```typescript
// app/utils/misc.tsx:9-10
return objectKey
	? `/resources/images?objectKey=${encodeURIComponent(objectKey)}`
```

`encodeURIComponent` 会对特殊字符进行编码：
- `..` → `%2E%2E`
- `/` → `%2F`

但让我们检查 images.tsx 如何使用：

```typescript
// app/routes/resources/images.tsx:35
const objectKey = searchParams.get('objectKey')
// ...
// app/utils/storage.server.ts:88
const url = `${STORAGE_ENDPOINT}/${STORAGE_BUCKET}/${key}`
```

**直接拼接路径！**

虽然对象存储通常不支持传统的路径遍历（因为键名是扁平的），但风险仍然存在：

1. **存储服务差异**：某些 S3 兼容服务可能有不同的行为
2. **信息泄露**：可能访问到预期外的对象
3. **拒绝服务**：构造特殊键名可能导致错误

### 3.3 权限边界问题汇总

| 问题类型 | 严重程度 | 描述 |
|---------|---------|------|
| 完全无权限检查 | **高** | 任何知道 objectKey 的人都可访问 |
| 私有资源越权 | **高** | 私有笔记图片可被任意访问 |
| 缓存权限泄露 | **高** | `Cache-Control: public` 导致 CDN 缓存私有内容 |
| 长缓存时间 | 中 | `max-age=31536000` 权限撤销后仍可访问 |
| 对象键暴露 | 中 | 公开页面暴露其他用户的 objectKey |
| 路径拼接风险 | 低 | 直接使用 objectKey 构建存储路径 |

### 3.4 改进建议

#### 方案一：添加权限检查层（推荐）

**架构设计：**

```
请求 /resources/images?objectKey=xxx
    ↓
1. 解析 objectKey，判断资源类型
    ↓
2. 查询数据库验证资源存在性和权限
    ├── 用户头像：验证用户存在（通常公开）
    ├── 笔记图片：验证笔记存在且访问权限
    └── 其他：拒绝访问
    ↓
3. 权限检查通过 → 继续处理
   权限检查失败 → 返回 404 或 403
```

**实现代码：**

```typescript
// 新增权限检查函数
import { prisma } from '#app/utils/db.server.ts'
import { requireUserId, optionalRequireUserId } from '#app/utils/auth.server.ts'
import { userHasPermission } from '#app/utils/user.ts'

interface ResourceAccessCheck {
	allowed: boolean
	cacheControl?: string  // 可覆盖默认缓存策略
	reason?: string
}

/**
 * 检查对图片资源的访问权限
 */
async function checkImageAccess(
	request: Request,
	objectKey: string,
): Promise<ResourceAccessCheck> {
	// 解析 objectKey 路径
	const parts = objectKey.split('/')
	
	// 验证基本格式：users/{userId}/...
	if (parts.length < 3 || parts[0] !== 'users') {
		return { allowed: false, reason: 'Invalid objectKey format' }
	}
	
	const resourceUserId = parts[1]  // users/{userId}
	const resourceType = parts[2]    // profile-images 或 notes
	
	// 获取当前用户（可选，未登录用户也可能有访问权限）
	const currentUser = await optionalRequireUserId(request)
	
	switch (resourceType) {
		case 'profile-images':
			// 用户头像通常是公开的
			// 验证用户存在
			const userExists = await prisma.user.findUnique({
				where: { id: resourceUserId },
				select: { id: true },
			})
			if (!userExists) {
				return { allowed: false, reason: 'User not found' }
			}
			// 头像可以公开访问，但使用私有缓存
			return { 
				allowed: true, 
				cacheControl: 'private, max-age=31536000, immutable' 
			}
		
		case 'notes':
			// 笔记图片需要检查笔记权限
			if (parts.length < 5) {
				return { allowed: false, reason: 'Invalid note image path' }
			}
			const noteId = parts[3]  // users/{userId}/notes/{noteId}/...
			
			// 查询笔记及其权限
			const note = await prisma.note.findUnique({
				where: { id: noteId },
				select: {
					id: true,
					ownerId: true,
					// 假设有权限字段，实际可能需要检查关联
				},
			})
			
			if (!note) {
				return { allowed: false, reason: 'Note not found' }
			}
			
			// 权限检查逻辑
			const isOwner = currentUser?.id === note.ownerId
			
			if (isOwner) {
				// 所有者可以访问
				return { 
					allowed: true,
					cacheControl: 'private, max-age=31536000'
				}
			}
			
			// 检查是否有访问笔记的权限
			// 这里需要根据实际的权限系统实现
			// 例如：笔记是否公开，或者当前用户是否被授权
			
			// 简单实现：检查是否有读取权限
			if (currentUser) {
				const hasPermission = await checkNoteReadPermission(
					currentUser.id,
					noteId
				)
				if (hasPermission) {
					return { allowed: true, cacheControl: 'private, max-age=31536000' }
				}
			}
			
			// 无权访问
			return { allowed: false, reason: 'Access denied' }
		
		default:
			return { allowed: false, reason: 'Unknown resource type' }
	}
}

/**
 * 检查笔记读取权限（示例实现）
 */
async function checkNoteReadPermission(
	userId: string,
	noteId: string,
): Promise<boolean> {
	// 实现根据应用的权限模型
	// 这里是一个简单示例
	
	// 1. 检查是否是所有者
	const note = await prisma.note.findUnique({
		where: { id: noteId, ownerId: userId },
		select: { id: true },
	})
	
	if (note) return true
	
	// 2. 检查是否有其他权限（如共享、公开等）
	// ... 根据实际业务逻辑
	
	return false
}
```

**修改 images.tsx loader：**

```typescript
// 修改后的 loader
export async function loader({ request }: Route.LoaderArgs) {
	const url = new URL(request.url)
	const searchParams = url.searchParams

	const objectKey = searchParams.get('objectKey')

	// 如果是通过 objectKey 访问，先检查权限
	if (objectKey) {
		const accessCheck = await checkImageAccess(request, objectKey)
		
		if (!accessCheck.allowed) {
			// 返回 404 而不是 403，避免泄露信息
			throw new Response('Not found', { status: 404 })
		}
		
		// 使用权限检查返回的缓存策略
		const headers = new Headers()
		headers.set(
			'Cache-Control',
			accessCheck.cacheControl ?? 'private, max-age=31536000',
		)

		return getImgResponse(request, {
			headers,
			allowlistedOrigins: [
				getDomainUrl(request),
				process.env.AWS_ENDPOINT_URL_S3,
			].filter(Boolean),
			cacheFolder: await getCacheDir(),
			getImgSource: () => {
				const { url: signedUrl, headers: signedHeaders } =
					getSignedGetRequestInfo(objectKey)
				return {
					type: 'fetch',
					url: signedUrl,
					headers: signedHeaders,
				}
			},
		})
	}

	// 处理其他来源（src 参数）的逻辑保持不变
	// ...
}
```

#### 方案二：使用签名 URL（短期有效）

**架构思路：**

```
页面渲染时
    ↓
为每个图片生成带签名的短期 URL（如 15 分钟）
    ↓
<img src="/resources/images?objectKey=xxx&sig=abc&exp=123456">
    ↓
请求时验证签名和过期时间
```

**实现：**

```typescript
// app/utils/signed-url.server.ts
import { createHmac } from 'crypto'

const SIGNING_SECRET = process.env.SESSION_SECRET ?? 'fallback-secret'
const DEFAULT_EXPIRY_SECONDS = 15 * 60 // 15 分钟

/**
 * 为图片 URL 生成签名
 */
export function signImageUrl(objectKey: string, expirySeconds?: number): {
	url: string
	expiresAt: number
} {
	const expiresAt = Math.floor(Date.now() / 1000) + (expirySeconds ?? DEFAULT_EXPIRY_SECONDS)
	
	// 创建签名基础
	const signatureBase = `${objectKey}:${expiresAt}`
	
	// 生成 HMAC 签名
	const signature = createHmac('sha256', SIGNING_SECRET)
		.update(signatureBase)
		.digest('hex')
	
	return {
		url: `/resources/images?objectKey=${encodeURIComponent(objectKey)}&exp=${expiresAt}&sig=${signature}`,
		expiresAt,
	}
}

/**
 * 验证图片 URL 签名
 */
export function verifyImageSignature(
	objectKey: string,
	expiresAt: number,
	signature: string,
): boolean {
	// 检查是否过期
	const now = Math.floor(Date.now() / 1000)
	if (now > expiresAt) {
		return false
	}
	
	// 重新计算签名进行比对
	const signatureBase = `${objectKey}:${expiresAt}`
	const expectedSignature = createHmac('sha256', SIGNING_SECRET)
		.update(signatureBase)
		.digest('hex')
	
	// 使用时序安全的比对
	return timingSafeEqual(
		Buffer.from(signature, 'hex'),
		Buffer.from(expectedSignature, 'hex'),
	)
}

import { timingSafeEqual } from 'crypto'
```

**修改 URL 生成函数：**

```typescript
// 注意：这些函数现在需要在服务端调用
export async function getUserImgSrcSigned(objectKey?: string | null) {
	if (!objectKey) return '/img/user.png'
	
	// 头像可以使用较长的过期时间，因为通常是公开的
	const { url } = signImageUrl(objectKey, 24 * 60 * 60) // 24 小时
	return url
}

export async function getNoteImgSrcSigned(objectKey: string) {
	// 笔记图片使用较短的过期时间
	const { url } = signImageUrl(objectKey, 15 * 60) // 15 分钟
	return url
}
```

**loader 验证：**

```typescript
// 在 images.tsx 中
export async function loader({ request }: Route.LoaderArgs) {
	const url = new URL(request.url)
	const searchParams = url.searchParams

	const objectKey = searchParams.get('objectKey')
	const expiresAt = searchParams.get('exp')
	const signature = searchParams.get('sig')

	if (objectKey) {
		// 必须有签名参数
		if (!expiresAt || !signature) {
			throw new Response('Not found', { status: 404 })
		}
		
		// 验证签名
		const isValid = verifyImageSignature(
			objectKey,
			parseInt(expiresAt, 10),
			signature
		)
		
		if (!isValid) {
			throw new Response('Not found', { status: 404 })
		}
		
		// 签名验证通过，继续处理
		// ...
	}
	
	// ...
}
```

#### 方案三：修改缓存策略（快速缓解）

**紧急修复：修改 Cache-Control 头**

```typescript
// 立即修改 images.tsx
export async function loader({ request }: Route.LoaderArgs) {
	const url = new URL(request.url)
	const searchParams = url.searchParams

	const headers = new Headers()
	
	// 改为 private，禁止 CDN 和共享缓存
	// 缩短缓存时间到 1 小时
	headers.set('Cache-Control', 'private, max-age=3600')

	// ... 其他逻辑
}
```

**缓存策略对比：**

| 策略 | 指令 | 安全性 | 性能 | 适用场景 |
|------|------|-------|------|---------|
| 当前（不安全） | `public, max-age=31536000, immutable` | ❌ 低 | ✅ 高 | 完全公开资源 |
| 改进方案 1 | `private, max-age=31536000` | ⚠️ 中 | ✅ 高 | 用户特定但不敏感 |
| 改进方案 2 | `private, max-age=3600` | ✅ 较高 | ⚠️ 中 | 敏感资源 |
| 改进方案 3 | `no-store` | ✅ 最高 | ❌ 低 | 高度敏感 |

### 3.5 综合建议

#### 推荐实施方案

**短期（立即实施）：**
1. ✅ 修改 `Cache-Control: public` 为 `private`
2. ✅ 缩短 `max-age` 到合理时间（如 1 小时）
3. ✅ 移除 `immutable` 指令

**中期（1-2 周）：**
1. 实现方案一的权限检查层
2. 区分不同资源类型的访问策略：
   - 用户头像：公开但 `private` 缓存
   - 笔记图片：检查笔记访问权限

**长期（1-2 月）：**
1. 考虑实现方案二的签名 URL 机制
2. 对于高度敏感的资源，使用短期签名
3. 实现资源访问日志和审计

#### 权限矩阵设计建议

| 资源类型 | 访问权限 | 缓存策略 | 签名要求 |
|---------|---------|---------|---------|
| 用户头像 | 公开（用户存在即可） | `private, max-age=86400` | 可选 |
| 公开笔记图片 | 公开 | `private, max-age=3600` | 可选 |
| 私有笔记图片 | 仅所有者/授权用户 | `private, max-age=3600` | 推荐 |
| 管理员资源 | 仅管理员 | `no-store` | 必需 |

#### 安全开发最佳实践

1. **对象键设计：**
   - 使用不可预测的 ID（当前的 cuid2 是好的选择）
   - 不要在 objectKey 中包含敏感信息
   - 保持命名规则一致，便于权限检查

2. **URL 处理：**
   - 始终验证输入格式
   - 不要直接拼接路径到存储请求
   - 考虑使用白名单验证 objectKey 格式

3. **日志和监控：**
   - 记录图片访问日志（但不要记录完整 URL）
   - 监控异常访问模式（如大量 404、异常 IP）
   - 建立告警机制

---

## 四、总结与行动清单

### 4.1 核心问题汇总

| 领域 | 问题 | 风险等级 | 优先级 |
|------|------|---------|--------|
| 文件校验 | 无文件类型实际验证，仅依赖 `file.type` | **高** | P0 |
| 对象回收 | 所有删除操作都不清理存储对象 | **高** | P0 |
| 缓存策略 | `Cache-Control: public` 导致权限泄露 | **高** | P0 |
| 读取权限 | 图片路由无任何权限检查 | **高** | P0 |
| 长期缓存 | `max-age=31536000` 权限撤销后仍可访问 | 中 | P1 |
| 成本增长 | 遗弃对象无限增加存储成本 | 中 | P1 |

### 4.2 立即行动清单（P0）

1. **修改缓存策略**（1 小时工作量）
   - [ ] 将 `Cache-Control: public` 改为 `private`
   - [ ] 缩短 `max-age` 到 3600 秒（1 小时）
   - [ ] 移除 `immutable` 指令

2. **评估当前风险**（2 小时工作量）
   - [ ] 检查存储桶中对象数量和大小
   - [ ] 识别已遗弃的对象（比较数据库和存储）
   - [ ] 评估私有资源暴露风险

3. **添加基础权限检查**（4 小时工作量）
   - [ ] 实现 objectKey 格式验证
   - [ ] 验证资源（用户/笔记）是否存在
   - [ ] 返回 404 而不是 403

### 4.3 短期行动清单（P1，1-2 周）

1. **实现对象清理机制**
   - [ ] 设计软删除或清理队列表
   - [ ] 修改删除操作记录清理任务
   - [ ] 实现定时清理任务

2. **完善权限检查系统**
   - [ ] 区分公开和私有资源
   - [ ] 实现笔记图片的访问权限检查
   - [ ] 添加访问日志

3. **添加文件类型验证**
   - [ ] 使用 `file-type` 库验证实际文件内容
   - [ ] 限制允许的图片类型
   - [ ] 考虑添加图片重新编码管道

### 4.4 长期优化建议（P2，1-2 月）

1. **签名 URL 机制**
   - [ ] 实现 HMAC 签名的 URL
   - [ ] 短期有效的访问令牌
   - [ ] 防止 URL 复用和篡改

2. **存储生命周期管理**
   - [ ] 配置存储服务端生命周期规则
   - [ ] 实现对象版本管理
   - [ ] 定期审计存储使用情况

3. **安全加固**
   - [ ] 实现资源访问速率限制
   - [ ] 添加异常访问检测
   - [ ] 定期安全审计

---

## 附录

### A. 相关代码位置

| 功能 | 文件路径 | 关键行 |
|------|---------|--------|
| 上传核心 | `app/utils/storage.server.ts` | 全程 |
| 头像上传 | `app/routes/settings/profile/photo.tsx` | 66-107 |
| 笔记图片上传 | `app/routes/users/$username/notes/+shared/note-editor.server.tsx` | 27-131 |
| 图片路由 | `app/routes/resources/images.tsx` | 28-80 |
| URL 生成 | `app/utils/misc.tsx` | 8-16 |
| 笔记删除 | `app/routes/users/$username/notes/$noteId.tsx` | 56-90 |
| Schema 定义 | `prisma/schema.prisma` | 51-76 |

### B. 工具和参考

- **文件类型验证**：[file-type](https://www.npmjs.com/package/file-type)
- **图片处理**：[sharp](https://www.npmjs.com/package/sharp)
- **AWS Signature V4**：[AWS 文档](https://docs.aws.amazon.com/general/latest/gr/signature-version-4.html)
- **S3 生命周期规则**：[AWS 文档](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
