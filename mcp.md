# MCP 接入

MCP 用于让 WorkBuddy、DeepSeek Harness、龙虾 OpenClaw、爱马仕 Hermes 等客户端连接猿媒BOT。

## 1. 签发令牌

进入：

```text
个人中心 → MCP 接入
```

点「签发 MCP 令牌」。

令牌：

- 长期有效
- 只完整显示一次
- 可随时撤销
- 数据库只保存哈希

## 2. 配置 IP 白名单（可选）

固定服务器出口 IP 时填写：

```text
1.2.3.4
10.0.0.0/8
```

留空表示不限制来源。

## 3. 客户端

- [WorkBuddy](clients/workbuddy.md)
- [DeepSeek Harness](clients/deepseek-harness.md)
- [龙虾 OpenClaw](clients/openclaw.md)
- [爱马仕 Hermes](clients/hermes.md)

## 4. 验证

先调用：

```text
whoami
```

返回工作空间、机器人和工具范围后，再调用其他工具。

## 5. 工具清单

<!-- MCP-TOOLS-RISK:START -->
| 工具 | 能力 | 风险 |
| --- | --- | --- |
| whoami | 验证令牌，获取工作空间和机器人 | 低 |
| list_robots | 列出机器人 | 低 |
| list_conversations | 列出群聊和联系人 | 低 |
| read_messages | 读取消息，支持 after 增量 | 低 |
| search_messages | 搜索消息 | 低 |
| send_text | 发送文本 | 中 |
| send_image | 发送图片 URL | 中 |
| send_link | 发送链接卡片 | 中 |
| schedule_message | 定时发送 | 中 |
| get_group_members | 查询客户群成员花名册 | 低 |
| list_tasks | 列出任务跟踪 | 低 |
| create_task | 登记任务跟踪（到期提醒 + 确认词） | 中 |
| upload_media | 上传本地图片并返回可发送的 URL | 中 |
<!-- MCP-TOOLS-RISK:END -->

## 6. 返回格式

```json
{
  "status": true,
  "msg": "ok",
  "data": {}
}
```

失败时 `status: false`。

## 7. 限制

- 发送受授权、额度、10 秒/机器人限速
- 发送是异步队列，返回成功不代表已送达
- 图片必须是公网 URL
- 测试空间只能访问测试群

## 8. OAuth 授权（远程连接器）

Claude.ai、ChatGPT 这类**只支持 OAuth** 的远程 MCP 客户端，可以通过标准授权码 + PKCE 接入，不需要手动填令牌。

### 端点

| 端点 | 用途 |
| --- | --- |
| `/.well-known/oauth-authorization-server` | 授权服务器元数据 |
| `/.well-known/oauth-protected-resource` | 受保护资源元数据（`resource` = `/mcp`） |
| `POST /oauth/register` | 动态客户端注册（公开客户端，无密钥） |
| `GET /oauth/authorize` | 授权页（需先登录工作台） |
| `POST /oauth/authorize` | 同意 / 拒绝授权 |
| `POST /oauth/token` | 用授权码 + `code_verifier` 换令牌 |

### 流程

1. 客户端 `POST /oauth/register`，提交 `client_name` 与 `redirect_uris`（HTTPS；本机 `http://127.0.0.1` 例外）
2. 客户端带 PKCE（`code_challenge` / `S256`）打开 `/oauth/authorize`
3. 用户在授权页确认权限范围，允许后回跳 `redirect_uri?code=...&state=...`
4. 客户端用 `code` + `code_verifier` 调 `/oauth/token` 换 `access_token`
5. 之后按 `Authorization: Bearer <access_token>` 调 `/mcp`

### 说明

- 令牌与「个人中心 → MCP 接入」的凭据同构，可随时撤销；权限范围取授权时勾选子集（`read` / `send` / `schedule`）
- 只有工作空间所有者能完成授权
- 授权页要求浏览器已登录工作台；未登录会引导登录后回到授权页
- 当前不签发 refresh token（令牌长期有效）；授权码 10 分钟有效、一次性，落库保存（服务重启后仍可用）
- 动态注册客户端有数量上限与过期清理（30 天未使用且总量超阈值时回收）
- 回调地址必须是 HTTPS（本机调试地址除外），不做内网回调
