# 猿媒BOT

企业微信私域运营工作台，把机器人、群聊、客户、AI 知识库和自动化流程放在一个后台里管理。

它适合：

- 用企业微信做客户群运营的团队
- 需要 AI 群答疑的学校、门店、服务机构
- 需要把订单、工单、告警自动推送到企微群的业务团队
- 已经在用猿媒BOT，希望接 MCP / API 的客户

## 能做什么

| 场景 | 能力 |
| --- | --- |
| 群消息 | 收发消息、读取记录、搜索消息、图片/链接卡片 |
| 客户运营 | 活码、欢迎语、标签、客户列表、群运营 |
| 自动化 | 关键词回复、跨群转发、AI 问答、业务接口应答 |
| 定时任务 | 预约发送、到点提醒 |
| 内容 | 公众号文章卡片、视频号卡片 |
| AI | 知识库上传、模型服务、群内答疑、文档图片识别 |
| 接入 | Webhook 推送、MCP、服务端发送 API |
| 任务 | 到店/交稿提醒、确认词关闭、超时跟进 |

## 快速开始

1. 打开 [猿媒BOT 工作台](https://chatbot.apetd.com)
2. 注册账号，或点「体验演示」
3. 由平台交付机器人 secret
4. 在「机器人连接」绑定机器人并同步群聊
5. 配置回调地址，在群里发消息验证

完整步骤见 [快速开始](quick-start.md)。

## 接入方式

### MCP

适合 WorkBuddy、DeepSeek Harness、龙虾 OpenClaw、爱马仕 Hermes、AI Agent。

- 长期令牌，可撤销
- 可选 IP 白名单
- 工具带风险等级
- 支持读消息、搜消息、发文本/图片/链接、定时发送

见 [MCP 接入](mcp.md)。

### 服务端发送 API

适合客户自己的后台生成回复后发回群。

```http
POST https://chatbot.apetd.com/api/open/send
Authorization: Bearer <token>
```

见 [发送 API](api.md)。

## 客户端支持

- [WorkBuddy](clients/workbuddy.md)
- [DeepSeek Harness](clients/deepseek-harness.md)
- [龙虾 OpenClaw](clients/openclaw.md)
- [爱马仕 Hermes](clients/hermes.md)

## 边界

当前不支持：

- 微信语音消息转发
- 微信群入站图片直接显示
- 小程序/名片发送
- 视频号正文解析

这些不影響文本群运营和 AI 答疑，但不要作为卖点承诺。

## 文档目录

- [快速开始](quick-start.md)
- [MCP 接入](mcp.md)
- [发送 API](api.md)
- [AI 知识库](ai-knowledge.md)
- [常见问题](faq.md)
- [安全与数据](security.md)

## 支持

遇到问题，请把以下信息发给平台：

- 工作空间名称
- 机器人名称
- 发生时间
- 群名称
- 页面截图或接口返回
