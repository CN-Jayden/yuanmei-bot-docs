# 事件回调

群消息到达、机器人消息发出、任务提醒 / 确认 / 到期时，猿媒BOT 会把事件实时推送到你自己的系统。业务系统可以直接返回回复自动回群，也可以稍后调用发送 API 主动回复。

适合：工单/CRM 同步、客服提醒、关键词告警、自己的 AI 服务。

## 1. 配置

```text
个人中心 → API 接入 → 事件回调
```

- 回调地址必须使用 **HTTPS**
- 地址不能指向本机或内网 IP
- 保存时生成密钥，**密钥只完整显示一次**
- 可随时轮换密钥（旧密钥立即失效）
- 可勾选**订阅哪些事件**，没勾选的不会推送；至少保留一个
- 老账户未配置订阅时默认全量推送
- 「测试」按钮会推送一条 `webhook.test` 事件，用于验证地址和签名

订阅也可以通过接口配置：

```json
POST /api/dev-webhook/config
{
  "enabled": true,
  "url": "https://your-server.com/yuanmei/webhook",
  "events": ["message.received", "task.confirmed"]
}
```

取消某个事件的订阅后，该事件的历史重试也会被丢弃。

## 2. 请求

```http
POST <你的回调地址>
Content-Type: application/json; charset=utf-8
X-Yuanmei-Event: message.received
X-Yuanmei-Timestamp: 1793017003
X-Yuanmei-Nonce: 8f3c2a1b9d4e5f60
X-Yuanmei-Signature: sha256=...
```

请求体：

```json
{
  "event": "message.received",
  "eventId": "message-id",
  "accountId": "account-id",
  "robotId": "robot-id",
  "robotName": "课程顾问机器人",
  "conversationId": "R:10949759080209864",
  "conversationName": "2026 秋季招生群",
  "isGroup": true,
  "messageId": "message-id",
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

## 2.1 事件类型

| 事件 | 触发时机 | 主要字段 |
| --- | --- | --- |
| `message.received` | 收到客户消息 | `messageId`、`conversationId`、`senderId`、`senderName`、`content`、`reply` |
| `message.sent` | 机器人消息被上游受理 | `messageId`、`conversationId`、`content`、`status: accepted` |
| `message.failed` | 上游明确拒绝发送 | 同上，`status: failed`、`error` |
| `message.uncertain` | 发送超时，结果未知 | 同上，`status: uncertain` |
| `task.reminded` | 任务提醒发出 | `taskId`、`title`、`conversationId`、`remindCount`、`dueAt`、`person` |
| `task.confirmed` | 任务被确认 | `taskId`、`status: confirmed`、`confirmedBy` |
| `task.expired` | 任务到期结束 | `taskId`、`status: expired`、`remindCount` |

补充说明：

- 入站消息（`message.received`）不会被机器人自己的回流触发；已忽略的回流不推送
- 排队中（`pending`）、模拟发送（`simulated`）、被拦截（`blocked`）不推送出站事件
- 每次任务提醒都是独立事件，`eventId` 形如 `任务ID:task.reminded:提醒次数`；确认 / 到期每任务只发一次
- 出站事件 `eventId` 形如 `消息ID:message.sent`
- 只在勾选的事件范围内推送；未配置订阅的老账户 = 全量推送

出站事件示例：

```json
{
  "event": "message.sent",
  "eventId": "message-id:message.sent",
  "accountId": "account-id",
  "robotId": "robot-id",
  "robotName": "课程顾问机器人",
  "conversationId": "R:10949759080209864",
  "conversationName": "2026 秋季招生群",
  "isGroup": true,
  "messageId": "message-id",
  "senderId": "robot",
  "senderName": "课程顾问机器人",
  "msgtype": "text",
  "content": "课程价格 199 元。",
  "status": "accepted",
  "error": "",
  "createdAt": "2026-10-26T12:00:00.000Z"
}
```

任务事件示例：

```json
{
  "event": "task.reminded",
  "eventId": "task-id:task.reminded:1",
  "accountId": "account-id",
  "robotId": "robot-id",
  "taskId": "task-id",
  "title": "回访张老师",
  "conversationId": "R:10949759080209864",
  "conversationName": "2026 秋季招生群",
  "person": {"name": "张老师", "userId": "wmXXXX"},
  "confirmWords": ["已到店"],
  "dueAt": "2026-10-26T13:00:00.000Z",
  "remindCount": 1,
  "status": "pending",
  "note": ""
}
```

## 3. 验证签名

签名算法：**HMAC-SHA256**，对 `` `${timestamp}.${原始请求体}` `` 计算，结果带 `sha256=` 前缀，放在 `X-Yuanmei-Signature` 头。

Node：

```js
import {createHmac} from 'node:crypto';

const expected = 'sha256=' + createHmac('sha256', secret).update(`${timestamp}.${rawBody}`).digest('hex');
if (expected !== signature) throw new Error('签名不匹配');
```

Python：

```python
import hmac, hashlib

expected = 'sha256=' + hmac.new(secret.encode(), f'{timestamp}.{raw}'.encode(), hashlib.sha256).hexdigest()
```

建议：

- 用**原始请求体字符串**验签，不要先反序列化再拼 JSON
- 校验 `X-Yuanmei-Timestamp` 在 5 分钟内，防止旧请求重放；`X-Yuanmei-Nonce` 可做去重
- 用常量时间比较（Node `timingSafeEqual` / Python `hmac.compare_digest`）

## 4. 幂等

`eventId` 就是消息 ID，同一个 `eventId` 在重试时会重复投递。请按 `eventId` 去重，保证业务侧只处理一次。

## 5. 直接回复

**只有 `message.received` 支持直接回复**（任务 / 出站事件忽略 `reply`，避免把回复再发一遍）。

回调返回 2xx 且响应体带 `reply` 时，猿媒BOT 会把回复发回原群：

```json
{"reply": {"type": "text", "content": "课程价格 199 元。"}}
```

也支持图片和链接卡片：

```json
{"reply": {"type": "image", "url": "https://example.com/a.png"}}
{"reply": {"type": "link", "title": "课程介绍", "url": "https://example.com/course", "desc": "12 节课"}}
```

简写：`{"replyText": "课程价格 199 元。"}` 等价于文本回复。

回复走和手动发送同一条队列，受授权、额度、10 秒限速约束；`replyState` 会记录在投递日志里（`sent` / `failed:原因`）。

## 6. 重试与投递记录

- 请求超时 8 秒
- 超时或 5xx：进入**持久化重试队列**，最多尝试 3 次（失败约 30 秒后第 2 次、再约 2 分钟后第 3 次）
- 队列落库，**服务重启后继续投递**，不需要业务方补拉
- 重试时 `eventId` 不变，按它去重即可
- 4xx：认为业务方明确拒绝，不重试、不入队
- 关闭回调、停用账户后，历史队列会被丢弃
- 每次尝试记一条投递记录；「事件回调」页展示最近 20 条（时间、事件、HTTP 状态、耗时、错误、回复状态）

## 7. 安全

- 密钥 AES 加密存储，接口不回显，只在生成/轮换时返回一次
- 回调地址做 HTTPS 与内网地址校验
- 日志不记录完整密钥
- 如果密钥泄露，立即轮换；旧密钥立即失效

## 8. 和发送 API 的关系

| 做法 | 适用 |
| --- | --- |
| 回调直接返回 `reply` | 快速同步回复，接入成本最低 |
| 回调里只做入库，稍后调 `/api/open/send` | 异步处理、AI 生成耗时较长、需要人工审核 |

发送 API 见 [开发者 API](api.md)。
