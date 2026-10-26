# DeepSeek Harness 接入

猿媒BOT 提供 DSH 插件，安装后自动注册 MCP。

## 1. 签发令牌

```text
个人中心 → API 接入 → MCP 接入 → 签发 MCP 令牌
```

## 2. 安装插件

```bash
git clone https://github.com/CN-Jayden/yuanmei-bot-dsh-mcp.git
cd yuanmei-bot-dsh-mcp
dsh plugin --profile web add link:$(pwd)
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
