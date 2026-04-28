# Agent 模块 (agent/)

## 概述

`agent/` 目录包含 AIAgent 的核心内部组件，提供模型适配、记忆管理、上下文压缩、凭证管理等功能。

## 目录结构

```
agent/
├── __init__.py
├── memory_manager.py       # 记忆上下文管理
├── memory_provider.py     # 记忆提供者抽象基类
├── context_compressor.py   # 上下文压缩
├── prompt_builder.py      # 系统提示构建
├── prompt_caching.py      # 提示缓存
├── model_metadata.py      # 模型元数据
├── usage_pricing.py      # 用量计费
├── account_usage.py      # 账户使用统计
├── credential_pool.py    # 凭证池
├── error_classifier.py   # 错误分类
├── display.py            # UI 显示工具
├── retry_utils.py        # 重试工具
├── title_generator.py    # 标题生成器
├── subdirectory_hints.py # 子目录提示
├── context_references.py # 上下文引用
├── trajectory.py         # 轨迹保存
├── skill_commands.py     # 技能命令
├── skill_preprocessing.py # 技能预处理
├── skill_utils.py        # 技能工具
├── redac.py              # 内容编辑
├── transports/            # 传输层
├── provider adapters/     # 模型提供者适配器
└── ...
```

## 核心组件

### memory_manager.py

记忆上下文管理和清理：

```python
class StreamingContextScrubber:
    """流式上下文清理"""

    async def scrub(self, messages: list) -> list:
        """清理消息中的敏感信息"""

def build_memory_context_block(
    query: str,
    limit: int = 10
) -> dict:
    """构建记忆上下文块"""

def sanitize_context(messages: list) -> list:
    """清理上下文中的敏感信息"""
```

### context_compressor.py

上下文压缩：

```python
class ContextCompressor:
    def __init__(self, llm_client):
        self.llm = llm_client

    async def compress(
        self,
        messages: list,
        target_tokens: int
    ) -> list:
        """压缩消息到目标 token 数"""

    async def summarize_messages(
        self,
        messages: list
    ) -> str:
        """总结消息"""
```

### prompt_builder.py

系统提示构建：

```python
# 常量
DEFAULT_AGENT_IDENTITY = "..."
PLATFORM_HINTS = {...}
MEMORY_GUIDANCE = "..."
SKILLS_GUIDANCE = "..."
HERMES_AGENT_HELP_GUIDANCE = "..."

def build_skills_system_prompt(enabled_skills: list) -> str: ...
def build_context_files_prompt(context_files: list) -> str: ...
def build_environment_hints() -> str: ...
def load_soul_md(path: Path) -> str: ...
```

### credential_pool.py

凭证池管理：

```python
class CredentialPool:
    """多 API 密钥轮询"""

    def __init__(self, provider: str, credentials: list):
        self.provider = provider
        self.credentials = credentials

    async def get_credential(self) -> dict: ...
    def release_credential(self, cred: dict): ...
```

### model_metadata.py

模型元数据缓存：

```python
def fetch_model_metadata(
    provider: str,
    model: str,
    api_key: str
) -> dict: ...

def estimate_tokens_rough(text: str, model: str) -> int: ...
def estimate_messages_tokens_rough(
    messages: list,
    model: str
) -> int: ...

def query_ollama_num_ctx(base_url: str) -> int: ...
```

### error_classifier.py

API 错误分类：

```python
class FailoverReason(Enum):
    RATE_LIMIT = "rate_limit"
    TIMEOUT = "timeout"
    AUTH_ERROR = "auth_error"
    SERVER_ERROR = "server_error"
    CONTEXT_OVERFLOW = "context_overflow"
    UNKNOWN = "unknown"

def classify_api_error(error: Exception) -> FailoverReason: ...
```

### display.py

UI 显示工具：

```python
class KawaiiSpinner:
    """可爱风格的旋转动画"""

    def __init__(
        self,
        frames: tuple = _DEFAULT_FRAMES,
        prefix: str = ""
    ): ...

    def __enter__(self): ...
    def __exit__(self, *args): ...
    def set(self, message: str): ...

def get_tool_emoji(tool_name: str) -> str: ...
def get_cute_tool_message(tool_name: str, message: str) -> str: ...
```

### retry_utils.py

重试工具：

```python
def jittered_backoff(
    base: float = 1.0,
    max_delay: float = 60.0,
    jitter: float = 0.1
) -> Generator: ...
```

### usage_pricing.py

用量和计费：

```python
def estimate_usage_cost(
    model: str,
    input_tokens: int,
    output_tokens: int
) -> float: ...

def normalize_usage(usage: dict) -> CanonicalUsage: ...

@dataclass
class CanonicalUsage:
    model: str
    input_tokens: int
    output_tokens: int
    cost: float
```

## 提供者适配器

| 文件 | 支持者 |
|------|--------|
| `anthropic_adapter.py` | Anthropic (Claude) |
| `auxiliary_client.py` | Nous Portal, OpenRouter, Moonshot, MiniMax, xAI 等 |
| `bedrock_adapter.py` | AWS Bedrock |
| `gemini_native_adapter.py` | Google Gemini Native |
| `gemini_cloudcode_adapter.py` | Google Gemini CloudCode |
| `google_code_assist.py` | Google Code Assist |
| `google_oauth.py` | Google OAuth |
| `moonshot_schema.py` | Moonshot/Kimi |
| `codex_responses_adapter.py` | OpenAI Codex Responses |
| `nous_rate_guard.py` | Nous Rate Guard |

## 传输层 (transports/)

```python
# transports/http.py
class HTTPTransport:
    async def request(
        self,
        method: str,
        url: str,
        **kwargs
    ) -> Response: ...

# transports/websocket.py
class WebSocketTransport:
    async def connect(self, url: str): ...
    async def send(self, data: dict): ...
    async def receive(self) -> dict: ...
```

## 扩展点

### 钩子

```python
# pre_tool_call
async def my_pre_tool_hook(tool_name: str, args: dict, task_id: str):
    pass

# post_tool_call
async def my_post_tool_hook(tool_name: str, result: str, task_id: str):
    pass

# pre_llm_call
async def my_pre_llm_hook(messages: list, model: str):
    pass

# post_llm_call
async def my_post_llm_hook(response: dict, model: str):
    pass
```

### Context Engine 插件

```python
# agent/context_engine.py
class ContextEngine(ABC):
    @abstractmethod
    async def get_relevant_context(
        self,
        query: str,
        limit: int = 10
    ) -> list: ...
```
