# 龙虾 OpenClaw 接入

## 1. 签发令牌

```text
个人中心 → API 接入 → MCP 接入 → 签发 MCP 令牌
```

## 2. 配置

在 OpenClaw 的 MCP 配置入口添加远程 HTTP 服务：

```json
{
  "mcpServers": {
    "yuanmei-bot": {
      "type": "http",
      "url": "https://chatbot.apetd.com/mcp",
      "headers": {
        "Authorization": "Bearer <TOKEN>"
      }
    }
  }
}
```

不同 OpenClaw 版本的字段名可能不同，以当前客户端 MCP 配置页为准。

## 3. 验证

接入后先调用 `whoami`，不要先发消息。
