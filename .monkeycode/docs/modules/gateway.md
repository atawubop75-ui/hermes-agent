# Gateway 模块 (gateway/)

## 概述

`gateway/` 目录包含消息网关的实现，提供多平台消息集成、会话管理、Webhook 处理等功能。

## 目录结构

```
gateway/
├── __init__.py
├── run.py                 # 网关入口 (~2.5k LOC)
├── config.py              # 网关配置
├── session.py             # 会话管理
├── session_context.py     # 会话上下文
├── channel_directory.py   # 频道目录
├── hooks.py               # 钩子系统
├── status.py              # 状态管理
├── delivery.py             # 消息投递
├── display_config.py      # 显示配置
├── restart.py              # 重启处理
├── sticker_cache.py        # 贴纸缓存
├── stream_consumer.py       # 流消费者
├── mirror.py               # 镜像
├── pairing.py              # 配对
├── whitelist.py            # 白名单
├── platforms/              # 平台适配器
│   ├── __init__.py
│   ├── base.py             # 基础适配器 (~2.5k LOC)
│   ├── telegram.py         # Telegram (~3k LOC)
│   ├── discord.py          # Discord (~3.5k LOC)
│   ├── slack.py            # Slack (~2k LOC)
│   ├── whatsapp.py         # WhatsApp (~1k LOC)
│   ├── signal.py           # Signal (~1k LOC)
│   ├── matrix.py           # Matrix (~2k LOC)
│   ├── mattermost.py       # Mattermost
│   ├── email.py            # Email
│   ├── sms.py              # SMS
│   ├── feishu.py           # 飞书 (~4k LOC)
│   ├── dingtalk.py         # 钉钉 (~1.5k LOC)
│   ├── wecom.py            # 企业微信 (~1.5k LOC)
│   ├── weixin.py           # 微信 (~3.5k LOC)
│   ├── homeassistant.py    # Home Assistant
│   ├── qqbot/              # QQ 机器人
│   ├── webhook.py          # 通用 Webhook
│   ├── api_server.py       # REST API 服务器 (~2.5k LOC)
│   ├── helpers.py          # 辅助函数
│   ├── ADDING_A_PLATFORM.md # 添加平台指南
│   └── ...
├── builtin_hooks/          # 内置钩子
│   ├── boot-md.py
│   └── ...
└── ...
```

## 网关入口 (run.py)

### GatewayRunner

```python
class GatewayRunner:
    """消息网关运行器"""

    def __init__(self, config: dict):
        self.config = config
        self.platforms: dict[str, BasePlatform] = {}
        self.session_manager = SessionManager()

    async def start(self):
        """启动所有平台适配器"""

    async def stop(self):
        """停止所有平台适配器"""

    async def restart(self):
        """重启网关"""
```

### 启动方式

```bash
# 从 CLI
hermes gateway start

# 从模块
python -m gateway.run

# 使用 systemd
hermes gateway install-service
```

## 会话管理 (session.py)

### SessionManager

```python
class SessionManager:
    """会话管理器"""

    def create_session(
        self,
        session_key: str,
        platform: str,
        user_id: str,
        chat_id: str,
        **kwargs
    ) -> Session: ...

    def get_session(self, session_key: str) -> Session: ...

    def delete_session(self, session_key: str): ...

    def get_or_create_session(
        self,
        platform: str,
        user_id: str,
        chat_id: str,
        **kwargs
    ) -> Session: ...
```

### Session

```python
@dataclass
class Session:
    session_key: str
    platform: str
    user_id: str
    chat_id: str
    created_at: datetime
    last_activity: datetime
    agent: AIAgent = None
    messages: list = field(default_factory=list)
```

## 平台适配器 (platforms/)

### BasePlatform

所有平台适配器的基类：

```python
class BasePlatform:
    platform_name: str = "base"
    supports_sessions: bool = True

    _pending_messages: asyncio.Queue
    _active_sessions: set

    async def connect(self) -> None:
        """连接到平台"""

    async def disconnect(self) -> None:
        """断开连接"""

    async def send_message(
        self,
        session_key: str,
        message: str,
        **kwargs
    ) -> None:
        """发送消息"""

    async def send_file(
        self,
        session_key: str,
        file_path: str,
        **kwargs
    ) -> None:
        """发送文件"""

    async def handle_update(self, update: dict) -> None:
        """处理平台更新"""

    async def handle_command(
        self,
        session_key: str,
        command: str
    ) -> None:
        """处理命令"""
```

### 适配器特性

| 平台 | 文件大小 | 特性 |
|------|----------|------|
| Telegram | ~138KB | Bot API, Webhook, Commands Menu |
| Discord | ~172KB | Bot API, Slash Commands, Components |
| Slack | ~102KB | Bolt SDK, Modal |
| WhatsApp | ~45KB | Business API, Media |
| Matrix | ~106KB | E2E Encryption, VoIP |
| 飞书 | ~192KB | 文档, 云盘, 评论, 审批 |
| 钉钉 | ~56KB | 群消息, 回调 |
| 企业微信 | ~77KB | 应用消息, 网页 |

## 消息处理

### 消息流程

```
平台更新
    ↓
适配器.handle_update()
    ↓
验证和解析
    ↓
命令检查 (/stop, /new...)
    ↓
活动会话？ → 队列消息
    ↓
空闲会话？ → 直接分发
    ↓
AIAgent 处理
    ↓
响应适配器
    ↓
平台发送
```

### 消息队列

```python
async def handle_update(self, update: dict):
    if session_key in self._active_sessions:
        # 队列消息等待处理
        await self._pending_messages.put(update)
    else:
        # 直接分发
        await self._dispatch_direct(update)
```

## 命令处理

### Gateway 命令

这些命令绕过活动会话检查：

```python
GATEWAY_KNOWN_COMMANDS = frozenset({
    "stop",
    "new",
    "reset",
    "queue",
    "status",
    "approve",
    "deny",
})
```

### 命令分发

```python
async def _process_command(
    self,
    session_key: str,
    command: str
) -> None:
    if canonical in GATEWAY_KNOWN_COMMANDS:
        await self._dispatch_gateway_command(
            session_key,
            canonical,
            args
        )
    else:
        await self._forward_to_agent(
            session_key,
            command
        )
```

## Webhook 处理

### Telegram Webhook

```python
async def telegram_webhook(request: Request):
    """处理 Telegram Webhook"""

    # 验证 secret token
    # 解析更新
    # 转发到 handle_update
```

### Discord Webhook

```python
async def discord_webhook(request: Request):
    """处理 Discord Webhook"""

    # 验证 signature
    # 解析更新
    # 转发到 handle_update
```

### 通用 Webhook

```python
async def generic_webhook(request: Request):
    """处理通用 Webhook"""

    # 解析负载
    # 转换为内部格式
    # 转发到 handle_update
```

## 配置

### Gateway 配置

```yaml
gateway:
  host: "0.0.0.0"
  port: 8080
  log_level: "info"

  platforms:
    telegram:
      enabled: true
      bot_token: "${TELEGRAM_BOT_TOKEN}"

    discord:
      enabled: true
      bot_token: "${DISCORD_BOT_TOKEN}"
```

### 平台配置

每个平台有特定的配置项：

```python
# Telegram
{
    "bot_token": "...",
    "webhook": {
        "enabled": true,
        "url": "https://...",
        "secret": "..."
    },
    "allowed_users": ["..."],
    "commands": [...]
}

# Discord
{
    "bot_token": "...",
    "application_id": "...",
    "signing_secret": "...",
    "guild_id": "..."
}
```

## 消息投递 (delivery.py)

```python
class DeliveryManager:
    """消息投递管理器"""

    async def deliver(
        self,
        session_key: str,
        message: str,
        retry: bool = True
    ) -> bool: ...

    async def deliver_with_typing(
        self,
        session_key: str,
        message: str
    ) -> bool: ...
```

## 状态管理 (status.py)

```python
class GatewayStatus:
    """网关状态"""

    running: bool
    platforms: dict
    active_sessions: int
    uptime: timedelta

    def to_dict(self) -> dict: ...
```

## 添加新平台

参考 `platforms/ADDING_A_PLATFORM.md`：

1. 创建新适配器类
2. 继承 `BasePlatform`
3. 实现必需方法
4. 在 `run.py` 注册
5. 添加配置选项
