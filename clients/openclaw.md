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


## 可选：安装 Agent Skill

如果 OpenClaw 使用 AgentSkills，可以把：

```text
integrations/agent-skills/yuanmei-mcp
```

复制到：

```text
<workspace>/skills/yuanmei-mcp
```

或：

```text
~/.openclaw/skills/yuanmei-mcp
```

Skill 只负责教 Agent 使用 `mcp__yuanmei__*` 工具，MCP 地址和令牌仍需先配置。

仓库：

https://github.com/CN-Jayden/yuanmei-bot-agent-skills/tree/main/skills/yuanmei-mcp
