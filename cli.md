# 命令行 CLI

适合：服务器脚本、cron 定时任务、运维排查，以及**不支持 MCP 的 Agent**（只要有 shell 就能用）。

CLI 基于[开发者 API](api.md)，零依赖，Node ≥ 20。

## 1. 安装

在仓库里直接运行：

```bash
node integrations/cli/yuanmei.mjs --help
```

或本地全局安装：

```bash
cd integrations/cli
npm link
yuanmei --help
```

## 2. 配置

```bash
export YUANMEI_TOKEN='个人中心 → API 接入 签发的 Token'
export YUANMEI_BASE='https://chatbot.apetd.com'   # 可省略
```

也可以用 `--token` / `--base` 覆盖。

## 3. 常用命令

```bash
# 工作空间与机器人
yuanmei whoami
yuanmei robots
yuanmei conversations --robot <robotId>

# 读消息（支持 after 增量）
yuanmei read --conversation 'R:10949759080209864' --after '2026-10-26T00:00:00Z' --limit 20 --direction in

# 搜索
yuanmei search '课程' --conversation 'R:10949759080209864'

# 发文本（支持群 @ 与幂等键）
yuanmei send --conversation 'R:10949759080209864' --text '课程价格 199 元。' --at 'wmXXXX,wmYYYY' --idempotency 'order-123'

# 发图片：先上传本地图片，再发返回的 URL
yuanmei upload ./poster.png --robot <robotId>
yuanmei send --conversation 'R:10949759080209864' --image 'https://.../uploads/mcp-xxx.png'

# 定时发送（未来 30 天内一次）
yuanmei schedule --conversation 'R:10949759080209864' --due '2026-10-27T01:00:00Z' --text '明天 9 点提醒家长交作业'

# 客户群成员花名册
yuanmei members --conversation 'R:10949759080209864'

# 查发送状态
yuanmei status <messageId>

# 任务跟踪：列表 / 登记 / 确认 / 取消
yuanmei tasks --status pending
yuanmei task:create --conversation 'R:10949759080209864' --title '回访张老师' \
  --due '2026-10-26T13:00:00Z' --person-name '张老师' --confirm-words '已到店'
yuanmei task:confirm <taskId>
yuanmei task:cancel <taskId>

# 持续读取新消息（一行一条 JSON，适合管道）
yuanmei watch --conversation 'R:10949759080209864' --interval 5 | jq -c '{senderName,content}'
```

## 4. 参数速查

| 命令 | 必填 | 可选 |
| --- | --- | --- |
| `read` | `--conversation` | `--after`、`--direction`、`--limit`、`--robot` |
| `search` | 关键词位置参数 | `--conversation`、`--limit`、`--robot` |
| `send` | `--conversation` + `--text` / `--image` / `--link --title` | `--at`、`--desc`、`--idempotency`、`--robot` |
| `upload` | 文件路径 | `--type`、`--robot` |
| `watch` | `--conversation` | `--interval`、`--direction`、`--once`、`--after`、`--limit` |
| `schedule` | `--conversation`、`--due` + `--text` / `--image` / `--link --title` | `--at`、`--desc`、`--robot` |
| `members` | `--conversation` | `--robot`、`--limit` |
| `tasks` | 无 | `--status`、`--robot`、`--limit` |
| `task:create` | `--conversation`、`--title`、`--due` | `--person-name`、`--person-id`、`--confirm-words`、`--remind-before`、`--repeat-minutes`、`--max-reminds`、`--note`、`--robot` |
| `task:confirm` / `task:cancel` | 任务 ID | 无 |

输出统一是 JSON；`watch` 每行一条消息（NDJSON）。

## 5. 退出码

| 退出码 | 含义 |
| --- | --- |
| 0 | 成功 |
| 1 | 参数错误或请求失败 |
| 2 | 未配置 Token / 未知命令 |
| 3 | 令牌无效或权限不足（401 / 403） |

## 6. 和 MCP / SDK 怎么选

| 场景 | 推荐 |
| --- | --- |
| AI 客户端（DSH / WorkBuddy / Hermes / OpenClaw） | [MCP](mcp.md) |
| 自己的后端 / 低代码 | [开发者 API + SDK](api.md) |
| 脚本、cron、运维、只有 shell 的 Agent | CLI |
