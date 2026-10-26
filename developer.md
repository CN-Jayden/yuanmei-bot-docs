# 开发者接入指南

面向客户自建系统 / CRM / 工单 / AI 服务的开发者。读完本文，你可以：

- 用 API Token 调用 REST 接口发送、读取、搜索群消息；
- 用 `Idempotency-Key` 保证重复请求不重复发群；
- 配置业务事件回调，校验 HMAC-SHA256 签名并自动回复；
- 按错误码快速定位 401 / 403 / 429 / 回调收不到等问题。

接口参考见 [开发者 API](api.md)，回调细节见 [事件回调](callback.md)。本文把两者串成一条完整的接入流程。

## 1. 十分钟接入

1. 平台已为你的工作空间绑定企业微信机器人，并在「机器人连接」里同步过群聊；
2. 用工作空间所有者账号登录，进入 `个人中心 → API 接入 → 服务端发送 API`，签发 Token（勾选 `send` / `read`）；
3. 调一次 `robots` 拿到 `robotId`，再调 `conversations` 拿到群/联系人 `conversationId`；
4. 调 `send` 发一条消息，或用 `send/status` 查状态；
5. 需要实时同步时，在 `个人中心 → API 接入 → 事件回调` 配置 HTTPS 地址，按第 7 节验签、去重、回复。

第一条请求（确认 Token 可用）：

```bash
curl -s https://chatbot.apetd.com/api/v1/open/version
# {"ok":true,"apiVersion":"2026-10-26","basePath":"/api/v1/open","legacyBasePath":"/api/open"}

curl -s https://chatbot.apetd.com/api/v1/open/robots \
  -H "Authorization: Bearer $YUANMEI_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}'
```

## 2. 通用约定

| 项目 | 约定 |
| --- | --- |
| Base URL | `https://chatbot.apetd.com` |
| 协议 | REST 接口一律 `POST` + `application/json`；唯一例外是图片上传（原始字节） |
| 鉴权 | `Authorization: Bearer <token>` |
| 成功 | HTTP 2xx + `{"ok": true, ...}`；`send` 新建返回 `202`，上传返回 `201`，其余 `200` |
| 失败 | 对应 HTTP 状态码 + `{"error": "中文原因"}` |
| 时间 | ISO 8601 UTC 字符串，如 `2026-10-26T12:00:00.000Z` |
| 标识 | `conversationId` 群聊以 `R:` 开头、私聊以 `S:` 开头；`messageId`、`robotId` 为字符串 |
| 限流 | 按**来源 IP + 接口组**每分钟计数：发送类 120 次、读取类 120 次、上传 60 次，超限 `429` |
| 测试空间 | 测试空间签发的 Token 只能操作测试群：发送、读取都必须指定测试群会话 |

请求体里同时接受 `conversationId` / `conversation_id`、`type` / `msgtype`，推荐统一用驼峰。未知字段会被忽略，不会报错。

> 请在**服务端**调用 API：浏览器跨域请求会带 `Origin` 头，平台会以 `403 请求来源不允许` 拒绝（未开放 CORS）。前端页面应调用你自己的后端，再由后端调用猿媒BOT。

## 3. API 版本与兼容

当前 API 版本：`2026-10-26`。

- **新接入使用 `/api/v1/open/*`**，例如 `POST /api/v1/open/send`；
- **旧地址 `/api/open/*` 继续兼容**，两者指向同一实现，行为完全一致；
- 两类地址的响应头都带：

  ```http
  X-Yuanmei-Api-Version: 2026-10-26
  ```

- 版本探测接口（无需鉴权，GET）：

  ```http
  GET https://chatbot.apetd.com/api/v1/open/version
  # 旧地址 GET /api/open/version 等价
  ```

  ```json
  {
    "ok": true,
    "apiVersion": "2026-10-26",
    "basePath": "/api/v1/open",
    "legacyBasePath": "/api/open"
  }
  ```

升级策略：新增接口、新增响应字段不改变版本号；只有不兼容变更才会调整 `apiVersion`。旧路径目前没有弃用计划，如需下线会提前在更新日志中说明。建议代码里把版本号做成配置，并记录响应头用于排查。

## 4. Token 与权限

### 4.1 签发

```text
个人中心 → API 接入 → 服务端发送 API → 签发 Token
```

- 只有**工作空间所有者**可以签发 / 撤销；
- 权限范围只有两种：`send`、`read`，可同时勾选；
- Token 是 64 位十六进制字符串，**只完整显示一次**，数据库只保存哈希，可随时撤销；
- 旧版本签发的 Token 没有权限记录，默认同时具备 `send` + `read`；
- 再次签发会替换旧 Token，旧 Token 立即失效。

也可以通过接口签发（浏览器登录态，不是 Bearer Token）：

```http
POST /api/api-token/issue
Content-Type: application/json

{"scopes": ["send", "read"]}
```

```json
{
  "ok": true,
  "token": "a1b2…（64 位）",
  "prefix": "a1b2c3d4",
  "scopes": ["send", "read"],
  "note": "Token 只完整显示这一次，请立即保存"
}
```

### 4.2 权限矩阵

| 接口 | 需要权限 |
| --- | --- |
| `POST /send`、`POST /schedule` | `send` |
| `POST /upload` | `send` |
| `POST /tasks/create`、`/tasks/confirm`、`/tasks/cancel` | `send` |
| `POST /send/status` | `read` |
| `POST /robots`、`/conversations`、`/messages`、`/search` | `read` |
| `POST /tasks/list`、`/group-members` | `read` |

Token 缺少对应权限时返回 `403`，`error` 形如 `该 API Token 没有 send 权限`。

### 4.3 存储建议

- 放在环境变量 / 密钥管理里，不要写进代码、前端或 git；
- 生产与测试用不同 Token，测试空间 Token 无法操作真实群；
- 泄露后立刻撤销并重新签发；不要用 Token 代替用户登录态调用管理接口。

## 5. REST 接口

以下示例默认 Base URL 为 `https://chatbot.apetd.com`，使用 `/api/v1/open` 前缀。

### 5.1 机器人列表

```http
POST /api/v1/open/robots
Authorization: Bearer <token>
Content-Type: application/json

{}
```

```json
{
  "ok": true,
  "robots": [
    {
      "id": "f0c1…",
      "name": "课程顾问机器人",
      "mode": "live",
      "status": "bound",
      "groups": 3,
      "users": 12
    }
  ]
}
```

`groups` / `users` 是已同步的群聊与联系人数。`mode`：`live` 真实企微机器人，`demo` 演示机器人。

### 5.2 会话列表

```http
POST /api/v1/open/conversations
Authorization: Bearer <token>
Content-Type: application/json

{"robotId": "f0c1…"}
```

`robotId` 可省略，默认使用账户下第一个可用机器人。

```json
{
  "ok": true,
  "robot": {"id": "f0c1…", "name": "课程顾问机器人"},
  "groups": [
    {"conversation_id": "R:10949759080209864", "name": "2026 秋季招生群"}
  ],
  "users": [
    {"conversation_id": "S:wmXXXX", "name": "张老师"}
  ]
}
```

会话 ID 必须来自这里（或 `消息` / 回调里的 `conversationId`）。发送前平台会校验会话是否已同步，未同步返回 `400 请先同步并选择此机器人的会话`。

### 5.3 发送消息

```http
POST /api/v1/open/send
Authorization: Bearer <token>
Content-Type: application/json
Idempotency-Key: order-123-notify

{
  "conversationId": "R:10949759080209864",
  "type": "text",
  "content": "课程价格为 199 元。"
}
```

文本、图片、链接三种载荷：

```json
{
  "conversationId": "R:10949759080209864",
  "type": "text",
  "content": "课程价格为 199 元。",
  "atUserIds": ["wmXXXX"]
}
```

```json
{
  "conversationId": "R:10949759080209864",
  "type": "image",
  "imageUrl": "https://example.com/a.png"
}
```

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

返回（HTTP `202`，新任务）：

```json
{
  "ok": true,
  "messageId": "3f2a…",
  "status": "pending",
  "conversationId": "R:10949759080209864"
}
```

说明：

- `pending` 只表示**已入队**，不代表已送达。发送是异步队列，受上游企微限速与账户额度约束，最终状态用 `send/status` 或 `message.sent` / `message.failed` / `message.uncertain` 回调确认；
- 文本上限 2000 字，标题上限 100 字，链接描述上限 500 字；
- `atUserIds` 仅群会话（`R:` 开头）可用，最多 50 个；
- 图片必须是公网可访问 URL（或先调 `upload` 拿到 URL）；
- 单账户待发送队列上限 100 条，满时返回 `400 待发送任务已达 100 条，请先处理队列`。

### 5.4 查询发送状态

```http
POST /api/v1/open/send/status
Authorization: Bearer <token>
Content-Type: application/json

{"messageId": "3f2a…"}
```

```json
{
  "ok": true,
  "messageId": "3f2a…",
  "status": "accepted",
  "conversationId": "R:10949759080209864",
  "createdAt": "2026-10-26T12:00:00.000Z",
  "source": "API 发送"
}
```

| 状态 | 含义 | 建议 |
| --- | --- | --- |
| `pending` | 已入队，等待发送 | 稍后再查 |
| `accepted` | 上游已受理 | 成功，可记业务成功 |
| `failed` | 上游明确拒绝 | 可安全重发（换幂等键或确认原因） |
| `uncertain` | 超时，结果未知 | **不要自动重发**，人工核实后再决定 |
| `blocked` | 授权 / 额度等账户问题未发出 | 联系平台开通或充值 |
| `simulated` | 演示 / 模拟发送 | 仅演示环境出现 |

### 5.5 读取消息

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
| `robotId` | 可选，默认第一个可用机器人 |
| `conversationId` | 可选；不传则读取该机器人全部会话。测试空间**必须**指定测试群 |
| `after` | 可选，只返回该时间之后的消息，用于增量轮询 |
| `direction` | `in` 只看客户消息、`out` 只看机器人消息、`all` 全部；默认 `all` |
| `limit` | 默认 20，范围 1–100 |

```json
{
  "ok": true,
  "messages": [
    {
      "id": "9c8b…",
      "conversationId": "R:10949759080209864",
      "conversationName": "2026 秋季招生群",
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

轮询时把 `nextAfter` 作为下一次的 `after`。带 `after` 时从**最早未读**开始按时间正序返回，避免一次超过 `limit` 漏消息；不带 `after` 时返回最近 N 条。

> 平台消息记录用于最近检索，单账户只保留最近约 500 条。需要长期归档请用事件回调落库（见第 7 节），不要把 `/messages` 当历史数据库。

### 5.6 搜索消息

```http
POST /api/v1/open/search
Authorization: Bearer <token>
Content-Type: application/json

{"query": "课程", "conversationId": "R:10949759080209864", "limit": 20}
```

```json
{
  "ok": true,
  "query": "课程",
  "messages": [
    {
      "id": "9c8b…",
      "conversationId": "R:10949759080209864",
      "conversationName": "2026 秋季招生群",
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

- 关键词同时匹配**正文**与**发言人昵称**，大小写不敏感的子串匹配；
- 返回最近匹配的 N 条，按时间正序；`limit` 默认 20，上限 100；
- `query` 必填（最长 80 字），为空返回 `400 缺少搜索关键词`；
- 搜索范围同样受「最近约 500 条」限制，也受测试空间隔离。

### 5.7 图片上传

```http
POST /api/v1/open/upload
Authorization: Bearer <token>
Content-Type: image/png

<图片原始字节>
```

```json
{"ok": true, "url": "https://…/uploads/up-xxxx.png", "storage": "local", "bytes": 1234}
```

- 支持 PNG / JPG / GIF / WebP，单张 ≤ 5MB，需要 `send` 权限；
- 校验顺序：声明类型 → 大小 → 文件魔数（改扩展名会被 `415` 拒绝）；
- 可选 `?robotId=…`，用于图床路径模板；
- 拿到 `url` 后作为 `send` 的 `imageUrl`。

### 5.8 其他接口

| 接口 | 用途 | 权限 |
| --- | --- | --- |
| `POST /schedule` | 预约 5 秒后至 30 天内的一次发送 | `send` |
| `POST /tasks/list`、`/tasks/create`、`/tasks/confirm`、`/tasks/cancel` | 任务跟踪（到店 / 交稿 / 回访提醒） | 读 / 写按接口 |
| `POST /group-members` | 查询客户群成员花名册（来自企微同步） | `read` |

字段与返回示例见 [开发者 API](api.md) 对应章节。

## 6. 幂等发送

发送接口支持幂等键，避免超时重试、消息队列重投导致群里收到两条：

```http
Idempotency-Key: order-123-notify
```

也可以放在请求体：`{"idempotencyKey": "order-123-notify"}`。两者等价，请求头优先。

规则：

- 幂等范围是**单个工作空间**，键最长 120 字符；
- 有效期 **24 小时**，同一键 24 小时内重复提交不会重复发送；
- 重复请求返回 HTTP `200` 和第一次的 `messageId`，并带 `"duplicate": true`：

  ```json
  {
    "ok": true,
    "duplicate": true,
    "messageId": "3f2a…",
    "status": "accepted",
    "conversationId": "R:10949759080209864"
  }
  ```

- 重复返回里的 `status` 是这条消息**当前**状态，不一定是 `pending`；
- 建议用「业务单据 ID + 用途」做键，例如 `order-1234:payment-ok`；不同内容不要共用一个键；
- 同一键只对 `send` 生效，`schedule`（定时发送）不支持幂等键。

## 7. 业务事件回调

群消息到达、机器人消息最终状态、任务提醒 / 确认 / 到期时，猿媒BOT 会实时 `POST` 到你的 HTTPS 地址。

### 7.1 配置

页面：`个人中心 → API 接入 → 事件回调`。也可以调接口（工作空间所有者浏览器登录态）：

```http
POST /api/dev-webhook/config
Content-Type: application/json

{
  "enabled": true,
  "url": "https://your-server.com/yuanmei/webhook",
  "events": ["message.received", "message.sent", "message.failed", "message.uncertain", "task.reminded", "task.confirmed", "task.expired"],
  "filters": {"conversationIds": ["R:10949759080209864"], "keywords": ["报价", "退款"]},
  "qps": 5,
  "timeoutMs": 8000,
  "rotateSecret": false
}
```

| 字段 | 说明 |
| --- | --- |
| `enabled` | 是否启用；启用前必须已有合法 `url` |
| `url` | **必须 HTTPS**，公网可解析，不能指向 localhost / 内网 IP / 带账号密码的地址 |
| `events` | 订阅的事件列表，至少 1 个；不传则保留原配置；老账户未配置时默认全量推送 |
| `filters.conversationIds` | 只推送这些会话的事件，最多 50 个；为空表示不限 |
| `filters.keywords` | 只推送**消息事件**中包含这些关键词的事件，最多 50 个；为空表示不限 |
| `qps` | 每秒最多推送条数，1–50，默认 5；超出的进入延迟队列而不是丢弃 |
| `timeoutMs` | 单次投递超时，1000–15000 毫秒，默认 8000 |
| `rotateSecret` | 传 `true` 时重新生成签名密钥，旧密钥立即失效 |

返回里的 `secret` 只在**首次保存或轮换**时出现一次，务必立刻保存：

```json
{
  "ok": true,
  "secret": "whsec_0123…（48 位十六进制）",
  "config": {"enabled": true, "events": ["message.received"], "qps": 5, "timeoutMs": 8000}
}
```

其他回调管理接口：

| 接口 | 用途 |
| --- | --- |
| `GET /api/dev-webhook` | 查看配置与最近 20 条投递记录（时间、事件、HTTP 状态、耗时、错误、回复状态） |
| `POST /api/dev-webhook/test` | 推送一条 `webhook.test` 测试事件，验证地址与验签 |
| `POST /api/dev-webhook/replay` | 重放某条投递记录，body：`{"eventId":"…"}` 或 `{"at":"…"}` |

### 7.2 请求格式

```http
POST <你的回调地址>
Content-Type: application/json; charset=utf-8
User-Agent: yuanmei-bot-webhook/1.0
X-Yuanmei-Event: message.received
X-Yuanmei-Timestamp: 1793017003
X-Yuanmei-Nonce: 8f3c2a1b9d4e5f60
X-Yuanmei-Signature: sha256=…
```

| 头 | 说明 |
| --- | --- |
| `X-Yuanmei-Event` | 事件名，见 7.3 |
| `X-Yuanmei-Timestamp` | Unix 秒字符串，参与签名 |
| `X-Yuanmei-Nonce` | 随机串，可用于去重 |
| `X-Yuanmei-Signature` | `sha256=<hex>`，见 7.4 |

通用请求体字段：

| 字段 | 说明 |
| --- | --- |
| `event` | 事件名 |
| `eventId` | 事件唯一 ID，重试不变，用于去重 |
| `accountId` / `robotId` / `robotName` | 工作空间与机器人 |
| `conversationId` / `conversationName` / `isGroup` | 会话；群聊 `isGroup: true` |
| `messageId` / `senderId` / `senderName` / `msgtype` / `content` / `imageUrl` | 消息字段（消息类事件） |
| `status` / `error` / `source` | 出站消息状态、失败原因、消息来源（出站类事件） |
| `createdAt` | 事件发生时间（ISO 8601） |

`message.received` 的请求体还会带 `atList` 和 `reply`（提示用哪个接口回复 Token），其余事件不带 `reply` 字段。

### 7.3 七种事件

| 事件 | 触发时机 | 关键字段 |
| --- | --- | --- |
| `message.received` | 收到客户消息（已忽略的机器人回流不推） | `messageId`、`conversationId`、`senderId`、`senderName`、`content`、`atList` |
| `message.sent` | 机器人消息被上游受理（`accepted`） | `messageId`、`status: "accepted"`、`source` |
| `message.failed` | 上游明确拒绝发送（`failed`） | `messageId`、`status: "failed"`、`error` |
| `message.uncertain` | 发送超时，结果未知（`uncertain`） | `messageId`、`status: "uncertain"` |
| `task.reminded` | 任务提醒发出 | `taskId`、`title`、`remindCount`、`dueAt`、`person` |
| `task.confirmed` | 任务被确认词关闭 | `taskId`、`status: "confirmed"`、`confirmedBy` |
| `task.expired` | 任务到期结束 | `taskId`、`status: "expired"`、`remindCount` |

`eventId` 生成规则：

- `message.received`：等于消息 ID；
- 出站事件：`<消息ID>:<事件名>`，例如 `m123:message.sent`；
- `task.reminded`：`<任务ID>:task.reminded:<提醒次数>`，**每次提醒都是独立事件**；
- `task.confirmed` / `task.expired`：`<任务ID>:<事件名>`，每个任务只发一次。

入站消息示例：

```json
{
  "event": "message.received",
  "eventId": "9c8b…",
  "accountId": "a1…",
  "robotId": "f0c1…",
  "robotName": "课程顾问机器人",
  "conversationId": "R:10949759080209864",
  "conversationName": "2026 秋季招生群",
  "isGroup": true,
  "messageId": "9c8b…",
  "senderId": "wmXXXX",
  "senderName": "张老师",
  "msgtype": "text",
  "content": "课程怎么收费",
  "imageUrl": "",
  "atList": [],
  "createdAt": "2026-10-26T12:00:00.000Z",
  "reply": {"api": "/api/open/send", "token": "使用 API 接入 Token"}
}
```

出站消息示例：

```json
{
  "event": "message.sent",
  "eventId": "3f2a…:message.sent",
  "accountId": "a1…",
  "robotId": "f0c1…",
  "robotName": "课程顾问机器人",
  "conversationId": "R:10949759080209864",
  "conversationName": "2026 秋季招生群",
  "isGroup": true,
  "messageId": "3f2a…",
  "senderId": "robot",
  "senderName": "课程顾问机器人",
  "msgtype": "text",
  "content": "课程价格 199 元。",
  "status": "accepted",
  "error": "",
  "source": "API 发送",
  "createdAt": "2026-10-26T12:00:00.000Z"
}
```

任务事件示例：

```json
{
  "event": "task.reminded",
  "eventId": "t1…:task.reminded:1",
  "taskId": "t1…",
  "title": "回访张老师",
  "conversationId": "R:10949759080209864",
  "conversationName": "2026 秋季招生群",
  "isGroup": true,
  "person": {"name": "张老师", "userId": "wmXXXX"},
  "confirmWords": ["已到店"],
  "dueAt": "2026-10-26T13:00:00.000Z",
  "remindCount": 1,
  "status": "pending",
  "note": "",
  "createdAt": "2026-10-26T12:00:00.000Z"
}
```

补充：

- 排队中（`pending`）、模拟发送（`simulated`）、被拦截（`blocked`）不推 `message.*` 出站事件；
- 测试空间账户不推送事件，避免体验数据外发；
- `webhook.test` 是「测试」按钮发送的调试事件，不属于七种业务事件。

### 7.4 校验签名（HMAC-SHA256）

算法：

```text
signature = "sha256=" + HMAC_SHA256(secret, `${X-Yuanmei-Timestamp}.${原始请求体}`) 的十六进制
```

三个要点：

1. **用原始请求体**（byte 级）参与计算，不要先 JSON 反序列化再重新序列化；
2. 用**常量时间**比较（Node `timingSafeEqual` / Python `hmac.compare_digest`）；
3. 校验时间戳在 5 分钟内（防重放），并按 `eventId` / `X-Yuanmei-Nonce` 去重。

Node 完整示例（原生 `http`，不依赖框架）：

```js
import {createServer} from 'node:http';
import {createHmac, timingSafeEqual} from 'node:crypto';

const SECRET = process.env.YUANMEI_WEBHOOK_SECRET;

function verify(secret, timestamp, rawBody, signature) {
  const expected = 'sha256=' + createHmac('sha256', secret).update(`${timestamp}.${rawBody}`).digest('hex');
  const a = Buffer.from(expected);
  const b = Buffer.from(String(signature || ''));
  return a.length === b.length && timingSafeEqual(a, b);
}

createServer(async (req, res) => {
  if (req.method !== 'POST' || req.url !== '/yuanmei/webhook') {
    res.writeHead(404).end();
    return;
  }
  const chunks = [];
  for await (const chunk of req) chunks.push(chunk);
  const rawBody = Buffer.concat(chunks).toString('utf8'); // 原始请求体

  const timestamp = req.headers['x-yuanmei-timestamp'];
  const signature = req.headers['x-yuanmei-signature'];
  if (!verify(SECRET, timestamp, rawBody, signature)) {
    res.writeHead(401, {'Content-Type': 'application/json'}).end('{"error":"bad signature"}');
    return;
  }
  if (Math.abs(Date.now() / 1000 - Number(timestamp)) > 300) {
    res.writeHead(401).end('{"error":"timestamp expired"}');
    return;
  }

  const payload = JSON.parse(rawBody);
  // 1) 按 eventId 幂等去重（重试时 eventId 不变）
  // 2) 业务处理：入库 / 调内部服务 / 生成回复
  const reply = payload.event === 'message.received' ? {reply: {type: 'text', content: '收到，马上为您查询。'}} : {};

  // 3) 必须在超时时间内返回 2xx；返回体带 reply 会自动发回群
  res.writeHead(200, {'Content-Type': 'application/json; charset=utf-8'}).end(JSON.stringify(reply));
}).listen(8080);
```

Python 完整示例（Flask）：

```python
import hashlib
import hmac
import os
import time

from flask import Flask, request, jsonify, abort

app = Flask(__name__)
SECRET = os.environ["YUANMEI_WEBHOOK_SECRET"].encode()


def verify(secret: bytes, timestamp: str, raw_body: bytes, signature: str) -> bool:
    expected = "sha256=" + hmac.new(secret, f"{timestamp}.".encode() + raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature or "")


@app.post("/yuanmei/webhook")
def webhook():
    raw_body = request.get_data()  # 原始请求体，务必不要用 request.get_json()
    timestamp = request.headers.get("X-Yuanmei-Timestamp", "")
    signature = request.headers.get("X-Yuanmei-Signature", "")
    if not verify(SECRET, timestamp, raw_body, signature):
        abort(401)
    if abs(time.time() - int(timestamp or 0)) > 300:
        abort(401)

    payload = request.get_json(force=True)
    # 按 payload["eventId"] 幂等去重 + 业务处理
    if payload["event"] == "message.received":
        return jsonify({"reply": {"type": "text", "content": "收到，马上为您查询。"}})
    return jsonify({"ok": True})


if __name__ == "__main__":
    app.run(port=8080)
```

SDK 里也内置了验签函数（见第 9 节）：

```js
import {verifyWebhookSignature} from 'integrations/sdk/node/yuanmei-bot-sdk.mjs';
const ok = verifyWebhookSignature(secret, timestamp, rawBody, signatureHeader);
```

```python
from yuanmei_bot import verify_webhook_signature
ok = verify_webhook_signature(secret, timestamp, raw_body, signature_header)
```

验签失败排查顺序：密钥是否是最新（轮换后旧密钥立即失效）→ 是否用了原始请求体 → 是否把 `timestamp.` 拼进签名 → 是否框架改写了请求体 / 编码。

### 7.5 直接回复（reply 自动回群）

回调返回 **HTTP 2xx** 且响应体是 JSON，带以下任一字段时，猿媒BOT 会把内容发回原会话：

```json
{"reply": {"type": "text", "content": "课程价格 199 元。"}}
```

```json
{"reply": {"type": "image", "url": "https://example.com/a.png"}}
```

```json
{"reply": {"type": "link", "title": "课程介绍", "url": "https://example.com/course", "desc": "12 节课"}}
```

简写：`{"replyText": "课程价格 199 元。"}` 等价于文本回复。

规则与限制：

- 只有 `message.received` 场景建议返回 `reply`；任务事件不会回群；**出站事件（`message.*` 的 sent/failed/uncertain）不要返回 `reply`**，否则机器人会回复自己的消息，形成回环；
- 文本超过 2000 字会被截断，标题 100 字、链接描述 500 字；空文本会被忽略；
- 回复走和手动发送同一条队列，同样受授权、额度、上游限速约束；是否成功会记在投递日志的 `replyState`（`sent` / `failed:原因`）；
- 回复失败不会让回调算失败，接口不会重发你的 reply；
- 需要更长处理时间（例如调 AI 几十秒）时，不要阻塞回调：先 2xx 入队，稍后调 `POST /api/v1/open/send` 主动回复。

### 7.6 超时、重试与重放

- 单次投递超时：默认 **8 秒**，可在配置里设为 1000–15000 毫秒；
- 成功条件：HTTP **2xx**；
- **超时、连接失败、5xx** 会进入**持久化重试队列**，最多尝试 **3 次**：
  - 第 1 次失败后约 **30 秒**重试；
  - 第 2 次失败后约 **120 秒**重试；
  - 第 3 次仍失败则放弃；
- 重试队列落库，**服务重启后继续投递**，不需要业务方补拉；
- 重试时 `eventId` 不变，按它去重即可；
- **4xx / 3xx 不重试**（视为业务方明确拒绝）；
- 关闭回调、取消事件订阅、停用账户后，历史重试队列会被丢弃；
- 队列上限 500 条，超出时丢最旧的一条；投递日志保留最近 100 条，页面展示最近 20 条；
- 「重放」用于排查或恢复失败回调：`POST /api/dev-webhook/replay`，body 传 `{"eventId":"…"}` 或 `{"at":"…"}`，要求回调仍启用且该事件仍在订阅范围内。重放只投递一次，不进入重试队列。

### 7.7 会话 / 关键词过滤

在回调配置的 `filters` 里配置，命中过滤规则的事件**不推送**（也不会进入重试队列）：

```json
{
  "filters": {
    "conversationIds": ["R:10949759080209864", "R:20949759080209864"],
    "keywords": ["报价", "退款", "投诉"]
  }
}
```

- `conversationIds`：最多 50 个；事件带 `conversationId` 且不在列表内则跳过；为空表示所有会话；
- `keywords`：最多 50 个，**只对 `message.*` 事件生效**；用 `content` + `senderName` 做大小写不敏感的子串匹配，都不包含则跳过；
- 任务事件受 `conversationIds` 过滤，不受 `keywords` 过滤；
- 过滤在推送前完成，日志里不会出现被过滤的事件。

### 7.8 QPS 与并发

- 每个工作空间默认 **5 QPS**（每秒最多 5 次投递），可设 1–50；
- 超出的投递**不会丢弃**，而是延迟约 1 秒后重新排队；
- QPS 只限制新事件推送；失败重试的投递不占用 QPS，但同样受单次超时约束；
- 未配置 `qps` 的老账户不限制（兼容行为），新配置默认 5；
- 你的回调处理要**快速返回**：慢接口会占满 QPS 并触发超时重试；重活放到队列异步处理。

### 7.9 安全建议

- 密钥 `whsec_` 开头，AES 加密存储、接口不回显、日志不记录完整密钥；
- 只接受 HTTPS，地址会校验公网、拒绝内网 / localhost；
- 校验签名 + 时间戳 + 去重三件套，缺一不可；
- 密钥泄露立即轮换（`rotateSecret: true`），旧密钥立即失效；
- 回调地址建议加自己的鉴权参数或 IP 白名单作为第二道防线。

## 8. 常见错误与排查

### 8.1 错误码

| HTTP | 典型 `error` | 原因与处理 |
| --- | --- | --- |
| `400` | `会话 ID不能为空，且不能超过 200 字` | 少传 / 传空 `conversationId`；先用 `conversations` 拿 ID |
| `400` | `消息内容不能为空，且不能超过 2000 字` | 文本为空或超长 |
| `400` | `msgtype 仅支持 text / image / link` | `type` 拼错，只支持这三种 |
| `400` | `请先同步并选择此机器人的会话` | 会话未同步 / `conversationId` 不属于该机器人 |
| `400` | `请输入完整的 HTTP / HTTPS 地址` / `地址格式不支持` | 图片 / 链接 URL 不合法 |
| `400` | `缺少搜索关键词` | `search` 的 `query` 为空 |
| `400` | `请输入 messageId` | `send/status` 少传 `messageId` |
| `400` | `请指定测试群会话` | 测试空间 Token 读消息未指定会话 |
| `400` | `dueAt 需为 5 秒后至 30 天内的 ISO 时间` | `schedule` 定时时间越界 |
| `400` | `到期时间需晚于当前 5 秒以上` / `到期时间需在未来 30 天内` | `tasks/create` 到期时间越界 |
| `400` | `关联群必须是该机器人已同步的群聊` | 任务只能关联已同步的群会话 |
| `400` | `进行中的跟踪任务最多 300 条` | 先确认或取消历史任务 |
| `400` | `任务不在跟踪中` | 任务已确认 / 取消 / 到期，不能重复操作 |
| `400` | `待发送任务已达 100 条，请先处理队列` | 发送队列满，降速或等待 |
| `401` | `缺少 Authorization: Bearer <token>` | 请求头没带 Token |
| `401` | `API Token 无效或账户已停用` | Token 被撤销 / 替换 / 账户停用 |
| `403` | `账户未开通或已停用，请联系平台开通授权` | 授权到期或未开通，联系平台 |
| `403` | `该 API Token 没有 send 权限` / `… read 权限` | Token 权限不含该接口所需权限，重新签发 |
| `403` | `请求来源不允许` | 浏览器跨域带了 `Origin`，改为服务端调用 |
| `403` | `测试空间只能发送到测试群` / `测试空间只能读取测试群` | 测试空间 Token 访问了真实会话 |
| `404` | `没有可用的机器人` | 机器人未绑定 / 全部解绑 |
| `404` | `消息不存在` | `messageId` 不属于当前账户，或消息已被清理 |
| `404` | `任务不存在` | `taskId` 不属于当前账户，或已被清理 |
| `404` | `接口不存在` | 路径拼错，注意 `/api/v1/open/...` |
| `413` | `请求内容过大` / `图片不能超过 5MB` | 上传超过 5MB |
| `415` | `需要 application/json` | POST 请求头缺少 `Content-Type: application/json` |
| `415` | `仅支持 PNG / JPG / GIF / WebP 图片` / `文件内容与声明的图片格式不符` | MIME 不支持或文件魔数不符 |
| `429` | `操作过于频繁，请一分钟后再试` | 触发 IP 限流，按响应等待后重试 |
| `500` | `服务暂时不可用` | 平台内部错误，记录时间与请求后联系平台 |

### 8.2 常见场景排查

**401 / 403**

1. `Authorization` 是否为 `Bearer <token>`（`Bearer` 后一个空格）；
2. Token 是否被重新签发 / 撤销，前缀是否对得上；
3. 账户授权是否到期；
4. 调用的接口所需权限是否在 Token 的 `scopes` 里。

**429**

- 每个 IP 每分钟：发送类 120 次、读取类 120 次、上传 60 次；多台机器共用一个出口 IP 时会叠加；
- 降低轮询频率（用回调和 `after` 增量读取代替高频全量轮询），指数退避重试。

**发送一直是 `pending` 或没有送达**

1. 检查账户授权、发送服务是否开通、额度是否用完（`blocked`）；
2. 检查机器人是否被上游限速 / 断连；
3. 用 `send/status` 看最终状态；`uncertain` 不要自动重发，避免重复；
4. 回调日志里是否有 `message.sent` / `message.failed` / `message.uncertain`。

**回调一条都收不到**

1. 回调是否启用、`url` 是否为 HTTPS 公网地址；
2. 事件是否勾选（订阅列表至少 1 个），老账户未配置 = 全量；
3. `filters.conversationIds` / `filters.keywords` 是否把你的事件过滤掉了；
4. 是否配置了 `qps` 且被打满（看投递记录里是否有延迟）；
5. 服务器防火墙 / 安全组 / WAF 是否放行平台出口；
6. 用「测试」按钮发 `webhook.test`，先验证网络与验签，再验证业务事件；
7. 测试空间账户不推送事件，确认用的是正式工作空间。

**回调收到了但验签失败**

1. 密钥是不是最新（轮换后旧密钥立刻失效）；
2. 是否用**原始请求体**参与签名；
3. 时间戳是否拼成了 `` `${timestamp}.${body}` ``；
4. 框架是否改写了请求体（例如统一转 JSON、压缩、去掉空白）；
5. 比较是否用了常量时间函数，`signature` 是否以 `sha256=` 开头。

**返回了 reply 但群里没消息**

1. HTTP 状态是否 2xx；
2. 响应体是否是合法 JSON，`reply.type` 是否为 `text` / `image` / `link`；
3. 是否在出站事件上返回了 reply（会被用来回机器人自己的消息，不建议）；
4. 投递记录里的 `replyState` 是否为 `sent`；`failed:` 开头表示入队失败；
5. 回复同样受授权、额度、上游限速限制。

**同一事件收到多次**

- 这是重试或重放导致的正常现象，按 `eventId` 做业务幂等即可。

## 9. Node / Python SDK

仓库内提供零依赖 SDK：

- Node：`integrations/sdk/node/yuanmei-bot-sdk.mjs`（Node 18+，原生 `fetch`）
- Python：`integrations/sdk/python/yuanmei_bot.py`（Python 3.9+，仅标准库）

```js
import {YuanmeiBot} from './yuanmei-bot-sdk.mjs';

const bot = new YuanmeiBot({token: process.env.YUANMEI_API_TOKEN});
const r = await bot.sendText('R:10949759080209864', '课程价格 199 元。', {idempotencyKey: 'order-123-notify'});
const status = await bot.status(r.messageId);
const msgs = await bot.messages({conversationId: 'R:10949759080209864', limit: 20});
```

```python
from yuanmei_bot import YuanmeiBot

bot = YuanmeiBot(token="YOUR_TOKEN")
r = bot.send_text("R:10949759080209864", "课程价格 199 元。", idempotency_key="order-123-notify")
print(r["messageId"])
print(bot.status(r["messageId"]))
```

完整方法、参数和错误处理见 SDK README：`integrations/sdk/README.md`。

## 10. 上线检查清单

接入完成、上线前逐项确认：

- [ ] 使用 `/api/v1/open/*`，Token 存在服务端环境变量里；
- [ ] 发送带 `Idempotency-Key`，业务能按 `messageId` 跟踪状态；
- [ ] 超时 / 5xx 会重试，`uncertain` 不会自动重发；
- [ ] 事件回调用 HTTPS、回调地址公网可达；
- [ ] 验签使用原始请求体 + 常量时间比较 + 5 分钟时间窗；
- [ ] 按 `eventId` 去重，重试 / 重放不会重复处理；
- [ ] 回调处理在 8 秒内返回，重活异步；
- [ ] 记录了投递日志并有人负责查看失败重试；
- [ ] 测试空间与正式空间使用不同 Token / 回调地址；
- [ ] 密钥泄露 / Token 泄露的轮换流程已确认。

## 11. 相关文档

- [开发者 API](api.md) — 全部 REST 接口字段与示例
- [事件回调](callback.md) — 回调配置与事件说明
- [MCP 接入](mcp.md) — AI 客户端接入
- [命令行 CLI](cli.md) — 脚本 / cron 场景
- [常见问题](faq.md) — 账号与使用问题
- SDK README：`integrations/sdk/README.md` — Node / Python SDK
- OpenAPI 3.1 规范：`docs/public/openapi.yaml`（可直接导入 Postman / Apifox 等工具）
