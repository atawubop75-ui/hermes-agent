# 消息网关

## 概述

Hermes Agent 的消息网关（Gateway）提供多平台消息集成，支持 Telegram、Discord、Slack、WhatsApp 等即时通讯平台。

## 架构

```
                    ┌─────────────────┐
                    │  Gateway Runner  │
                    │   (run.py)       │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
    ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
    │ Telegram │        │ Discord │        │  Slack   │
    │ Adapter  │        │ Adapter │        │ Adapter  │
    └────┬────┘        └────┬────┘        └────┬────┘
         │                   │                   │
    ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
    │ Telegram │        │ Discord │        │  Slack  │
    │   Bot    │        │   Bot   │        │   Bot   │
    └─────────┘        └─────────┘        └─────────┘
```

## 启动网关

```bash
# 启动网关
hermes gateway start

# 配置网关
hermes gateway setup

# 停止网关
hermes gateway stop

# 查看状态
hermes gateway status
```

## 平台适配器

### 支持的平台

| 平台 | 文件 | 特点 |
|------|------|------|
| Telegram | `platforms/telegram.py` | Bot API, Webhook |
| Discord | `platforms/discord.py` | Bot API, Slash Commands |
| Slack | `platforms/slack.py` | Bolt SDK |
| WhatsApp | `platforms/whatsapp.py` | WhatsApp Business API |
| Signal | `platforms/signal.py` | Signal Messenger |
| Matrix | `platforms/matrix.py` | Matrix protocol |
| Mattermost | `platforms/mattermost.py` | Mattermost webhook |
| Email | `platforms/email.py` | SMTP/IMAP |
| SMS | `platforms/sms.py` | Twilio 等 |
| 飞书 | `platforms/feishu.py` | 文档、评论 |
| 钉钉 | `platforms/dingtalk.py` | 钉钉群消息 |
| 企业微信 | `platforms/wecom.py` | 企业微信应用 |
| 微信 | `platforms/weixin.py` | 微信消息 |
| Home Assistant | `platforms/homeassistant.py` | HA 集成 |
| QQBot | `platforms/qqbot/` | QQ 机器人 |
| BlueBubbles | `platforms/bluebubbles.py` | iMessage 桥接 |

### BasePlatform 接口

所有平台适配器继承 `BasePlatform`：

```python
# gateway/platforms/base.py
class BasePlatform:
    platform_name: str
    supports_sessions: bool = True

    async def connect(self): ...
    async def disconnect(self): ...
    async def send_message(self, session_key: str, message: str): ...
    async def send_file(self, session_key: str, file_path: str): ...
    async def handle_update(self, update: dict): ...
```

## 会话管理

### SessionKey 格式

```
{platform}:{user_id}@{chat_id}
```

示例：
- `telegram:123456@789012` - Telegram 用户
- `discord:987654:123456` - Discord 频道/用户

### 会话隔离

每个平台适配器维护独立的会话状态：

```python
# gateway/session.py
class SessionManager:
    def get_session(self, session_key: str) -> Session: ...
    def create_session(self, session_key: str) -> Session: ...
    def delete_session(self, session_key: str) -> None: ...
```

## 消息处理

### 消息流程

```
平台消息 → 适配器 → 解析 → Gateway Runner
                                    ↓
                            命令处理 (/stop, /new...)
                                    ↓
                            AIAgent 处理
                                    ↓
                            返回响应
                                    ↓
                            适配器发送
```

### 消息队列

```python
# gateway/platforms/base.py
class BasePlatform:
    _pending_messages: asyncio.Queue

    async def handle_update(self, update: dict):
        if session_key in self._active_sessions:
            self._pending_messages.put(update)
        else:
            await self._dispatch_direct(update)
```

## 命令处理

### Gateway 命令

| 命令 | 说明 |
|------|------|
| `/stop` | 停止当前任务 |
| `/new` | 开始新对话 |
| `/queue` | 查看队列 |
| `/status` | 查看状态 |
| `/approve` | 批准操作 |
| `/deny` | 拒绝操作 |

### 平台特定命令

```python
# Telegram
/start - 开始
/help - 帮助
/model - 切换模型

# Discord
/hermes - 调用 Hermes
```

## Webhook 配置

### Telegram Webhook

```yaml
platforms:
  telegram:
    webhook:
      enabled: true
      url: "https://your-domain.com/telegram/webhook"
      secret: "${TELEGRAM_WEBHOOK_SECRET}"
```

### Discord Webhook

```yaml
platforms:
  discord:
    webhook:
      enabled: true
      port: 8080
```

## 安全

### 消息验证

每个平台使用其自身的验证机制：

- Telegram: `X-Telegram-Bot-Api-Secret-Token`
- Discord: `X-Signature-Ed25519`
- Slack: `Signing Secret`

### 访问控制

```yaml
platforms:
  telegram:
    allowed_users:
      - 123456789
    blocked_users: []
```

## 部署

### Docker 部署

```yaml
# docker-compose.yml
services:
  hermes-gateway:
    build: .
    environment:
      - HERMES_HOME=/data
    volumes:
      - ./data:/data
    ports:
      - "8000:8000"
```

### 生产环境

```bash
# 启动网关（生产模式）
hermes gateway start --production

# 使用 systemd
sudo hermes gateway install-service
```
