# 爱马仕 Hermes 接入

## 1. 签发令牌

```text
个人中心 → MCP 接入 → 签发 MCP 令牌
```

## 2. 添加远程 MCP

```bash
hermes mcp add yuanmei --url https://chatbot.apetd.com/mcp
```

然后在 Hermes 的 MCP 配置中补充请求头：

```text
Authorization: Bearer <TOKEN>
```

## 3. 验证

```bash
hermes mcp test yuanmei
```

再调用 `whoami` 确认工作空间。

## 4. 注意

MCP 是 profile 级的；如果当前 profile 看不到工具，确认是在目标 profile 中添加的。


## 可选：MCP Catalog / Skill

Hermes 可以直接用 MCP：

```bash
hermes mcp add yuanmei --url https://chatbot.apetd.com/mcp
```

如果希望进入 Hermes MCP Catalog，提交材料在：

```text
integrations/hermes-catalog
```

Catalog 需要 Hermes 官方审核，提交不代表一定收录。

也可以使用 AgentSkills：

```text
integrations/agent-skills/yuanmei-mcp
```

放入 Hermes 的 skills 目录。

仓库：

https://github.com/CN-Jayden/yuanmei-bot-agent-skills/tree/main/hermes
