# OpenClaw + AIClient-2-API 搭建指南

## 参考链接

- OpenClaw Docker 镜像：https://hub.docker.com/r/justlikemaki/openclaw-docker-cn-im
- AIClient-2-API 配置 Kiro 转发：https://tech-it.ai.mcdchina.net/post/29

---

## 一、AIClient-2-API 配置

### 1.1 创建 configs/config.json

AIClient-2-API 默认使用 `gemini-cli-oauth`，需要手动创建配置文件切换到 `claude-kiro-oauth`：

```json
{
    "REQUIRED_API_KEY": "123456",
    "SERVER_PORT": 3000,
    "HOST": "0.0.0.0",
    "MODEL_PROVIDER": "claude-kiro-oauth",
    "PROVIDER_POOLS_FILE_PATH": "configs/provider_pools.json",
    "LOG_ENABLED": true,
    "LOG_LEVEL": "info",
    "LOG_DIR": "logs"
}
```

### 1.2 Kiro OAuth 凭证

通过 AIClient-2-API 的 Web UI（`http://localhost:3000`）进行 Kiro OAuth 授权，授权后凭证文件保存在 `configs/kiro/` 目录下。

凭证会自动刷新，如果过期需要重新授权。

### 1.3 Provider Pool 配置

`configs/provider_pools.json` 中配置 claude-kiro-oauth 的凭证池：

```json
{
  "claude-kiro-oauth": [
    {
      "KIRO_OAUTH_CREDS_FILE_PATH": "./configs/kiro/<你的凭证目录>/<凭证文件>.json",
      "uuid": "<自动生成>",
      "checkHealth": false,
      "isHealthy": true,
      "isDisabled": false
    }
  ]
}
```

### 1.4 可用模型

`claude-kiro-oauth` 支持的模型：
- `claude-opus-4-6`（最强）
- `claude-sonnet-4-6`
- `claude-opus-4-5`
- `claude-sonnet-4-5`
- `claude-haiku-4-5`

### 1.5 添加 /chat/completions 路由别名

OpenClaw 发送请求到 `/chat/completions`（不带 `/v1` 前缀），但 AIClient-2-API 默认只接受 `/v1/chat/completions`。

需要在 `src/services/api-manager.js` 中添加路由别名：

```javascript
// 修改前
if (path === '/v1/chat/completions') {

// 修改后
if (path === '/v1/chat/completions' || path === '/chat/completions') {
```

修改后需要重启 AIClient-2-API。

---

## 二、OpenClaw Docker 容器配置

### 2.1 docker-compose.yml

创建 `docker/docker-compose.openclaw.yml`：

```yaml
services:
  openclaw:
    image: justlikemaki/openclaw-docker-cn-im:latest
    ports:
      - "18789:18789"
    extra_hosts:
      - "localhost:host-gateway"
    environment:
      - API_KEY=123456
      - BASE_URL=http://host.docker.internal:3000/claude-kiro-oauth/v1
      - API_PROTOCOL=openai-completions
      - MODEL_ID=claude-opus-4-6
      - IMAGE_MODEL_ID=claude-sonnet-4-6
      - OPENCLAW_GATEWAY_TOKEN=123456
      - OPENCLAW_GATEWAY_PORT=18789
      - OPENCLAW_GATEWAY_BIND=lan
      - OPENCLAW_GATEWAY_DANGEROUSLY_DISABLE_DEVICE_AUTH=true
      - FEISHU_APP_ID=<你的飞书应用ID>
      - FEISHU_APP_SECRET=<你的飞书应用Secret>
    volumes:
      - ~/.openclaw:/home/node/.openclaw
```

### 2.2 环境变量说明

| 变量 | 说明 |
|------|------|
| `API_KEY` | AIClient-2-API 的 REQUIRED_API_KEY |
| `BASE_URL` | AIClient 地址，容器内用 `host.docker.internal` 访问宿主机 |
| `API_PROTOCOL` | 协议类型，固定 `openai-completions` |
| `MODEL_ID` | 主模型 |
| `IMAGE_MODEL_ID` | 图像模型 |
| `OPENCLAW_GATEWAY_TOKEN` | 网关访问密码 |
| `OPENCLAW_GATEWAY_PORT` | 网关端口 |
| `OPENCLAW_GATEWAY_BIND` | 绑定模式，`lan` 表示局域网可访问 |
| `OPENCLAW_GATEWAY_DANGEROUSLY_DISABLE_DEVICE_AUTH` | 禁用设备认证 |
| `FEISHU_APP_ID` / `FEISHU_APP_SECRET` | 飞书机器人配置 |

> **注意**：容器的 init 脚本（`/usr/local/bin/init.sh`）会在每次启动时用环境变量覆盖配置文件，所以必须通过环境变量配置，不能只改配置文件。

### 2.3 启动方式

```bash
cd ~/AndroidStudioProjects/AIClient-2-API
docker compose -f docker/docker-compose.openclaw.yml up -d
```

也可以在 Docker Desktop GUI 中直接点击启动按钮。

### 2.4 访问 OpenClaw

浏览器打开：`http://localhost:18789?token=123456`

---

## 三、踩坑记录与解决方案

### 3.1 Connection error（容器无法连接 AIClient）

**现象**：OpenClaw 发送消息后显示 "Connection error"。

**原因**：OpenClaw 内部 agent 连接 `127.0.0.1:3000`，但容器内的 localhost 不是宿主机。

**解决**：在 docker-compose 中添加 `extra_hosts`：
```yaml
extra_hosts:
  - "localhost:host-gateway"
```
这样容器内的 localhost 会解析到宿主机 IP。

### 3.2 HTTP 404（路径不匹配）

**现象**：容器能连通 AIClient，但返回 404。

**原因**：OpenClaw 发请求到 `/chat/completions`，AIClient 只接受 `/v1/chat/completions`。

**解决**：在 `src/services/api-manager.js` 中添加 `/chat/completions` 路由别名（见 1.5 节）。

### 3.3 No healthy provider found in pool

**现象**：从容器内 curl AIClient 返回此错误。

**原因**：Kiro OAuth 凭证过期或未正确配置。

**解决**：通过 AIClient Web UI 重新授权 Kiro OAuth，确保 `provider_pools.json` 中的凭证路径正确且 `isHealthy: true`。

### 3.4 gateway.bind 配置错误

**现象**：网关无法启动。

**原因**：`OPENCLAW_GATEWAY_BIND` 必须使用模式名称（如 `lan`），不能用 IP 地址（如 `0.0.0.0`）。

### 3.5 内网链接无法访问

**现象**：OpenClaw 的 `web_fetch` 工具访问内网链接时报 `Blocked: resolves to private/internal/special-use IP address`。

**原因**：OpenClaw 内置 SSRF 防护，默认拦截解析到内网 IP 的请求。

**解决**：在 `~/.openclaw/openclaw.json` 的 `browser` 配置下添加 `ssrfPolicy`：

```json
{
  "browser": {
    "headless": true,
    "noSandbox": true,
    "defaultProfile": "openclaw",
    "executablePath": "/usr/bin/chromium",
    "ssrfPolicy": {
      "dangerouslyAllowPrivateNetwork": true,
      "allowedHostnames": [
        "pmo.mcd.com.cn",
        "gitlab-ex.mcd.com.cn"
      ]
    }
  }
}
```

修改后重启容器生效。需要添加更多内网域名时，往 `allowedHostnames` 数组里加就行。

---

## 四、飞书机器人配置

### 4.1 OpenClaw 侧配置

在 docker-compose 环境变量中添加：
```yaml
- FEISHU_APP_ID=<应用ID>
- FEISHU_APP_SECRET=<应用Secret>
```

容器启动后会自动通过 WebSocket 长连接接收飞书消息。

### 4.2 飞书开放平台配置

1. 进入 [飞书开放平台](https://open.feishu.cn/app) → 你的应用
2. 开启「机器人」能力
3. 「事件与回调」→ 订阅方式选择「使用长连接接收事件/回调」
4. 添加事件：`im.message.receive_v1`（接收消息）
5. 「权限管理」中开通 `im:message:receive_v1` 等消息相关权限
6. 保存并重新发布应用版本

### 4.3 使用方式

- 群聊中 @机器人 发消息
- 发送飞书云文档链接，机器人可通过 `feishu_doc` 工具读取内容
- 直接发文件附件无法识别，需要先上传到飞书云文档再发链接

### 4.4 其他支持的 IM 渠道

OpenClaw 还支持：
- 企业微信（需要 `WECOM_TOKEN` + `WECOM_ENCODING_AES_KEY`，需公网可达或内网穿透）
- 钉钉（需要 `DINGTALK_CLIENT_ID` + `DINGTALK_CLIENT_SECRET`）
- Telegram（需要 `TELEGRAM_BOT_TOKEN`）
- QQ 机器人（需要 `QQBOT_APP_ID` + `QQBOT_CLIENT_SECRET`）
- NapCat（需要 `NAPCAT_REVERSE_WS_PORT`）

---

## 五、IDENTITY.md 配置

OpenClaw 的身份配置文件位于 `~/.openclaw/workspace/IDENTITY.md`，可自定义机器人人设：

```markdown
# Name
小助手

# Creature
AI Assistant

# Vibe
友好、专业、高效

# Emoji
🤖

# Avatar
robot
```

---

## 六、OpenClaw 与 MCP 的关系

OpenClaw 目前**不支持 MCP 协议**。源码中明确忽略 MCP 配置（`ignoring N MCP servers`）。OpenClaw 使用自己的插件系统（`plugins` + `extensions`）。

Kiro IDE 中配置的 MCP 服务（如 Jira MCP、Confluence MCP）只能在 Kiro 中使用，无法被 OpenClaw 调用。
