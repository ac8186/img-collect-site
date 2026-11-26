# 小区物业管理问题汇总站（v3）

基于 Supabase 的静态站点：汇总收集小区物业管理问题——居民上传现场照片并填写问题描述，物业与居民都可按分类浏览已收录的记录。

v2 相比 v1 的变化：四类问题分类、问题描述字段、算术验证码 + IP 限流、匿名上传权限撤销（写入统一走 Edge Function）。

v3 相比 v2 的变化：新增管理后台 `/master/`（口令鉴权、筛选、搜索、批量删除、导出 CSV）；全站表述统一为「汇总收集物业管理问题」口径。

---

## 一、功能概览

**新增记录页 `upload.html`**

- 分类下拉固定为四条：停车管理、卫生问题、安全隐患、其他问题
- 问题描述必填，5–500 字，带实时字数统计
- 现场照片：点击或拖放选择，支持 JPG / PNG / WebP / GIF，单张 ≤ 10MB，非 GIF 自动压缩到最长边 1920px
- 算术验证码：一次性 token，5 分钟有效，可点「换一题」刷新，答错即作废
- 提交走 `submit` Edge Function（multipart），带真实上传进度与结果反馈
- 页面提示「图片中可能含他人隐私（如门牌号、车牌、人脸），请谨慎上传」

**浏览页 `browse.html`**

- 卡片展示：图片、分类标签、问题描述（3 行截断）、上传时间（本地时区，格式「月日 时分」）
- 按四类问题筛选（全部 / 停车管理 / 卫生问题 / 安全隐患 / 其他问题）
- 点击卡片打开灯箱：大图 + 完整描述 + 完整记录时间，支持左右方向键或按钮翻页、ESC 关闭
- 滚动分页（每页 24 条，按 `created_at` 倒序）

**入口页 `index.html`**：站点导航（新增记录 / 查看全部记录），页脚保留一个不显眼的管理入口链接指向 `master/`。

**管理后台 `master/index.html`**

- 路径 `/master/`，首次进入需输入管理口令（`ADMIN_TOKEN`），口令仅存于 `sessionStorage`，关闭标签页即失效
- 记录列表：缩略图、分类、问题描述、记录时间，顶部显示记录总数
- 四类筛选 + 按描述关键词搜索，分页每页 20 条
- 全选 / 单选后批量删除，删除前二次确认，同时删除 Storage 中的图片对象
- 导出当前筛选结果为 CSV（编号、分类、问题描述、记录时间、图片地址）
- 口令错误有明确提示并可重输；移动端可用

---

## 二、文件结构

```
output/
├── index.html                          # 入口导航页
├── upload.html                         # 新增记录页（内置 CONFIG）
├── browse.html                         # 浏览页（内置 CONFIG）
├── master/index.html                   # 管理后台（/master/，内置 CONFIG）
├── supabase_schema.sql                 # v1 初始库表（新项目需先执行）
├── migration_v2.sql                    # v2 迁移脚本（幂等，可重复执行）
├── supabase/functions/
│   ├── captcha/index.ts                # 验证码签发函数
│   ├── submit/index.ts                 # 记录提交函数（核心校验 + 写库）
│   └── admin/index.ts                  # 管理后台函数（口令校验 + 列表 + 批量删除）
└── README.md
```

---

## 三、部署步骤

### 1. 数据库

在 Supabase SQL Editor 中执行（按顺序）：

| 场景 | 需要执行的脚本 |
| --- | --- |
| 全新项目 | 先 `supabase_schema.sql`，再 `migration_v2.sql` |
| 已有 v1 库（已跑过 schema） | 只执行 `migration_v2.sql` |

`migration_v2.sql` 是幂等的，可重复执行，内容为：

1. `images` 增加 `description` 列，并加 `check` 约束（`null` 或 5–500 字）
2. 重建 `categories`：插入四条新分类（停车管理 10、卫生问题 20、安全隐患 30、其他问题 99），**把旧分类下的图片改挂到「其他问题」后删除旧五条**
3. 撤销 `anon` 对 `public.images` 与 `storage.objects` 的 `insert` 策略，**保留公开读策略**
   （`images_select_all`、`categories_select_all`、`uploads_select_all`）
4. 新建 `captcha_challenges`（token、题干、答案、是否已用、过期时间）
5. 新建 `upload_rate`（ip_hash、created_at）
6. 两张新表开启 RLS，且**不授予 anon 任何策略**，仅 `service_role` 可访问
7. `notify pgrst, 'reload schema'`，让 PostgREST 立即感知表结构变化

> 注意：`supabase_schema.sql` 中的 `images_insert_anon`、`uploads_insert_anon` 策略已由迁移脚本删除，因此 **v1 的前端直传方式在 v2 下会失败**，必须改用 `submit` 函数。

### 2. Edge Functions

```bash
supabase login
supabase link --project-ref jcaexhqpmbucaezwoczx

supabase functions deploy captcha --project-ref jcaexhqpmbucaezwoczx
supabase functions deploy submit  --project-ref jcaexhqpmbucaezwoczx
supabase functions deploy admin   --project-ref jcaexhqpmbucaezwoczx
```

环境变量：`SUPABASE_URL`、`SUPABASE_SERVICE_ROLE_KEY` 由平台自动注入，无需手工配置。

管理后台口令必须手工设置（**未设置时 `admin` 函数直接返回 `SERVER_MISCONFIGURED`**）：

```bash
supabase secrets set ADMIN_TOKEN="一个足够长的随机口令" --project-ref jcaexhqpmbucaezwoczx
```

可选：设置限流哈希盐，避免不同环境共享同一份 IP 指纹

```bash
supabase secrets set IP_HASH_SALT="你的随机字符串" --project-ref jcaexhqpmbucaezwoczx
```

三个函数默认会校验请求的 `apikey` / `Authorization`（前端传入 anon key），未配置 JWT 校验时可以直连。`admin` 函数在此基础上还要校验 `x-admin-token` 请求头。

### 3. 前端

`upload.html`、`browse.html`、`index.html` 是纯静态文件，放到任意静态托管（Vercel / Netlify / GitHub Pages / 对象存储）或本地直接打开即可，三个页面需在同一目录。

配置集中在两个页面 `<head>` 内的 `CONFIG` 区块，**已填好当前项目**，一般无需改动：

```js
const CONFIG = {
  SUPABASE_URL: 'https://jcaexhqpmbucaezwoczx.supabase.co',
  SUPABASE_ANON_KEY: '...',
  BUCKET: 'uploads',
  ...
};
```

---

## 四、数据模型

### images（v2 新增 description）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| id | bigserial | 主键 |
| category_id | bigint | 关联 categories.id，`on delete restrict` |
| file_path | text | 桶内对象路径，唯一，形如 `2026/11/3f9a...c1.jpg` |
| file_name | text | 原始文件名（已安全化） |
| mime_type | text | `image/jpeg` / `image/png` / `image/webp` / `image/gif` |
| file_size | bigint | 字节，`<= 10485760` |
| width / height | int | 预留，函数写入时为 null |
| **description** | text | **问题描述，5–500 字（v2 新增）** |
| created_at | timestamptz | 记录时间 |

### categories（v2 重建为四条）

| id | name | sort_order |
| --- | --- | --- |
| 1 | 停车管理 | 10 |
| 2 | 卫生问题 | 20 |
| 3 | 安全隐患 | 30 |
| 4 | 其他问题 | 99 |

### captcha_challenges（仅 service_role 可读写）

| 字段 | 说明 |
| --- | --- |
| id | 主键 |
| token | 32 位十六进制随机串，唯一 |
| question | 题干，如 `3 + 5 = ?`、`9 - 2 = ?` |
| answer | 答案（字符串） |
| used | 是否已使用（答错也置 true） |
| created_at | 签发时间 |
| expires_at | 过期时间（签发起 5 分钟） |

### upload_rate（仅 service_role 可读写）

| 字段 | 说明 |
| --- | --- |
| id | 主键 |
| ip_hash | `SHA-256(salt:IP)`，不落明文 IP |
| created_at | 记录时间（用于统计 1 小时窗口） |

---

## 五、接口说明

### `POST /functions/v1/captcha`

请求：空 body（`{}` 即可），需带 `apikey` 与 `Authorization: Bearer <anon key>`。

成功响应：

```json
{
  "ok": true,
  "token": "9f2c...1a",
  "question": "3 + 5 = ?",
  "expires_in": 300,
  "expires_at": "2026-11-26T08:45:00.000Z"
}
```

### `POST /functions/v1/submit`

请求：`multipart/form-data`

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| file | 是 | 图片文件（也接受字段名 `image`） |
| category_id | 是 | 分类 id（必须是有效且 `is_active = true` 的分类） |
| description | 是 | 问题描述，5–500 字（按码点计） |
| captcha_token | 是 | `captcha` 函数返回的 token |
| captcha_answer | 是 | 用户填写的计算结果 |

成功响应：

```json
{
  "ok": true,
  "image": {
    "id": 128,
    "category_id": 1,
    "category_name": "停车管理",
    "description": "3 号楼地下车库入口消防通道被占用",
    "file_path": "2026/11/3f9a1b2c3d4e.jpg",
    "file_name": "IMG_2031.jpg",
    "mime_type": "image/jpeg",
    "file_size": 486213,
    "created_at": "2026-11-26T08:40:12.345Z",
    "public_url": "https://jcaexhqpmbucaezwoczx.supabase.co/storage/v1/object/public/uploads/2026/11/3f9a1b2c3d4e.jpg"
  },
  "message": "提交成功，感谢您的反馈"
}
```

失败响应统一为 `{ "ok": false, "code": "...", "message": "中文提示" }`：

| code | HTTP | 含义 |
| --- | --- | --- |
| INVALID_DESCRIPTION | 400 | 描述字数不在 5–500 之间 |
| MISSING_FILE | 400 | 未收到图片文件 |
| EMPTY_FILE | 400 | 图片内容为空 |
| FILE_TOO_LARGE | 413 | 图片超过 10MB |
| INVALID_FILE_TYPE | 415 | 非 JPG / PNG / WebP / GIF |
| INVALID_CATEGORY | 400 | 分类 id 无效、不存在或已停用 |
| RATE_LIMITED | 429 | 同 IP 1 小时内超过 20 次 |
| CAPTCHA_REQUIRED | 400 | 缺少 token 或答案 |
| CAPTCHA_INVALID | 400 | token 不存在 |
| CAPTCHA_USED | 400 | token 已被使用 |
| CAPTCHA_EXPIRED | 400 | token 已过期（> 5 分钟） |
| CAPTCHA_WRONG | 400 | 答案错误（该 token 立即作废） |
| UPLOAD_FAILED | 500 | 写入 storage 失败 |
| DB_ERROR | 500 | 数据库操作失败（多为未执行迁移） |
| SERVER_MISCONFIGURED | 500 | 函数缺少环境变量 |
| METHOD_NOT_ALLOWED | 405 | 非 POST / OPTIONS 请求 |


### `POST /functions/v1/admin`

请求：JSON body，必须带 `x-admin-token: <ADMIN_TOKEN>`（同时带上 `apikey` 与 `Authorization: Bearer <anon key>`）。

#### action = list

```json
{ "action": "list", "category_id": 10, "q": "停车", "limit": 20, "offset": 0 }
```

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| category_id | 否 | 分类 id，省略或 `"all"` 表示全部 |
| q | 否 | 按 `description` 模糊匹配（`ilike`），最长 100 字 |
| limit | 否 | 每页条数，默认 20，最大 200 |
| offset | 否 | 偏移量，默认 0 |

成功响应：

```json
{
  "ok": true,
  "data": {
    "total": 137,
    "limit": 20,
    "offset": 0,
    "category_id": 10,
    "q": "停车",
    "records": [
      {
        "id": 128,
        "file_path": "2026/11/3f9a1b2c3d4e.jpg",
        "description": "3 号楼地下车库入口消防通道被占用",
        "category_id": 10,
        "category_name": "停车管理",
        "created_at": "2026-11-26T08:40:12.345Z",
        "public_url": "https://.../uploads/2026/11/3f9a1b2c3d4e.jpg"
      }
    ]
  },
  "message": "查询成功"
}
```

`total` 为当前筛选条件下的总条数，不受 `limit` / `offset` 影响。

#### action = delete

```json
{ "action": "delete", "ids": [128, 129] }
```

单次最多 100 条。执行顺序：先按 id 查 `file_path` → 删除 `uploads` 桶中的对象 → 删除 `images` 记录。**Storage 删除失败时不删库记录**，避免留下找不到文件的孤儿记录。

成功响应：

```json
{
  "ok": true,
  "data": {
    "requested": 2,
    "success_count": 2,
    "failed_count": 0,
    "deleted": [128, 129],
    "failed": []
  },
  "message": "成功删除 2 条，失败 0 条"
}
```

失败响应统一为 `{ "ok": false, "data": null, "code": "...", "message": "中文提示" }`：

| code | HTTP | 含义 |
| --- | --- | --- |
| UNAUTHORIZED | 401 | 缺少或错误的 `x-admin-token` |
| SERVER_MISCONFIGURED | 500 | 未配置 `ADMIN_TOKEN` 或缺少 Supabase 环境变量 |
| BAD_JSON | 400 | 请求体不是合法 JSON |
| BAD_ACTION | 400 | `action` 不是 `list` / `delete` |
| BAD_CATEGORY | 400 | `category_id` 不合法 |
| BAD_IDS | 400 | `ids` 为空或存在非正整数 |
| TOO_MANY_IDS | 400 | 单次删除超过 100 条 |
| QUERY_FAILED | 500 | 查询数据库失败 |
| METHOD_NOT_ALLOWED | 405 | 非 POST / OPTIONS 请求 |

---

## 六、安全模型

| 角色 | images | categories | storage.objects(uploads) | captcha_challenges / upload_rate |
| --- | --- | --- | --- | --- |
| anon（浏览器） | 只读 | 只读 | 只读（含公开 URL） | 无任何权限 |
| service_role（函数） | 读写 | 读写 | 读写 | 读写 |

- 浏览器端没有任何写权限：既不能直接 `insert` 表，也不能直接上传文件；所有写入必须经过 `submit`，从而强制经过验证码、限流、类型/大小/字数校验。
- 验证码一次性消费：答错即置 `used = true`，消费用 `update ... where used = false` 原子完成，并发下只有一次成功。
- 限流只存 IP 的哈希（`SHA-256(salt:IP)`），不保存明文 IP。
- 记录写入失败时，函数会自动删除已上传的对象，避免产生孤儿文件。
- 前端提示用户图片可能含他人隐私，请谨慎上传。
- 管理后台所有操作都经 `admin` 函数完成，浏览器不持有 `service_role` key；口令 `ADMIN_TOKEN` 只存在 Supabase Secrets 与服务端，前端仅在 `sessionStorage` 中暂存用户输入的口令，关闭标签页即失效。
- `admin` 页面带 `noindex, nofollow`，且未在站点导航中突出显示。

---

## 七、常见问题

**Q：页面提示「分类加载失败 / 未读到四类问题分类」**
未执行 `migration_v2.sql`，或执行后 PostgREST schema 缓存未刷新（脚本末尾已带 `notify pgrst`，仍异常可到 Dashboard 重启项目）。

**Q：提交报 `UPLOAD_FAILED`**
`uploads` 桶不存在或不是 public。检查 Storage 中是否存在 `uploads` 桶（file_size_limit 10MB，allowed_mime_types 含 jpeg/png/webp/gif）。

**Q：提交报 `DB_ERROR`**
多为未执行 `migration_v2.sql`（缺 `images.description` 列或两张新表）。按返回的 message 排查。

**Q：验证码一直加载失败**
`captcha` 函数未部署，或 `CONFIG.SUPABASE_URL` 填错；浏览器控制台会显示函数返回的错误。

**Q：换网络后仍提示 `RATE_LIMITED`**
限流按 `x-forwarded-for` 首个 IP 计算，同一出口 IP（如小区宽带、公司 NAT）共享 20 次/小时配额，等待窗口滑过后自动恢复。

**Q：上传大图很慢**
非 GIF 图片在前端已压缩到最长边 1920px、JPEG 质量 0.82，通常只有几百 KB。若仍慢，检查上行带宽或减少原图体积。

**Q：访问 `/master/` 提示「管理口令不正确」或一直 401**
未执行 `supabase secrets set ADMIN_TOKEN=...`，或输入的口令与服务端不一致。改完 secrets 无需重新部署函数，下次请求即生效。

**Q：后台列表里缩略图不显示**
`uploads` 桶不是 public，或记录的 `file_path` 已不存在。公开读策略 `uploads_select_all` 必须保留。

**Q：想修改分类**
修改 `migration_v2.sql` 中的四条分类名称/排序后重跑；同时同步两个页面 `CONFIG.CATEGORY_ORDER` 数组（决定下拉与筛选顺序）。
