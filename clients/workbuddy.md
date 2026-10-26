# WorkBuddy 接入

## 1. 签发令牌

```text
个人中心 → API 接入 → MCP 接入 → 签发 MCP 令牌
```

## 2. 配置

WorkBuddy 的用户级配置：

```text
~/.workbuddy/mcp.json
```

也可以从：

```text
侧边栏 插件 → MCP 服务器 → 配置 MCP
```

粘贴：

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

## 3. 验证

保存后查看 MCP 状态是否为绿色，然后在 WorkBuddy 里让它调用 `whoami`。
