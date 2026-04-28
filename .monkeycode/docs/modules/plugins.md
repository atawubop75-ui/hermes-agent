# 插件系统 (plugins/)

## 概述

Hermes Agent 的插件系统提供可扩展的模块化架构，支持通用插件和特定类型的插件（如记忆提供者、上下文引擎等）。

## 插件类型

```
plugins/
├── memory/           # 记忆提供者插件
├── context_engine/   # 上下文引擎插件
├── image_gen/        # 图像生成插件
├── dashboard/        # 仪表板插件
├── google_meet/      # Google Meet 插件
├── spotify/          # Spotify 插件
├── disk-cleanup/     # 磁盘清理插件
├── strike-freedom-cockpit/  # 自定义插件
└── ...
```

## 通用插件

### 插件结构

```
plugins/<name>/
├── __init__.py      # 必须包含 register(ctx) 函数
├── plugin.json      # 插件元数据（可选）
└── ...              # 其他模块
```

### register 函数

```python
# plugins/my_plugin/__init__.py
def register(ctx: PluginContext):
    """插件入口点"""

    # 注册钩子
    ctx.register_hook("pre_tool_call", my_pre_hook)
    ctx.register_hook("post_tool_call", my_post_hook)

    # 注册工具
    ctx.register_tool(
        name="my_tool",
        schema={...},
        handler=my_tool_handler
    )

    # 注册 CLI 命令
    ctx.register_cli_command("mycommand", add_mycommand_args)
```

### PluginContext 接口

```python
class PluginContext:
    def register_hook(
        self,
        hook_name: str,
        callback: Callable
    ) -> None:
        """注册生命周期钩子"""

    def register_tool(
        self,
        name: str,
        schema: dict,
        handler: Callable
    ) -> None:
        """注册工具"""

    def register_cli_command(
        self,
        name: str,
        arg_parser: Callable[[ArgumentParser], None]
    ) -> None:
        """注册 CLI 子命令"""
```

### 可用钩子

| 钩子名称 | 参数 | 说明 |
|----------|------|------|
| `pre_tool_call` | tool_name, args, task_id | 工具调用前 |
| `post_tool_call` | tool_name, result, task_id | 工具调用后 |
| `pre_llm_call` | messages, model | LLM 调用前 |
| `post_llm_call` | response, model | LLM 调用后 |
| `on_session_start` | session_id | 会话开始 |
| `on_session_end` | session_id | 会话结束 |

### 插件发现

```python
# hermes_cli/plugins.py
class PluginManager:
    def discover_plugins(self):
        """从多个来源发现插件"""
        # 1. ~/.hermes/plugins/
        # 2. ./.hermes/plugins/
        # 3. pip 入口点

    def load_plugin(self, name: str):
        """加载插件"""
```

### 插件命令

```bash
# 列出插件
hermes plugins list

# 安装插件
hermes plugins install <path-or-url>

# 卸载插件
hermes plugins uninstall <name>

# 启用/禁用插件
hermes plugins enable <name>
hermes plugins disable <name>
```

## 记忆提供者插件

### MemoryProvider 抽象基类

```python
# agent/memory_provider.py
from abc import ABC, abstractmethod

class MemoryProvider(ABC):
    @abstractmethod
    async def setup(self, config: dict) -> None:
        """初始化提供者"""

    @abstractmethod
    async def sync_turn(
        self,
        turn_messages: list[dict]
    ) -> None:
        """同步对话轮次到记忆"""

    @abstractmethod
    async def prefetch(self, query: str) -> list[dict]:
        """预取相关记忆"""

    @abstractmethod
    async def shutdown(self) -> None:
        """关闭提供者"""

    async def post_setup(
        self,
        hermes_home: Path,
        config: dict
    ) -> None:
        """可选：设置后回调（用于设置向导集成）"""
```

### 内置提供者

| 提供者 | 目录 | 说明 |
|--------|------|------|
| Honcho | `plugins/memory/honcho/` | dialectic user modeling |
| Mem0 | `plugins/memory/mem0/` | Mem0 记忆服务 |
| Supermemory | `plugins/memory/supermemory/` | Supermemory |
| Byterover | `plugins/memory/byterover/` | Byterover 记忆 |
| Hindsight | `plugins/memory/hindsight/` | Hindsight |
| Holographic | `plugins/memory/holographic/` | Holographic |
| OpenViking | `plugins/memory/openviking/` | OpenViking |
| RetainDB | `plugins/memory/retaindb/` | RetainDB |

### Honcho 提供者

```python
# plugins/memory/honcho/honcho.py
class HonchoMemoryProvider(MemoryProvider):
    async def setup(self, config: dict):
        self.client = HonchoClient(
            api_key=config.get("api_key")
        )

    async def sync_turn(self, turn_messages: list):
        await self.client.sync(turn_messages)

    async def prefetch(self, query: str) -> list:
        return await self.client.search(query)
```

### 配置

```yaml
memory:
  provider: "honcho"
  honcho:
    user_model: "default"
    api_key: "${HONCHO_API_KEY}"
```

## 上下文引擎插件

### ContextEngine 抽象基类

```python
# agent/context_engine.py
class ContextEngine(ABC):
    @abstractmethod
    async def get_relevant_context(
        self,
        query: str,
        limit: int = 10
    ) -> list[dict]:
        """获取相关上下文"""
```

### 发现

```python
# agent/context_engine.py
def discover_context_engines() -> list[type[ContextEngine]]:
    """发现可用的上下文引擎"""
```

## 图像生成插件

### ImageGenProvider 抽象基类

```python
# agent/image_gen_provider.py
class ImageGenProvider(ABC):
    @abstractmethod
    async def generate(
        self,
        prompt: str,
        **kwargs
    ) -> bytes:
        """生成图像"""

    @abstractmethod
    def get_provider_name(self) -> str:
        """获取提供者名称"""
```

### 发现

```python
# agent/image_gen_registry.py
class ImageGenRegistry:
    @classmethod
    def get_provider(cls, name: str) -> ImageGenProvider: ...
    @classmethod
    def list_providers(cls) -> list[str]: ...
```

## CLI 插件

### 注册 CLI 子命令

```python
# plugins/my_plugin/cli.py
def register_cli(subparser: ArgumentParser):
    """注册 CLI 子命令"""
    subparser.add_parser("mycommand")
```

### 发现

```python
# hermes_cli/plugins_cmd.py
def discover_plugin_cli_commands():
    """发现所有插件的 CLI 命令"""
    # 仅对当前活动的内存提供者生效
```

## 插件开发

### 创建插件

1. 创建插件目录
2. 实现 `register(ctx)` 函数
3. 添加元数据（可选）
4. 测试插件
5. 发布（可选）

### 插件模板

```python
# plugins/my_plugin/__init__.py
"""
My Plugin - 插件描述
"""

def register(ctx: PluginContext):
    """注册插件"""

    # 注册钩子
    ctx.register_hook("pre_tool_call", pre_hook)
    ctx.register_hook("post_tool_call", post_hook)

    # 注册工具
    ctx.register_tool(
        name="my_tool",
        schema={
            "name": "my_tool",
            "description": "我的工具",
            "parameters": {...}
        },
        handler=my_tool_handler
    )

async def pre_hook(tool_name: str, args: dict, task_id: str):
    """工具调用前钩子"""
    pass

async def post_hook(tool_name: str, result: str, task_id: str):
    """工具调用后钩子"""
    pass
```

## 插件市场

### 安装第三方插件

```bash
hermes plugins install /path/to/plugin
hermes plugins install https://github.com/user/plugin
```

### 发布插件

```bash
hermes plugins publish
```
