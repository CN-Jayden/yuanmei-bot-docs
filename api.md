# 开发者 API

客户系统读取群消息、生成回复后发回群，或把群消息同步到自己的系统时使用。

所有接口都是 `POST` + JSON，鉴权统一用 `Authorization: Bearer <token>`。接入流程、回调验签和排错见 [开发者接入指南](developer.md)。

## 1. 版本与地址

当前 API 版本：`2026-10-26`。

| 项目 | 说明 |
| --- | --- |
| Base URL | `https://chatbot.apetd.com` |
| 推荐前缀 | `/api/v1/open/*` |
| 兼容前缀 | `/api/open/*`（与 v1 完全等价，继续兼容） |
| 版本响应头 | `X-Yuanmei-Api-Version: 2026-10-26`（两类前缀都会返回） |

版本探测（无需鉴权，GET）：

```http
GET https://chatbot.apetd.com/api/v1/open/version
```

```json
{
  "ok": true,
  "apiVersion": "2026-10-26",
  "basePath": "/api/v1/open",
  "legacyBasePath": "/api/open"
}
```

新增接口 / 新增字段不改变版本号；不兼容变更才会调整 `apiVersion`，旧前缀目前没有弃用计划。

## 2. 签发 Token 与权限

```text
个人中心 → API 接入 → 服务端发送 API → 签发 Token
```

签发时选择权限范围：

| 权限 | 能做什么 |
| --- | --- |
| `send` | 发送文本 / 图片 / 链接、上传图片、定时发送、创建 / 确认 / 取消任务 |
| `read` | 机器人列表、会话列表、读取消息、搜索消息、查询发送状态、任务列表、群成员 |

Token 长期有效、只完整显示一次、可随时撤销；数据库只保存哈希。旧版本签发的 Token 默认同时具备 `send` 与 `read`。再次签发会替换旧 Token，旧 Token 立即失效。

也可以调接口签发（浏览器登录态，仅工作空间所有者）：

```http
POST /api/api-token/issue
Content-Type: application/json

{"scopes": ["send", "read"]}
```

```json
{
  "ok": true,
  "token": "a1b2…（64 位十六进制，只显示这一次）",
  "prefix": "a1b2c3d4",
  "scopes": ["send", "read"]
}
```

## 3. 通用约定

- 请求体：`Content-Type: application/json`（图片上传例外，见第 11 节）；缺这个头会返回 `415`
- 成功：HTTP 2xx，返回 `{"ok": true, ...}`；新建发送 `202`、上传 `201`
- 失败：对应 HTTP 状态码 + `{"error": "错误原因"}`
- 限流：按来源 IP + 接口组每分钟计数——发送类 120 次、读取类 120 次、上传 60 次，超限返回 `429`
- 时间：ISO 8601 UTC，如 `2026-10-26T12:00:00.000Z`
- 会话 ID：群聊 `R:` 开头、私聊 `S:` 开头
- 测试空间签发的 Token 只能操作测试群，读消息时必须指定会话
- 未知字段会被忽略，不报错
- 请在服务端调用：浏览器跨域请求会带 `Origin` 头，平台会以 `403 请求来源不允许` 拒绝（未开放 CORS）

## 4. 发送消息

```http
POST https://chatbot.apetd.com/api/v1/open/send
Authorization: Bearer <token>
Content-Type: application/json
Idempotency-Key: order-123-notify

{
  "conversationId": "R:10949759080209864",
  "type": "text",
  "content": "课程价格为 199 元。"
}
```

文本：

```json
{
  "conversationId": "R:10949759080209864",
  "type": "text",
  "content": "课程价格为 199 元。",
  "atUserIds": ["wmXXXX"]
}
```

- `atUserIds` 仅群会话可用，最多 50 个
- 文本上限 2000 字

图片（必须公网可访问）：

```json
{
  "conversationId": "R:10949759080209864",
  "type": "image",
  "imageUrl": "https://example.com/a.png"
}
```

链接卡片：

```json
{
  "conversationId": "R:10949759080209864",
  "type": "link",
  "title": "课程介绍",
  "url": "https://example.com/course",
  "imageUrl": "https://example.com/cover.png",
  "desc": "12 节课，199 元"
}
```

- 标题上限 100 字，描述上限 500 字
- `type` 也可写成 `msgtype`，`conversationId` 也可写成 `conversation_id`

返回（HTTP `202`）：

```json
{
  "ok": true,
  "messageId": "...",
  "status": "pending",
  "conversationId": "R:10949759080209864"
}
```

`pending` 表示已入队，不代表已送达；最终结果用发送状态接口或 `message.sent` / `message.failed` / `message.uncertain` 回调确认。发送是异步队列，单账户待发送队列上限 100 条。

## 5. 幂等发送

同一请求重复提交时，用幂等键避免群里收到两条：

```http
Idempotency-Key: order-123-notify
```

也可以放在请求体：`{"idempotencyKey": "order-123-notify"}`。请求头优先。

- 幂等范围是单个工作空间，键最长 120 字符，有效期 24 小时
- 24 小时内相同幂等键返回第一次的 `messageId`，HTTP `200`，并带 `"duplicate": true`
- 重复返回的 `status` 是消息当前状态，不一定是 `pending`
- 建议用「业务单据 ID + 用途」做键；`schedule` 定时发送不支持幂等键

```json
{
  "ok": true,
  "duplicate": true,
  "messageId": "...",
  "status": "accepted",
  "conversationId": "R:10949759080209864"
}
```

## 6. 查询发送状态

```http
POST /api/v1/open/send/status
Authorization: Bearer <token>
Content-Type: application/json

{"messageId": "..."}
```

```json
{
  "ok": true,
  "messageId": "...",
  "status": "accepted",
  "conversationId": "R:10949759080209864",
  "createdAt": "2026-10-26T12:00:00.000Z",
  "source": "API 发送"
}
```

需要 `read` 权限。返回 `status`：

| 状态 | 含义 | 建议 |
| --- | --- | --- |
| `pending` | 已入队，等待发送 | 稍后再查 |
| `accepted` | 上游已受理 | 成功 |
| `failed` | 上游明确拒绝，可安全重发 | 排查原因后重发 |
| `uncertain` | 超时结果未知，重发前请人工确认 | 不要自动重发 |
| `blocked` | 账户授权或额度问题未发出 | 联系平台开通或充值 |
| `simulated` | 演示 / 模拟发送 | 仅演示环境 |

## 7. 机器人列表

```http
POST /api/v1/open/robots
Authorization: Bearer <token>
Content-Type: application/json

{}
```

返回已连接机器人的 `id` / `name` / `mode` / `status`，以及群聊与联系人数量：

```json
{
  "ok": true,
  "robots": [
    {
      "id": "...",
      "name": "课程顾问机器人",
      "mode": "live",
      "status": "bound",
      "groups": 3,
      "users": 12
    }
  ]
}
```

## 8. 会话列表

```http
POST /api/v1/open/conversations
Authorization: Bearer <token>
Content-Type: application/json

{"robotId": "..."}
```

`robotId` 可省略，默认用第一个机器人。返回 `groups`（群聊）与 `users`（联系人），每项含 `conversation_id` 与 `name`：

```json
{
  "ok": true,
  "robot": {"id": "...", "name": "课程顾问机器人"},
  "groups": [{"conversation_id": "R:10949759080209864", "name": "2026 秋季招生群"}],
  "users": [{"conversation_id": "S:wmXXXX", "name": "张老师"}]
}
```

## 9. 读取消息

```http
POST /api/v1/open/messages
Authorization: Bearer <token>
Content-Type: application/json

{
  "conversationId": "R:10949759080209864",
  "after": "2026-10-26T12:00:00.000Z",
  "direction": "all",
  "limit": 20
}
```

| 参数 | 说明 |
| --- | --- |
| `robotId` | 可选，默认第一个机器人 |
| `conversationId` | 可选；不传则读取该机器人全部会话；测试空间必须指定测试群 |
| `after` | 可选，只返回该时间之后的消息，用于增量轮询 |
| `direction` | `in` 只看客户消息、`out` 只看机器人消息、`all` 全部；默认 `all` |
| `limit` | 默认 20，范围 1–100 |

返回：

```json
{
  "ok": true,
  "messages": [
    {
      "id": "...",
      "conversationId": "R:...",
      "conversationName": "测试群",
      "direction": "in",
      "sender": "wmXXXX",
      "senderName": "张老师",
      "msgtype": "text",
      "content": "课程怎么收费",
      "imageUrl": "",
      "status": "received",
      "createdAt": "2026-10-26T12:00:00.000Z"
    }
  ],
  "nextAfter": "2026-10-26T12:00:00.000Z"
}
```

轮询时把 `nextAfter` 作为下一次的 `after`。带 `after` 时从最早未读开始按时间正序返回，避免一次超出 `limit` 漏消息；不带 `after` 时返回最近 N 条。

## 10. 搜索消息

```http
POST /api/v1/open/search
Authorization: Bearer <token>
Content-Type: application/json

{"query": "课程", "conversationId": "R:...", "limit": 20}
```

```json
{
  "ok": true,
  "query": "课程",
  "messages": [
    {
      "id": "...",
      "conversationId": "R:...",
      "conversationName": "测试群",
      "direction": "in",
      "sender": "wmXXXX",
      "senderName": "张老师",
      "msgtype": "text",
      "content": "课程怎么收费",
      "createdAt": "2026-10-26T12:00:00.000Z"
    }
  ]
}
```

关键词同时匹配正文与发言人昵称（大小写不敏感的子串），返回最近匹配的 N 条并按时间正序，`limit` 默认 20、上限 100；`query` 必填（最长 80 字）。

## 11. 上传图片

把本地图片上传成公网 URL，再交给发送接口：

```http
POST https://chatbot.apetd.com/api/v1/open/upload
Authorization: Bearer <token>
Content-Type: image/png

<图片原始字节>
```

- 支持 PNG / JPG / GIF / WebP，单张 ≤ 5MB
- 需要 `send` 权限
- 可选查询参数 `?robotId=...`，用于图床路径模板

返回：

```json
{"ok": true, "url": "https://.../uploads/up-xxx.png", "storage": "local", "bytes": 1234}
```

校验顺序：声明类型白名单 → 文件大小 → 文件魔数（改扩展名会被拒绝）。拿到 `url` 后作为发送接口的 `imageUrl`。

```js
const {url} = await bot.uploadImage(fs.readFileSync('poster.png'), {contentType: 'image/png'});
await bot.sendImage('R:xxx', url);
```

## 12. 任务跟踪

维护「到店 / 交稿 / 回访」等带截止时间的事项：到期前自动在群里提醒，责任人回复确认词自动完成。

### 列表

```http
POST /api/v1/open/tasks/list
Authorization: Bearer <token>
Content-Type: application/json

{"robotId": "...", "status": "pending", "limit": 20}
```

`status`：`pending`（默认）/ `confirmed` / `cancelled` / `expired` / `all`；`limit` 默认 20、上限 50。需要 `read` 权限。

### 创建

```http
POST /api/v1/open/tasks/create
Authorization: Bearer <token>
Content-Type: application/json

{
  "robotId": "...",
  "conversationId": "R:10949759080209864",
  "title": "回访张老师",
  "dueAt": "2026-10-26T13:00:00.000Z",
  "personName": "张老师",
  "personUserId": "wmXXXX",
  "confirmWords": "已到店,已完成",
  "remindBefore": 10,
  "repeatMinutes": 10,
  "maxReminds": 5,
  "note": "记得带资料"
}
```

- `dueAt` 需在 5 秒后至 30 天内
- 仅支持已同步的群会话（`R:` 开头），私聊不支持
- `confirmWords` 默认 `已到店`，最多 5 个、每个不超过 20 字
- 需要 `send` 权限
- 进行中任务最多 300 条

### 确认 / 取消

```http
POST /api/v1/open/tasks/confirm
{"taskId": "..."}

POST /api/v1/open/tasks/cancel
{"taskId": "..."}
```

确认后任务变为 `confirmed`，取消后为 `cancelled`；已结束的任务不能重复操作。两者都需要 `send` 权限。

## 13. 定时发送

预约未来 30 天内的一次发送（文本 / 图片 / 链接）：

```http
POST /api/v1/open/schedule
Authorization: Bearer <token>
Content-Type: application/json

{
  "robotId": "...",
  "conversationId": "R:10949759080209864",
  "type": "text",
  "content": "明天 9 点提醒家长交作业",
  "dueAt": "2026-10-27T01:00:00.000Z"
}
```

```json
{
  "ok": true,
  "messageId": "...",
  "status": "pending",
  "dueAt": "2026-10-27T01:00:00.000Z",
  "conversationId": "R:10949759080209864"
}
```

- `dueAt` 需在 5 秒后至 30 天内，越界返回 `400`
- 需要 `send` 权限
- 载荷与发送接口相同，支持文本 / 图片 / 链接

## 14. 群成员

查询客户群成员花名册（数据来自「企微同步」）：

```http
POST /api/v1/open/group-members
Authorization: Bearer <token>
Content-Type: application/json

{"robotId": "...", "conversationId": "R:10949759080209864", "limit": 100}
```

返回 `{ok, found, name, owner, count, fetchedAt, stale, truncated, members}`；没有同步过该群时 `found: false`，成员为空。需要 `read` 权限，`limit` 上限 500。

```json
{
  "ok": true,
  "conversationId": "R:10949759080209864",
  "found": true,
  "name": "2026 秋季招生群",
  "owner": "张老师",
  "count": 128,
  "fetchedAt": "2026-10-26T12:00:00.000Z",
  "stale": false,
  "truncated": false,
  "members": [
    {"userId": "wmXXXX", "name": "张老师", "kind": "member", "groupNickname": "", "joinTime": 1793017003}
  ]
}
```

## 15. 限制

- 只有工作空间所有者能签发 / 撤销 Token
- 发送受授权、额度、上游限速、队列限制（单账户待发送上限 100 条）
- 测试空间只能操作测试群：发送与读取都必须指定测试群会话
- 图片必须是公网可访问 URL 或先通过上传接口换 URL
- 消息记录用于最近检索，单账户只保留最近约 500 条；长期归档请使用事件回调落库

## 16. 错误码速查

| HTTP | 含义 | 常见原因 |
| --- | --- | --- |
| `400` | 参数错误 | 缺字段、超长、URL 非法、会话未同步、`dueAt` 越界、`msgtype` 不支持 |
| `401` | 未认证 | 没带 Token、Token 被撤销 / 替换、账户停用 |
| `403` | 无权限 | 账户未开通、Token 缺少 `send` / `read`、测试空间访问真实会话、浏览器跨域 `Origin` 被拒 |
| `404` | 不存在 | 机器人未绑定、`messageId` 不属于当前账户、路径拼错 |
| `413` | 内容过大 | 上传超过 5MB |
| `415` | 类型不支持 | 缺少 `Content-Type: application/json`、图片 MIME 不在白名单、文件魔数与声明不符 |
| `429` | 触发限流 | 超过每分钟配额，等待后重试 |
| `500` | 平台错误 | 记录时间与请求后联系平台 |

完整的错误文案、排查步骤见 [开发者接入指南 · 常见错误与排查](developer.md#8-常见错误与排查)。

## 17. 事件回调与签名校验

群消息到达、机器人消息最终状态、任务提醒 / 确认 / 到期时会推送到你的回调地址，共 7 种事件：

`message.received`、`message.sent`、`message.failed`、`message.uncertain`、`task.reminded`、`task.confirmed`、`task.expired`。

请求头包含 `X-Yuanmei-Event`、`X-Yuanmei-Timestamp`、`X-Yuanmei-Nonce`、`X-Yuanmei-Signature`。签名算法为 HMAC-SHA256，对 `` `${timestamp}.${原始请求体}` `` 计算，结果带 `sha256=` 前缀。

Node：

```js
import {createHmac, timingSafeEqual} from 'node:crypto';
function verify(secret, timestamp, rawBody, signature) {
  const expected = 'sha256=' + createHmac('sha256', secret).update(`${timestamp}.${rawBody}`).digest('hex');
  const a = Buffer.from(expected),
    b = Buffer.from(signature || '');
  return a.length === b.length && timingSafeEqual(a, b);
}
```

Python：

```python
import hashlib, hmac
def verify(secret, timestamp, raw_body, signature):
    expected = "sha256=" + hmac.new(secret.encode(), f"{timestamp}.{raw_body}".encode(), hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature or "")
```

- 用原始请求体验签，不要先反序列化再拼 JSON
- 校验时间戳在 5 分钟内、按 `eventId` 去重
- 2xx 响应体里的 `reply` 会自动回群；失败会重试，详见 [事件回调](callback.md)

完整的配置字段、7 种事件示例、reply / 重试 / 重放 / 过滤 / QPS / 超时说明见 [开发者接入指南 · 业务事件回调](developer.md#7-业务事件回调)。

## 18. 与 MCP 的关系

REST 与 MCP 共用同一套读取逻辑，过滤、分页和测试空间隔离语义一致。唯一的默认值差异：

- MCP `read_messages` 默认 `direction=in`，只返回客户消息，避免 Agent 读到自己发的消息
- REST `/api/v1/open/messages` 默认 `direction=all`，方便数据同步

需要一致行为时显式传 `direction` 即可。

| 场景 | 推荐方式 |
| --- | --- |
| AI 客户端（DeepSeek Harness / WorkBuddy / Hermes / OpenClaw） | MCP 读 + MCP 写 |
| 客户自建系统 / 低代码 / 数据同步 | REST 读 + REST 写 |
| 脚本 / cron / 只有 shell 的 Agent | 命令行 CLI |

## 19. 相关文档

- [开发者接入指南](developer.md)：接入流程、回调验签代码、错误排查
- [事件回调](callback.md)：回调配置与事件说明
- [MCP 接入](mcp.md)
- [命令行 CLI](cli.md)
- [常见问题](faq.md)
- Node / Python SDK：`integrations/sdk/`
- 桥接参考实现：https://github.com/CN-Jayden/yuanmei-bot-workspace/tree/main/integrations/agent-bridge
