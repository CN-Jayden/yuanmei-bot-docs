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

| 工具 | 能力 | 风险 |
| --- | --- | --- |
| whoami | 验证令牌 | 低 |
| list_robots | 列出机器人 | 低 |
| list_conversations | 列出群聊和联系人 | 低 |
| read_messages | 读取消息，支持 after 增量 | 低 |
| search_messages | 搜索消息 | 低 |
| send_text | 发送文本 | 中 |
| send_image | 发送图片 URL | 中 |
| send_link | 发送链接卡片 | 中 |
| schedule_message | 定时发送 | 中 |

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
