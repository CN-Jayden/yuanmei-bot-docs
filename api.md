# 服务端发送 API

客户后台生成回复后，用这个接口发回群。

## 1. 签发 Token

```text
个人中心 → API 接入 → 服务端发送 API → 签发 Token
```

Token 长期有效，可随时撤销。

## 2. 发送地址

```http
POST https://chatbot.apetd.com/api/open/send
Authorization: Bearer <token>
Content-Type: application/json
```

## 3. 发送文本

```json
{
  "conversationId": "R:10949759080209864",
  "type": "text",
  "content": "课程价格为 199 元。"
}
```

返回：

```json
{
  "ok": true,
  "messageId": "...",
  "status": "pending",
  "conversationId": "R:10949759080209864"
}
```

`pending` 表示已入队，不代表已送达；最终结果看「收发记录」。

## 4. 发送图片

```json
{
  "conversationId": "R:10949759080209864",
  "type": "image",
  "imageUrl": "https://example.com/a.png"
}
```

图片必须公网可访问。

## 5. 发送链接卡片

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

## 6. 限制

- 只有工作空间所有者能签发/撤销 Token
- 发送受授权、额度、10 秒限速、队列限制
- 测试空间只能发送到测试群

## 7. 配合 MCP 读消息

推荐：

```text
MCP read_messages
  → 客户 AI / 知识库
  → REST /api/open/send
```

参考实现：

https://github.com/CN-Jayden/yuanmei-bot-workspace/tree/main/integrations/agent-bridge
