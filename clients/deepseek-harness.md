# DeepSeek Harness 接入

猿媒BOT 提供 DSH 插件，安装后自动注册 MCP。

## 1. 签发令牌

```text
个人中心 → API 接入 → MCP 接入 → 签发 MCP 令牌
```

## 2. 安装插件

桌面版直接：

```text
插件 → 添加插件 → 填 https://github.com/CN-Jayden/yuanmei-bot-dsh-mcp
```

命令行安装时先确认当前 profile（桌面版通常是 `desktop`，Web 版是 `web`）：

```bash
dsh plugin --profile <profile> add https://github.com/CN-Jayden/yuanmei-bot-dsh-mcp
```

## 3. 设置令牌

macOS / Linux：

```bash
export YUANMEI_MCP_TOKEN='你的令牌'
```

Windows PowerShell：

```powershell
$env:YUANMEI_MCP_TOKEN = '你的令牌'
```

## 4. 重启

```bash
dsh web
```

## 5. 验证

在 DSH 中调用：

```text
mcp__yuanmei__whoami
```

应该返回工作空间和工具范围。

插件仓库：

https://github.com/CN-Jayden/yuanmei-bot-dsh-mcp
