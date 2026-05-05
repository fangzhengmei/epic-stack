# Epic Stack 资产存储深度分析报告（R3 - 事实校正版）

## 重要校正说明

本报告基于仓库**现有代码实现**进行事实校正，主要修正点：

1. **关于"私有笔记"的推断修正**：
   - ❌ 上一版报告推断存在"公开/私有"笔记概念
   - ✅ **实际情况**：Schema 中无 `isPublic`/`visibility` 字段，所有笔记在设计上都是公开访问的

2. **关于权限检查的事实澄清**：
   - 笔记列表、笔记详情页**无任何权限检查**
   - 仅编辑、删除操作有基础权限检查

---

## 一、现有权限模型事实核查

### 1.1 数据库 Schema 核查

**Prisma Schema 中 Note 模型：**
```prisma
// prisma/schema.prisma:32-49
model Note {
  id      String @id @default(cuid())
  title   String
  content String

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  owner   User   @relation(fields: [ownerId], references: [id], onDelete: Cascade, onUpdate: Cascade)
  ownerId String

  images NoteImage[]

  // ⚠️ 没有 isPublic、visibility 或类似字段
  // ⚠️ 所有笔记在数据模型层面都是"公开"的
}
```

**关键结论**：**不存在"公开/私有"笔记的概念**。这是上一版报告的错误推断。

### 1.2 各页面权限检查实际情况

#### 页面 1：笔记列表页 (`_layout.tsx`)

**代码位置**：`app/routes/users/$username/notes/_layout.tsx:11-26`

```typescript
export async function loader({ params }: Route.LoaderArgs) {
	const owner = await prisma.user.findFirst({
		select: {
			id: true,
			name: true,
			username: true,
			image: { select: { objectKey: true } },
			notes: { select: { id: true, title: true } },  // ⚠️ 获取用户所有笔记
		},
		where: { username: params.username },
	})

	invariantResponse(owner, 'Owner not found', { status: 404 })

	return { owner }
}
```

**权限检查情况**：

| 检查项 | 是否存在 | 代码位置 |
|--------|---------|---------|
| 用户登录检查 | ❌ 无 | - |
| 笔记权限检查 | ❌ 无 | - |
| 所有者验证 | ❌ 无 | - |

**实际行为**：
- 任何人（包括未登录用户）可以访问 `/users/{username}/notes`
- 可以看到该用户的**所有笔记列表**（ID + 标题）
- 同时暴露用户头像的 `objectKey`

#### 页面 2：笔记详情页 (`$noteId.tsx`)

**代码位置**：`app/routes/users/$username/notes/$noteId.tsx:24-49`

```typescript
export async function loader({ params }: Route.LoaderArgs) {
	const note = await prisma.note.findUnique({
		where: { id: params.noteId },  // ⚠️ 仅按 ID 查询
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
					objectKey: true,  // ⚠️ 返回图片 objectKey
				},
			},
		},
	})

	invariantResponse(note, 'Not found', { status: 404 })

	const date = new Date(note.updatedAt)
	const timeAgo = formatDistanceToNow(date)

	return { note, timeAgo }
}
```

**权限检查情况**：

| 检查项 | 是否存在 | 代码位置 |
|--------|---------|---------|
| 用户登录检查 | ❌ 无 | - |
| 笔记所有者检查 | ❌ 无 | - |
| 任何形式的权限验证 | ❌ 无 | - |

**实际行为**：
- 任何人只要知道 `noteId` 就可以访问笔记详情
- 获取完整的笔记内容：标题、正文、所有图片的 `objectKey`
- 无任何访问限制

#### 页面 3：笔记编辑页 (`$noteId_.edit.tsx`)

**代码位置**：`app/routes/users/$username/notes/$noteId_.edit.tsx:10-31`

```typescript
export async function loader({ params, request }: Route.LoaderArgs) {
	const userId = await requireUserId(request)  // ✅ 需要登录
	const note = await prisma.note.findFirst({
		select: {
			id: true,
			title: true,
			content: true,
			images: {
				select: {
					id: true,
					altText: true,
					objectKey: true,
				},
			},
		},
		where: {
			id: params.noteId,
			ownerId: userId,  // ✅ 必须是笔记所有者
		},
	})
	invariantResponse(note, 'Not found', { status: 404 })
	return { note }
}
```

**权限检查情况**：

| 检查项 | 是否存在 | 代码位置 |
|--------|---------|---------|
| 用户登录检查 | ✅ 有 | `requireUserId(request)` |
| 笔记所有者检查 | ✅ 有 | `ownerId: userId` |

**实际行为**：
- 需要登录
- 只能编辑自己的笔记
- 这是**唯一有权限检查**的笔记相关页面

#### 页面 4：笔记删除操作 (`$noteId.tsx` action)

**代码位置**：`app/routes/users/$username/notes/$noteId.tsx:56-89`

```typescript
export async function action({ request }: Route.ActionArgs) {
	const userId = await requireUserId(request)  // ✅ 需要登录
	const formData = await request.formData()
	// ... 表单解析

	const { noteId } = submission.value

	const note = await prisma.note.findFirst({
		select: { id: true, ownerId: true, owner: { select: { username: true } } },
		where: { id: noteId },
	})
	invariantResponse(note, 'Not found', { status: 404 })

	const isOwner = note.ownerId === userId
	await requireUserWithPermission(
		request,
		isOwner ? `delete:note:own` : `delete:note:any`,  // ✅ 权限检查
	)

	await prisma.note.delete({ where: { id: note.id } })
	// ...
}
```

**权限检查情况**：

| 检查项 | 是否存在 | 代码位置 |
|--------|---------|---------|
| 用户登录检查 | ✅ 有 | `requireUserId(request)` |
| 权限验证 | ✅ 有 | `requireUserWithPermission` |
| 所有者判断 | ✅ 有 | `isOwner = note.ownerId === userId` |

### 1.3 权限模型总结

| 资源/操作 | 访问权限 | 权限检查 |
|----------|---------|---------|
| 用户列表 (`/users`) | 🌍 完全公开 | ❌ 无 |
| 用户资料 (`/users/{username}`) | 🌍 完全公开 | ❌ 无 |
| 用户头像 | 🌍 完全公开 | ❌ 无 |
| 笔记列表 (`/users/{username}/notes`) | 🌍 完全公开 | ❌ 无 |
| 笔记详情 (`/users/{username}/notes/{noteId}`) | 🌍 完全公开 | ❌ 无 |
| 笔记图片 | 🌍 完全公开 | ❌ 无 |
| 笔记编辑 | 🔒 仅所有者 | ✅ 有检查 |
| 笔记删除 | 🔒 需权限 | ✅ 有检查 |
| 头像上传/更新 | 🔒 需登录 | ✅ 有检查 |

**关键设计事实**：
- **这是一个"公开笔记"应用**的设计
- 所有笔记内容（包括图片）都是公开可访问的
- 权限检查仅用于**修改操作**（编辑、删除），而非**读取操作**
- 上一版报告中关于"私有笔记越权访问"的描述**不符合实际代码实现**

---

## 二、真实威胁模型与可利用前提

### 2.1 威胁模型框架

根据实际代码实现，重新分析威胁：

```
┌─────────────────────────────────────────────────────────────────┐
│                        威胁分类框架                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  【已存在风险】- 无需额外条件，当前代码即可被利用                │
│  ├── 1. 文件类型无实际校验                                       │
│  ├── 2. 存储对象无回收机制                                       │
│  ├── 3. 缓存配置可能导致意外暴露                                 │
│  └── 4. 笔记 ID 可通过公开渠道获取                               │
│                                                                 │
│  【条件性风险】- 需要特定条件或额外攻击面                        │
│  └── (原"私有笔记越权"实际不存在，因为所有笔记都是公开的)        │
│                                                                 │
│  【设计预期行为】- 按设计就是公开的，非安全问题                  │
│  ├── 笔记内容公开可访问                                          │
│  ├── 笔记列表公开可访问                                          │
│  └── 图片 URL 公开可访问                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 已存在风险（可直接利用）

#### 风险 1：文件类型无实际校验

**代码位置**：`app/utils/storage.server.ts:150-167`

```typescript
function getSignedPutRequestInfo(file: File | FileUpload, key: string) {
	const uploadDate = new Date().toISOString()
	const { url, baseHeaders } = getBaseSignedRequestInfo({
		method: 'PUT',
		key,
		contentType: file.type,  // ⚠️ 直接使用客户端提供的 file.type
		uploadDate,
	})

	return {
		url,
		headers: {
			...baseHeaders,
			'Content-Type': file.type,  // ⚠️ 信任客户端值
			'X-Amz-Meta-Upload-Date': uploadDate,
		},
	}
}
```

**后端 Schema 校验：**
```typescript
// app/routes/settings/profile/photo.tsx:35-44
const NewImageSchema = z.object({
	intent: z.literal('submit'),
	photoFile: z
		.instanceof(File)
		.refine((file) => file.size > 0, 'Image is required')
		.refine(
			(file) => file.size <= MAX_SIZE,
			'Image size must be less than 3MB',
		),  // ⚠️ 仅检查大小，无类型校验
})
```

**可利用前提**：✅ **无需额外条件**

| 条件 | 是否需要 | 说明 |
|------|---------|------|
| 登录账号 | ✅ 需要 | 需要登录才能上传 |
| 特殊工具 | ❌ 不需要 | 普通浏览器即可 |
| 内部知识 | ❌ 不需要 | 公开 API |

**利用方式**：
1. 登录账号（注册或登录现有账号）
2. 将恶意文件重命名为 `.jpg` 或其他图片扩展名
3. 上传到头像或笔记图片
4. 系统会信任 `file.type` 并存储

**实际影响分析**：

由于以下原因，实际风险**相对较低**：

| 因素 | 说明 |
|------|------|
| 存储位置 | 文件存储在对象存储，非应用服务器执行环境 |
| 访问方式 | 通过 `/resources/images` 代理访问，被 `openimg` 处理 |
| MIME 类型 | 返回时根据 `Content-Type` 处理，但 `openimg` 会尝试解析为图片 |

**仍需关注的场景**：
- 如果存储桶配置为静态网站托管，可能直接执行脚本
- SVG 文件可能包含恶意脚本（XSS 风险）
- 大文件或异常格式可能导致 `openimg` 处理出错

---

#### 风险 2：存储对象无回收机制

**代码核查**：

| 操作场景 | 数据库操作 | 存储服务操作 |
|---------|-----------|-------------|
| 更新用户头像 | `deleteMany` + `create` | ❌ 无 |
| 删除用户头像 | `deleteMany` | ❌ 无 |
| 更新笔记图片 | 新 `objectKey`，旧记录 `deleteMany` | ❌ 无 |
| 删除笔记图片 | `deleteMany` | ❌ 无 |
| 删除笔记 | 级联删除 `NoteImage` | ❌ 无 |
| 删除用户 | 级联删除所有关联 | ❌ 无 |

**代码证据 - 删除笔记时：**
```typescript
// app/routes/users/$username/notes/$noteId.tsx:83
await prisma.note.delete({ where: { id: note.id } })
// ⚠️ 仅删除数据库记录，存储中的图片文件完全未处理
```

**Schema 级联定义：**
```prisma
// prisma/schema.prisma:59-60
note   Note   @relation(fields: [noteId], references: [id], onDelete: Cascade, onUpdate: Cascade)
// ⚠️ 仅数据库级联，不涉及存储服务
```

**可利用前提**：✅ **无需额外条件**（这是设计缺陷，非攻击利用）

**实际影响**：

| 影响类型 | 严重程度 | 说明 |
|---------|---------|------|
| 存储成本增长 | 🟡 中 | 遗弃对象持续累积 |
| 合规风险 | 🟡 中 | 用户删除数据后，存储中仍有残留 |
| 管理复杂度 | 🟡 中 | 无法准确知道存储了哪些有效数据 |

**成本估算（1000 活跃用户）：**

| 时间 | 遗弃数据量 | 年存储成本（$0.02/GB） |
|------|-----------|----------------------|
| 1 年 | ~246 GB | ~$59 |
| 3 年 | ~738 GB | ~$177 |
| 5 年 | ~1.2 TB | ~$295 |

---

#### 风险 3：缓存配置可能导致意外暴露

**代码位置**：`app/routes/resources/images.tsx:32-33`

```typescript
const headers = new Headers()
headers.set('Cache-Control', 'public, max-age=31536000, immutable')
```

**配置分析**：

| 指令 | 含义 | 潜在影响 |
|------|------|---------|
| `public` | 允许任何缓存（CDN、代理、浏览器） | 共享缓存可能存储敏感内容 |
| `max-age=31536000` | 缓存 1 年 | 长时间内无法撤销 |
| `immutable` | 资源不会改变，无需重新验证 | 完全禁止缓存验证 |

**可利用前提**：⚠️ **条件性风险**

| 条件 | 是否需要 | 说明 |
|------|---------|------|
| CDN/代理缓存 | ⚠️ 可能需要 | 取决于部署配置 |
| 公开访问 | ✅ 已满足 | 按设计就是公开的 |

**实际风险评估**：

由于**所有笔记本身就是公开的**，这个配置的风险**相对较低**：

| 场景 | 实际风险 |
|------|---------|
| 笔记图片被 CDN 缓存 | ✅ 按设计预期 |
| 用户删除笔记后图片仍可访问 | ⚠️ 是问题，但不是"越权" |
| 未授权用户访问缓存内容 | ✅ 按设计，未授权用户本就可以直接访问 |

**需要关注的情况**：
- 如果未来实现了私有笔记功能，这个配置会成为问题
- 用户删除账户后，头像图片可能仍在 CDN 缓存中

---

#### 风险 4：笔记 ID 可通过公开渠道获取

**代码证据 - 笔记列表公开：**
```typescript
// app/routes/users/$username/notes/_layout.tsx:18
notes: { select: { id: true, title: true } },
// ⚠️ 任何人可以获取用户的所有笔记 ID
```

**可利用前提**：✅ **无需额外条件**

**利用方式**：
1. 访问 `/users` 或搜索用户
2. 点击进入用户笔记列表 `/users/{username}/notes`
3. 获取该用户的所有笔记 ID
4. 直接访问笔记详情和图片

**实际风险评估**：
- ✅ **按设计预期行为**，非安全漏洞
- 笔记列表和详情页都是公开的，这是应用的设计选择

---

### 2.3 条件性风险（需要特定场景）

#### 原"私有笔记越权"风险校正

**上一版描述**：
> 私有笔记图片可被任意访问（越权漏洞）

**事实校正**：

| 检查项 | 实际情况 |
|--------|---------|
| 是否存在"私有笔记"概念 | ❌ 不存在，Schema 无相关字段 |
| 笔记详情页是否有权限检查 | ❌ 无任何检查 |
| 笔记是否设计为公开 | ✅ 是的，所有笔记都是公开的 |

**结论**：
- 这**不是**一个"越权漏洞"
- 这是**按设计的公开访问**
- 如果未来需要实现私有笔记，需要：
  1. 添加 `isPublic` 或类似字段到 Schema
  2. 在笔记详情页 loader 添加权限检查
  3. 在图片路由添加权限检查

---

### 2.4 威胁模型总结表

| 风险 ID | 描述 | 风险类型 | 可利用性 | 实际影响 |
|---------|------|---------|---------|---------|
| R1 | 文件类型无实际校验 | 已存在风险 | ✅ 登录即可 | 🟡 中（存储非预期文件类型） |
| R2 | 存储对象无回收机制 | 已存在风险 | ✅ 设计缺陷 | 🟡 中（成本、合规） |
| R3 | 缓存配置 `public` | 条件性风险 | ⚠️ 需 CDN | 🟢 低（当前设计下） |
| R4 | 笔记 ID 可枚举获取 | 设计预期 | ✅ 公开访问 | ✅ 按设计 |

**重要区分**：

| 类别 | 包含内容 | 处理建议 |
|------|---------|---------|
| **安全漏洞** | 文件类型校验缺失 | 需要修复 |
| **设计缺陷** | 存储对象无回收 | 需要架构改进 |
| **配置风险** | 缓存 `public` 指令 | 注意未来扩展性 |
| **设计预期** | 笔记公开访问 | 无需修复（按设计） |

---

## 三、文件类型校验机制深度分析

### 3.1 当前实现核查

**前端限制**：
```typescript
// 头像上传
<input accept="image/*" ... />

// 笔记图片
<input accept="image/*" ... />
```

**后端校验**：
```typescript
// 仅检查文件大小
.refine((file) => file.size <= MAX_UPLOAD_SIZE, ...)

// 直接信任 file.type
contentType: file.type,
```

### 3.2 校验能力对比表

| 校验项 | 前端限制 | 后端校验 | 是否可绕过 |
|--------|---------|---------|-----------|
| 文件扩展名 | ⚠️ `accept="image/*"` | ❌ 无 | ✅ 可绕过（重命名） |
| MIME 类型 | ⚠️ 浏览器嗅探 | ❌ 仅信任 `file.type` | ✅ 可绕过（伪造请求） |
| 文件大小 | ❌ 无 | ✅ `file.size` | ❌ 不可绕过 |
| 魔数检查 | ❌ 无 | ❌ 无 | - |
| 实际内容解析 | ❌ 无 | ❌ 无 | - |

### 3.3 攻击场景分析

**场景 1：文件扩展名欺骗**
```
攻击者文件: malware.php
重命名为: malware.jpg
前端 accept="image/*": ✅ 接受
后端检查: ✅ 通过（仅检查 size）
实际存储: PHP 文件，Content-Type: image/jpeg
```

**场景 2：SVG XSS 攻击**
```svg
<?xml version="1.0" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg version="1.1" baseProfile="full" xmlns="http://www.w3.org/2000/svg">
  <polygon id="triangle" points="0,0 0,50 50,0" fill="#009900" stroke="#004400"/>
  <script type="text/javascript">
    alert('XSS');  // ⚠️ 如果 SVG 直接渲染，可能执行
  </script>
</svg>
```

**场景 3：图片炸弹/拒绝服务**
```
特制图片文件:
- 极小文件头，极大声称尺寸
- 解压炸弹（如特制 PNG）
- 可能导致图片处理库内存耗尽
```

### 3.4 实际风险评估

考虑到当前架构，实际风险相对可控：

| 攻击场景 | 成功概率 | 原因 |
|---------|---------|------|
| 脚本执行 | 低 | 通过 `openimg` 代理，不直接执行 |
| 存储恶意内容 | 中 | 文件确实会被存储 |
| 服务端 DoS | 低 | `openimg` 应该有防护 |
| 内容欺骗 | 中 | 存储非图片内容 |

---

## 四、改进建议（基于事实）

### 4.1 高优先级建议

#### 建议 1：添加文件类型实际校验

**实施方案**：

```typescript
// 新增 app/utils/file-validation.server.ts
import { fileTypeFromBuffer } from 'file-type'
import { Readable } from 'node:stream'
import { buffer } from 'node:stream/consumers'

const ALLOWED_IMAGE_TYPES = new Set([
	'image/jpeg',
	'image/png',
	'image/gif',
	'image/webp',
	// 考虑是否允许 SVG（有 XSS 风险）
	// 'image/svg+xml',
])

export interface FileValidationResult {
	isValid: boolean
	detectedType?: string
	error?: string
}

/**
 * 验证图片文件的实际类型（通过魔数检查）
 */
export async function validateImageFile(
	file: File | { stream: () => NodeJS.ReadableStream; size: number },
	maxSize: number = 3 * 1024 * 1024,
): Promise<FileValidationResult> {
	// 1. 大小检查（已存在，但作为防御性检查）
	if (file.size > maxSize) {
		return { isValid: false, error: `File size exceeds ${maxSize} bytes` }
	}
	if (file.size === 0) {
		return { isValid: false, error: 'File is empty' }
	}

	// 2. 读取文件开头进行魔数检查
	let fileBuffer: Buffer
	try {
		if (file instanceof File) {
			const arrayBuffer = await file.arrayBuffer()
			fileBuffer = Buffer.from(arrayBuffer)
		} else {
			// 对于流式文件，读取前 4100 字节（足够检测文件类型）
			fileBuffer = await readStreamPreview(file.stream(), 4100)
		}
	} catch (error) {
		return { isValid: false, error: 'Failed to read file' }
	}

	// 3. 检测文件类型
	const type = await fileTypeFromBuffer(fileBuffer)

	if (!type) {
		// 无法检测类型，可能是未知格式
		return { isValid: false, error: 'Unknown file type' }
	}

	// 4. 验证是否在允许列表中
	if (!ALLOWED_IMAGE_TYPES.has(type.mime)) {
		return {
			isValid: false,
			detectedType: type.mime,
			error: `File type ${type.mime} is not allowed`,
		}
	}

	return {
		isValid: true,
		detectedType: type.mime,
	}
}

/**
 * 从流中读取前 N 个字节
 */
async function readStreamPreview(
	stream: NodeJS.ReadableStream,
	bytes: number,
): Promise<Buffer> {
	const chunks: Buffer[] = []
	let totalBytes = 0

	for await (const chunk of stream) {
		const bufferChunk = Buffer.isBuffer(chunk) ? chunk : Buffer.from(chunk)
		chunks.push(bufferChunk)
		totalBytes += bufferChunk.length

		if (totalBytes >= bytes) {
			break
		}
	}

	return Buffer.concat(chunks)
}

/**
 * SVG 内容净化（如果允许 SVG 上传）
 */
export function sanitizeSvg(svgContent: string): string {
	// 移除脚本标签和事件处理属性
	return svgContent
		.replace(/<script[\s\S]*?<\/script>/gi, '')
		.replace(/\son\w+=["']?[^"'>]*["']?/gi, '')
		.replace(/javascript:/gi, '')
}
```

**使用示例**：
```typescript
// 在上传前添加校验
const validation = await validateImageFile(file, MAX_UPLOAD_SIZE)
if (!validation.isValid) {
	throw new Error(validation.error ?? 'Invalid file')
}

// 使用检测到的实际类型，而不是 file.type
const actualType = validation.detectedType ?? file.type
```

---

#### 建议 2：实现存储对象回收机制

**方案 A：软删除 + 定时清理（推荐）**

**Step 1：扩展 Schema**
```prisma
model DeletedStorageObject {
  id          String   @id @default(cuid())
  objectKey   String   @unique
  modelType   String   // 'UserImage' | 'NoteImage'
  originalId  String   // 原记录的 ID
  deletedAt   DateTime @default(now())

  @@index([deletedAt])
}
```

**Step 2：修改删除操作**
```typescript
// 在 photo.tsx 中
if (intent === 'delete') {
	// 先获取要删除的记录
	const oldImages = await prisma.userImage.findMany({
		where: { userId },
		select: { id: true, objectKey: true },
	})

	// 记录到删除队列
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

**Step 3：实现清理任务**
```typescript
// app/utils/storage-cleanup.server.ts
import { prisma } from './db.server.ts'
import { getBaseSignedRequestInfo } from './storage.server.ts'

const CLEANUP_DELAY_HOURS = 24 // 延迟 24 小时删除

export async function cleanupDeletedObjects() {
	const cutoffTime = new Date(Date.now() - CLEANUP_DELAY_HOURS * 60 * 60 * 1000)

	const objectsToDelete = await prisma.deletedStorageObject.findMany({
		where: { deletedAt: { lte: cutoffTime } },
		take: 100,
	})

	if (objectsToDelete.length === 0) {
		return { deleted: 0 }
	}

	let deletedCount = 0
	const errors: string[] = []

	for (const obj of objectsToDelete) {
		try {
			await deleteFromStorage(obj.objectKey)
			await prisma.deletedStorageObject.delete({
				where: { id: obj.id },
			})
			deletedCount++
		} catch (error) {
			errors.push(`Failed to delete ${obj.objectKey}`)
		}
	}

	return { deleted: deletedCount, errors, total: objectsToDelete.length }
}

async function deleteFromStorage(key: string) {
	// 需要扩展 storage.server.ts 实现 DELETE 方法
	const { url, headers } = getBaseSignedRequestInfo({
		method: 'DELETE' as any,
		key,
	})

	const response = await fetch(url, {
		method: 'DELETE',
		headers,
	})

	if (!response.ok) {
		throw new Error(`Storage delete failed: ${response.status}`)
	}
}
```

**Step 4：触发清理任务**
```typescript
// 在上传操作后异步触发
export async function uploadProfileImage(userId: string, file: File | FileUpload) {
	// ... 现有上传逻辑

	// 异步触发清理（不等待）
	cleanupDeletedObjects().catch((error) => {
		console.error('Storage cleanup failed:', error)
	})

	return key
}
```

**方案 B：存储生命周期规则（临时缓解）**

如果无法立即修改代码，可以在存储服务端配置：

```json
{
  "Rules": [
    {
      "ID": "AutoExpireUnreferenced",
      "Prefix": "users/",
      "Status": "Enabled",
      "Expiration": {
        "Days": 90
      },
      "Tags": [
        {
          "Key": "cleanup",
          "Value": "auto"
        }
      ]
    }
  ]
}
```

**注意**：这只是缓解措施，无法区分"有效"和"遗弃"对象。

---

### 4.2 中优先级建议

#### 建议 3：评估缓存配置

**当前配置**：
```typescript
headers.set('Cache-Control', 'public, max-age=31536000, immutable')
```

**评估建议**：

| 场景 | 建议配置 | 理由 |
|------|---------|------|
| 当前（全部公开） | `public, max-age=86400` | 缩短缓存时间，允许更快更新 |
| 未来（私有笔记） | `private, max-age=86400` | 禁止共享缓存 |
| 用户头像 | `public, max-age=31536000, immutable` | 头像更新时会改变 URL |

**修改建议**：
```typescript
// 对于笔记图片，使用较短的缓存时间
headers.set('Cache-Control', 'public, max-age=86400')  // 1 天

// 对于头像，可以保持较长时间（因为更新会改变 objectKey）
// 但建议移除 immutable，以防需要紧急撤销
```

---

#### 建议 4：添加访问日志和监控

**实施建议**：

```typescript
// 在 images.tsx loader 中添加基础日志
export async function loader({ request }: Route.LoaderArgs) {
	const objectKey = searchParams.get('objectKey')

	// 基础日志（生产环境应使用更完善的日志系统）
	if (objectKey) {
		console.log(JSON.stringify({
			event: 'image_access',
			objectKey,
			// 注意：不要记录完整的用户信息，遵守隐私法规
			hasUser: Boolean(currentUser),
			timestamp: new Date().toISOString(),
		}))
	}

	// ... 现有逻辑
}
```

**监控要点**：
- 异常访问模式（如大量 404、异常 IP）
- 存储使用量趋势
- 清理任务执行情况

---

### 4.3 未来扩展性建议

#### 如果未来实现私有笔记

需要修改的组件：

| 组件 | 修改内容 |
|------|---------|
| Schema | 添加 `Note.isPublic` 字段 |
| 笔记列表 loader | 过滤非公开笔记（非所有者） |
| 笔记详情 loader | 检查权限（公开或所有者） |
| 图片路由 | 检查图片所属笔记的权限 |
| 缓存配置 | 改为 `private` |

**图片路由权限检查示例**：
```typescript
async function checkImageAccess(
	request: Request,
	objectKey: string,
): Promise<boolean> {
	// 解析 objectKey 路径
	const parts = objectKey.split('/')
	if (parts.length < 3 || parts[0] !== 'users') {
		return false
	}

	const resourceType = parts[2]

	switch (resourceType) {
		case 'profile-images':
			// 用户头像是公开的
			return true

		case 'notes':
			if (parts.length < 5) return false
			const noteId = parts[3]

			// 检查笔记权限
			const note = await prisma.note.findUnique({
				where: { id: noteId },
				select: { ownerId: true, isPublic: true },  // 假设添加了 isPublic
			})

			if (!note) return false
			if (note.isPublic) return true

			// 检查是否是所有者
			const currentUser = await optionalRequireUserId(request)
			return currentUser?.id === note.ownerId

		default:
			return false
	}
}
```

---

## 五、总结与行动清单

### 5.1 关键事实校正

| 上一版推断 | 实际事实 |
|-----------|---------|
| 存在"公开/私有"笔记概念 | ❌ Schema 无此字段，所有笔记都是公开的 |
| 私有笔记越权访问是漏洞 | ❌ 按设计，笔记本就是公开的 |
| 需要检查笔记访问权限 | ✅ 仅修改操作有权限检查，读取操作无 |

### 5.2 真实风险清单

| 风险 | 类型 | 优先级 |
|------|------|--------|
| 文件类型无实际校验 | 安全漏洞 | 🔴 高 |
| 存储对象无回收机制 | 架构缺陷 | 🟡 中 |
| 缓存配置 `public` | 配置风险 | 🟢 低（当前设计下） |

### 5.3 立即行动清单

#### 短期（1-3 天）

- [ ] 评估文件类型校验缺失的实际影响
- [ ] 检查当前存储使用量，估算遗弃对象比例
- [ ] 评估 SVG 上传风险，考虑是否禁用

#### 中期（1-2 周）

- [ ] 实现文件类型魔数检查（使用 `file-type` 库）
- [ ] 设计并实现软删除 + 清理队列机制
- [ ] 考虑缩短缓存时间（从 1 年改为 1 天）

#### 长期（按需）

- [ ] 如果需要私有笔记功能，规划权限模型
- [ ] 实现完整的审计日志和监控
- [ ] 考虑图片重新编码管道（彻底清除恶意内容）

### 5.4 关于"私有笔记"的澄清

如果 Epic Stack 未来计划实现私有笔记功能，需要：

1. **Schema 变更**：添加 `isPublic` 或类似字段
2. **读取权限检查**：在笔记列表、详情、图片路由添加权限验证
3. **缓存策略变更**：从 `public` 改为 `private`
4. **访问控制**：实现细粒度的共享/权限模型

**当前实现**：所有笔记都是公开的，这是设计选择，不是安全漏洞。

---

## 附录

### A. 权限检查代码位置索引

| 页面/操作 | 权限检查 | 文件位置 |
|----------|---------|---------|
| 笔记列表 | ❌ 无 | `_layout.tsx:11-26` |
| 笔记详情 | ❌ 无 | `$noteId.tsx:24-49` |
| 笔记编辑 | ✅ 有 | `$noteId_.edit.tsx:10-31` |
| 笔记删除 | ✅ 有 | `$noteId.tsx:56-89` |
| 图片路由 | ❌ 无 | `images.tsx:28-80` |

### B. 关键代码片段

#### 笔记详情页 loader（无权限检查）

```typescript
// app/routes/users/$username/notes/$noteId.tsx:24-49
export async function loader({ params }: Route.LoaderArgs) {
	const note = await prisma.note.findUnique({
		where: { id: params.noteId },  // 仅按 ID 查询，无权限检查
		select: {
			id: true,
			title: true,
			content: true,
			ownerId: true,
			images: { select: { objectKey: true } },
		},
	})
	// 返回给任何人
}
```

#### 图片路由（无权限检查）

```typescript
// app/routes/resources/images.tsx:44-52
getImgSource: () => {
	if (objectKey) {
		const { url: signedUrl, headers: signedHeaders } =
			getSignedGetRequestInfo(objectKey)
		return {
			type: 'fetch',
			url: signedUrl,
			headers: signedHeaders,
		}
	}
	// 无任何权限检查
}
```

#### 存储上传（仅信任 file.type）

```typescript
// app/utils/storage.server.ts:150-167
function getSignedPutRequestInfo(file: File | FileUpload, key: string) {
	return {
		url,
		headers: {
			'Content-Type': file.type,  // ⚠️ 信任客户端提供的值
		},
	}
}
```
