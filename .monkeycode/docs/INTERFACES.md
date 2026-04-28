# Hermes Agent 接口文档

## 1. 公共 API 接口

### 1.1 AIAgent 类 (run_agent.py)

```python
class AIAgent:
    def __init__(
        self,
        base_url: str = None,
        api_key: str = None,
        provider: str = None,
        api_mode: str = None,              # "chat_completions" | "codex_responses"
        model: str = "",
        max_iterations: int = 90,
        enabled_toolsets: list = None,
        disabled_toolsets: list = None,
        quiet_mode: bool = False,
        save_trajectories: bool = False,
        platform: str = None,
        session_id: str = None,
        skip_context_files: bool = False,
        skip_memory: bool = False,
        credential_pool=None,
        **kwargs
    ): ...

    def chat(self, message: str) -> str:
        """简单接口 - 返回最终响应字符串"""

    def run_conversation(
        self,
        user_message: str,
        system_message: str = None,
        conversation_history: list = None,
        task_id: str = None
    ) -> dict:
        """完整接口 - 返回包含 final_response 和 messages 的字典"""
```

### 1.2 工具注册表 (tools/registry.py)

```python
class ToolRegistry:
    def register(
        self,
        name: str,
        toolset: str,
        schema: dict,
        handler: Callable,
        check_fn: Callable = None,
        requires_env: list = None,
        is_async: bool = False,
        description: str = None,
        emoji: str = None,
        max_result_size_chars: int = None
    ) -> None: ...

    def get_tool(self, name: str) -> ToolEntry: ...
    def get_toolset(self, name: str) -> list: ...
    def get_all_tools(self) -> list: ...
    def get_toolsets(self) -> set: ...
    def check_tool_requirements(self, name: str) -> tuple: ...
```

### 1.3 会话存储 (hermes_state.py)

```python
class SessionDB:
    def create_session(self, session_id: str, **kwargs) -> dict: ...
    def get_session(self, session_id: str) -> dict: ...
    def update_session(self, session_id: str, **kwargs) -> None: ...
    def delete_session(self, session_id: str) -> None: ...
    def search_sessions(self, query: str, limit: int = 10) -> list: ...
    def add_message(self, session_id: str, role: str, content: str, **kwargs) -> dict: ...
    def get_messages(self, session_id: str, limit: int = 100) -> list: ...
```

### 1.4 CLI 主入口 (hermes_cli/main.py)

```python
def main():
    """Hermes CLI 主入口"""
    # 处理命令行参数并启动 CLI 或 Gateway
```

## 2. 工具接口

### 2.1 工具注册示例

```python
from tools.registry import registry

def check_requirements() -> bool:
    return bool(os.getenv("EXAMPLE_API_KEY"))

def example_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"success": True, "data": "..."})

registry.register(
    name="example_tool",
    toolset="example",
    schema={
        "name": "example_tool",
        "description": "示例工具",
        "parameters": {
            "type": "object",
            "properties": {
                "param": {"type": "string", "description": "参数"}
            },
            "required": ["param"]
        }
    },
    handler=lambda args, **kw: example_tool(
        param=args.get("param", ""),
        task_id=kw.get("task_id")
    ),
    check_fn=check_requirements,
    requires_env=["EXAMPLE_API_KEY"],
)
```

### 2.2 工具 Schema 格式

```json
{
  "name": "tool_name",
  "description": "工具描述",
  "parameters": {
    "type": "object",
    "properties": {
      "param_name": {
        "type": "string",
        "description": "参数描述"
      }
    },
    "required": ["param_name"]
  }
}
```

### 2.3 工具返回格式

所有工具处理器必须返回 JSON 字符串：

```python
# 成功
return json.dumps({"success": True, "data": {...}})

# 错误
return json.dumps({"success": False, "error": "错误信息"})
```

## 3. 消息格式

### 3.1 OpenAI 格式消息

```python
messages = [
    {"role": "system", "content": "系统提示"},
    {"role": "user", "content": "用户消息"},
    {"role": "assistant", "content": "助手回复", "tool_calls": [...]},
    {"role": "tool", "tool_call_id": "...", "content": "工具结果"}
]
```

### 3.2 Reasoning 内容

```python
assistant_msg = {
    "role": "assistant",
    "content": "...",
    "reasoning": "推理过程内容"
}
```

## 4. 配置接口

### 4.1 config.yaml 结构

```yaml
# 基本配置
model: "claude-sonnet-4-20250514"
provider: "anthropic"

# 显示配置
display:
  skin: "default"
  spinner: true
  color: true

# 工具配置
tools:
  enabled: ["web", "terminal", "file"]
  disabled: []

# 记忆配置
memory:
  provider: "honcho"

# 平台配置
platforms:
  telegram:
    enabled: true
    bot_token: "${TELEGRAM_BOT_TOKEN}"

# 终端配置
terminal:
  backend: "local"
  cwd: "/path/to/working/directory"
```

### 4.2 环境变量

| 变量 | 说明 |
|------|------|
| `HERMES_HOME` | 配置目录路径 |
| `HERMES_QUIET` | 静默模式 |
| `ANTHROPIC_API_KEY` | Anthropic API 密钥 |
| `OPENAI_API_KEY` | OpenAI API 密钥 |
| `TELEGRAM_BOT_TOKEN` | Telegram Bot Token |
| `DISCORD_BOT_TOKEN` | Discord Bot Token |

## 5. Gateway 接口

### 5.1 Gateway 启动

```bash
hermes gateway start
hermes gateway setup
```

### 5.2 平台配置

每个平台适配器需要特定配置：

| 平台 | 必需配置 |
|------|----------|
| Telegram | bot_token |
| Discord | bot_token |
| Slack | bot_token, signing_secret |
| WhatsApp | phone_number_id, access_token |
| Signal | phone_number, signal_cli_path |
| Matrix | homeserver_url, access_token |
| Email | smtp_host, smtp_port, email, password |
| 飞书 | app_id, app_secret |
| 钉钉 | client_id, client_secret |
| 企业微信 | corp_id, corp_secret |
| Home Assistant | ha_url, long_lived_access_token |

## 6. 插件接口

### 6.1 通用插件

```python
class PluginContext:
    def register_tool(self, name: str, schema: dict, handler: Callable): ...
    def register_cli_command(self, name: str, parser: argparse.ArgumentParser): ...
    def register_hook(self, hook_name: str, callback: Callable): ...

def register(ctx: PluginContext):
    """插件入口函数"""
    ctx.register_hook("pre_tool_call", my_pre_tool_hook)
    ctx.register_hook("post_tool_call", my_post_tool_hook)
```

### 6.2 记忆提供者

```python
from agent.memory_provider import MemoryProvider

class MyMemoryProvider(MemoryProvider):
    async def setup(self, config: dict): ...
    async def sync_turn(self, turn_messages: list): ...
    async def prefetch(self, query: str): ...
    async def shutdown(self): ...
    async def post_setup(self, hermes_home: Path, config: dict): ...
```

## 7. 技能接口

### 7.1 SKILL.md 格式

```markdown
---
name: skill-name
description: 技能描述
version: "1.0.0"
platforms: [linux, macos]
metadata:
  hermes:
    tags: [tag1, tag2]
    category: "category-name"
    config:
      some_setting:
        type: string
        required: true
---

# Skill Content

技能的具体实现内容...
```

### 7.2 技能命令

```bash
hermes skills list              # 列出所有技能
hermes skills install <name>    # 安装技能
hermes skills uninstall <name>  # 卸载技能
```

## 8. WebSocket API (Dashboard)

### 8.1 PTY WebSocket

```
GET /api/pty?token=<session_token>
Upgrade: websocket
```

### 8.2 认证

使用临时的 `_SESSION_TOKEN` 进行认证。

## 9. MCP 接口

### 9.1 MCP 服务器连接

```python
# 通过 hermes_cli/mcp_config.py 配置
mcp_servers:
  - name: "example-server"
    command: ["npx", "mcp-server-example"]
    env:
      EXAMPLE_KEY: "${EXAMPLE_API_KEY}"
```

### 9.2 MCP OAuth

```python
# tools/mcp_oauth.py
class MCPOAuthManager:
    async def get_authorization_url(self, server_name: str) -> str: ...
    async def exchange_code(self, server_name: str, code: str) -> dict: ...
    async def refresh_token(self, server_name: str) -> dict: ...
```

## 10. 回调接口

### 10.1 工具回调

```python
# 工具执行前
def pre_tool_call(tool_name: str, arguments: dict, task_id: str): ...

# 工具执行后
def post_tool_call(tool_name: str, result: str, task_id: str): ...
```

### 10.2 LLM 回调

```python
# LLM 调用前
def pre_llm_call(messages: list, model: str): ...

# LLM 调用后
def post_llm_call(response: dict, model: str): ...
```

### 10.3 会话回调

```python
# 会话开始
def on_session_start(session_id: str): ...

# 会话结束
def on_session_end(session_id: str): ...
```
