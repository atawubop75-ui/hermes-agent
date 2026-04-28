# 记忆系统

## 概述

Hermes Agent 的记忆系统提供持久化记忆存储和跨会话上下文检索能力。

## 架构

```
Agent Session
    ↓
MemoryManager
    ↓
┌─────────────────┐
│ MemoryProvider  │ ← 插件式接口
├─────────────────┤
│ Honcho          │
│ Mem0            │
│ Supermemory     │
│ Hindsight       │
│ ...             │
└─────────────────┘
```

## MemoryProvider 接口

```python
from agent.memory_provider import MemoryProvider

class MyMemoryProvider(MemoryProvider):
    async def setup(self, config: dict): ...

    async def sync_turn(self, turn_messages: list):
        """同步对话轮次到记忆"""

    async def prefetch(self, query: str):
        """预取相关记忆"""

    async def shutdown(self): ...

    async def post_setup(self, hermes_home: Path, config: dict):
        """设置后回调"""
```

## 内置提供者

### Honcho

dialectic user modeling 提供者：

```yaml
memory:
  provider: "honcho"
  honcho:
    user_model: "default"
```

### Mem0

Mem0 记忆服务：

```yaml
memory:
  provider: "mem0"
  mem0:
    api_key: "${MEM0_API_KEY}"
```

### Supermemory

Supermemory 记忆系统：

```yaml
memory:
  provider: "supermemory"
  supermemory:
    api_key: "${SUPERMEMORY_API_KEY}"
```

## 记忆操作

### 写入记忆

```python
from tools.memory_tool import memory_write

result = memory_write(
    content="用户喜欢用中文交流",
    category="user_preference"
)
```

### 搜索记忆

```python
from tools.memory_tool import memory_search

result = memory_search(
    query="用户的语言偏好是什么"
)
```

### 同步对话

每轮对话结束后自动同步到记忆系统：

```python
# agent/memory_manager.py
class MemoryManager:
    async def sync_turn(self, turn_messages: list):
        """同步对话到记忆提供者"""
```

## 上下文构建

### build_memory_context_block

```python
from agent.memory_manager import build_memory_context_block

context = build_memory_context_block(
    query="当前对话的相关记忆",
    limit=5
)
```

### sanitize_context

清理上下文中的敏感信息：

```python
from agent.memory_manager import sanitize_context

cleaned = sanitize_context(messages)
```

## 会话搜索

### SessionDB FTS5

`hermes_state.py` 使用 SQLite FTS5 提供全文搜索：

```python
class SessionDB:
    def search_sessions(self, query: str, limit: int = 10) -> list:
        """搜索会话"""
```

## 配置

### 启用记忆

```yaml
memory:
  provider: "honcho"
  enabled: true
```

### 跳过记忆

```python
agent = AIAgent(
    skip_memory=True,  # 跳过记忆加载
    # ...
)
```

## 用户模型

### Honcho Dialectic

Honcho 提供基于 dialectic 方法的用户建模：

- 学习用户偏好
- 跟踪用户目标
- 建立用户画像

### 跨会话持久化

```python
# 用户画像自动跨会话持久化
memory = await honcho.get_user_model(user_id)
```
