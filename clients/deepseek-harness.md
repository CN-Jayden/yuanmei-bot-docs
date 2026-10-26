# DeepSeek Harness 接入

推荐用 MCP 管理插件，在界面里配置，不需要环境变量。

## 1. 签发令牌

```text
个人中心 → MCP 接入 → 签发 MCP 令牌
```

## 2. 安装 MCP 管理插件

在 DSH：

```text
插件 → 添加插件 → 搜索 dsh-mcp → 安装
```

项目地址：

https://github.com/ArvinQi/dsh-mcp

注意：这是第三方插件，安装前确认作者和源码。

## 3. 添加猿媒BOT

打开：

```text
设置 → MCP → 添加服务器
```

填写：

```text
名称：yuanmei
传输方式：streamable-http
URL：https://chatbot.apetd.com/mcp
请求头：Authorization: Bearer <TOKEN>
```

保存后点「连接测试」。

## 4. 验证

在会话中调用：

```text
mcp__yuanmei__whoami
```

返回工作空间和工具范围即可。

## 高级：安装猿媒BOT DSH 插件

如果希望插件自动注册 `yuanmei`，可以改用：

```text
https://github.com/CN-Jayden/yuanmei-bot-dsh-mcp
```

安装后需要设置：

```bash
YUANMEI_MCP_TOKEN='你的令牌'
```

并重启 DSH。

两种方式二选一，不要同时启用，否则 `yuanmei` 会重复注册。
