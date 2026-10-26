# DeepSeek Harness 接入

猿媒BOT 提供 DSH 插件，安装后自动注册 MCP。

## 1. 签发令牌

```text
个人中心 → API 接入 → MCP 接入 → 签发 MCP 令牌
```

## 方案 A：猿媒BOT DSH 插件（环境变量）

### 安装插件

桌面版直接：

```text
插件 → 添加插件 → 填 https://github.com/CN-Jayden/yuanmei-bot-dsh-mcp
```

命令行安装时先确认当前 profile（桌面版通常是 `desktop`，Web 版是 `web`）：

```bash
dsh plugin --profile <profile> add https://github.com/CN-Jayden/yuanmei-bot-dsh-mcp
```

### 设置令牌

macOS / Linux：

```bash
export YUANMEI_MCP_TOKEN='你的令牌'
```

Windows PowerShell：

```powershell
$env:YUANMEI_MCP_TOKEN = '你的令牌'
```

### 重启

```bash
dsh web
```

### 验证

在 DSH 中调用：

```text
mcp__yuanmei__whoami
```

应该返回工作空间和工具范围。

插件仓库：

https://github.com/CN-Jayden/yuanmei-bot-dsh-mcp


## 方案 B：用 MCP 管理插件，在界面配置

如果你不想设置环境变量，可以改用 `dsh-mcp` 管理插件。

1. 先卸载 `yuanmei-bot-dsh-mcp`，避免两个插件都注册 `yuanmei`
2. 在 DSH「插件 → 添加插件」安装 `dsh-mcp`
3. 打开「设置 → MCP → 添加服务器」
4. 填写：

```text
名称：yuanmei
传输：streamable-http
URL：https://chatbot.apetd.com/mcp
请求头：Authorization: Bearer <TOKEN>
```

5. 点「连接测试」
6. 在会话中调用 `mcp__yuanmei__whoami`

注意：`dsh-mcp` 是第三方插件，安装前确认作者和源码：

https://github.com/ArvinQi/dsh-mcp

两个方案二选一，不要同时启用。
