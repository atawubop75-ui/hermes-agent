# Hermes Agent 开发者指南

## 1. 开发环境设置

### 1.1 快速开始

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
./setup-hermes.sh     # 自动安装依赖并创建虚拟环境
./hermes              # 启动 CLI
```

### 1.2 手动设置

```bash
# 安装 uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 创建虚拟环境
uv venv venv --python 3.11
source venv/bin/activate

# 安装依赖
uv pip install -e ".[all,dev]"

# 运行测试
./scripts/run_tests.sh
```

### 1.3 可选依赖

```bash
# RL/Atropos 集成 (可选)
uv pip install -e ".[rl]"

# 开发依赖
uv pip install -e ".[dev]"
```

## 2. 项目结构

```
hermes-agent/
├── run_agent.py          # AIAgent 核心类 (~13k LOC)
├── model_tools.py        # 工具编排 (~1k LOC)
├── toolsets.py           # 工具集定义
├── cli.py                # CLI 界面 (~11k LOC)
├── batch_runner.py       # 并行批处理
├── hermes_state.py       # SQLite 会话存储
├── hermes_constants.py   # 路径管理
├── hermes_logging.py     # 日志系统
├── agent/                # Agent 内部组件
│   ├── memory_manager.py
│   ├── context_compressor.py
│   ├── prompt_builder.py
│   ├── provider adapters/
│   └── ...
├── hermes_cli/           # CLI 子命令
│   ├── main.py           # 主入口 (~8k LOC)
│   ├── commands.py       # 命令注册表
│   ├── config.py         # 配置管理
│   ├── skin_engine.py    # 皮肤引擎
│   └── ...
├── tools/                # 工具实现
│   ├── registry.py       # 工具注册表
│   ├── environments/     # 终端后端
│   └── *.py              # 各工具
├── gateway/              # 消息网关
│   ├── run.py            # 网关入口
│   ├── session.py        # 会话管理
│   ├── platforms/        # 平台适配器
│   └── builtin_hooks/    # 内置钩子
├── plugins/              # 插件系统
│   ├── memory/           # 记忆提供者
│   ├── context_engine/   # 上下文引擎
│   └── ...
├── skills/               # 内置技能
├── optional-skills/      # 可选技能
├── ui-tui/               # React TUI
├── tui_gateway/          # TUI 后端
└── tests/                # 测试套件
```

## 3. 核心开发指南

### 3.1 添加新工具

**步骤 1**: 创建工具文件 `tools/my_tool.py`

```python
import json
from tools.registry import registry

def check_requirements() -> bool:
    import os
    return bool(os.getenv("MY_API_KEY"))

def my_tool(param: str, task_id: str = None) -> str:
    result = {"success": True, "data": param.upper()}
    return json.dumps(result)

registry.register(
    name="my_tool",
    toolset="my_toolset",
    schema={
        "name": "my_tool",
        "description": "我的工具描述",
        "parameters": {
            "type": "object",
            "properties": {
                "param": {"type": "string", "description": "输入参数"}
            },
            "required": ["param"]
        }
    },
    handler=lambda args, **kw: my_tool(
        param=args.get("param", ""),
        task_id=kw.get("task_id")
    ),
    check_fn=check_requirements,
    requires_env=["MY_API_KEY"],
)
```

**步骤 2**: 添加到工具集 (`toolsets.py`)

```python
# 在 _HERMES_CORE_TOOLS 或新工具集中添加
_MY_TOOLSET = ["my_tool"]

# 在 ALL_TOOLSETS 中注册
ALL_TOOLSETS = {
    "my_toolset": _MY_TOOLSET,
    # ...
}
```

### 3.2 添加 Slash 命令

**步骤 1**: 在 `hermes_cli/commands.py` 添加命令定义

```python
CommandDef(
    "mycommand",
    "Description of what it does",
    "Session",
    aliases=("mc",),
    args_hint="[arg]"
),
```

**步骤 2**: 在 `cli.py` 添加处理器

```python
def process_command(self, cmd: str):
    # ...
    elif canonical == "mycommand":
        self._handle_mycommand(cmd_original)

def _handle_mycommand(self, cmd_original: str):
    # 处理命令逻辑
```

**步骤 3**: 在 `gateway/run.py` 添加 Gateway 处理器（如需要）

```python
if canonical == "mycommand":
    return await self._handle_mycommand(event)
```

### 3.3 添加平台适配器

参考 `gateway/platforms/ADDING_A_PLATFORM.md`：

1. 创建新适配器类继承 `BasePlatform`
2. 实现必要的方法
3. 在 `gateway/run.py` 注册

### 3.4 添加插件

**通用插件** (`plugins/<name>/`):

```python
# __init__.py
def register(ctx: PluginContext):
    ctx.register_hook("pre_tool_call", my_hook)
    ctx.register_tool("my_tool", schema, handler)
```

**记忆提供者** (`plugins/memory/<name>/`):

```python
from agent.memory_provider import MemoryProvider

class MyMemoryProvider(MemoryProvider):
    async def setup(self, config): ...
    async def sync_turn(self, messages): ...
    async def prefetch(self, query): ...
    async def shutdown(self): ...
```

## 4. 代码规范

### 4.1 Python 版本

- **最低版本**: Python 3.11
- **推荐版本**: Python 3.13

### 4.2 代码风格

- 遵循 PEP 8
- 使用 type hints
- 文档字符串使用 Google/NumPy 风格

### 4.3 导入规范

```python
# 标准库
import os
import json
from typing import List, Dict

# 第三方库
from rich.console import Console

# 本地导入
from agent.memory_manager import MemoryManager
from tools.registry import registry
```

### 4.4 路径处理

```python
# 正确 - 使用 get_hermes_home()
from hermes_constants import get_hermes_home
config_path = get_hermes_home() / "config.yaml"

# 错误 - 不要硬编码路径
config_path = Path.home() / ".hermes" / "config.yaml"
```

### 4.5 日志记录

```python
import logging
logger = logging.getLogger(__name__)

logger.info("信息消息")
logger.warning("警告消息")
logger.error("错误消息", exc_info=True)
```

## 5. 测试

### 5.1 运行测试

```bash
# 完整测试套件
./scripts/run_tests.sh

# 单个目录
./scripts/run_tests.sh tests/gateway/

# 单个测试
./scripts/run_tests.sh tests/test_model_tools.py::test_x

# 详细输出
./scripts/run_tests.sh -v --tb=long
```

### 5.2 测试规范

- 使用 pytest
- 使用 `tests/conftest.py` 的 `_isolate_hermes_home` fixture
- 不要硬编码 `~/.hermes/` 路径

```python
import pytest

def test_my_feature(_isolate_hermes_home):
    # 测试代码
    pass
```

### 5.3 注意事项

- **不要写 change-detector 测试**
- 测试应该验证行为，而不是快照数据
- 使用 fixtures 管理测试依赖

## 6. 配置管理

### 6.1 添加配置选项

**config.yaml 选项** (在 `hermes_cli/config.py`):

```python
# 添加到 DEFAULT_CONFIG
DEFAULT_CONFIG = {
    # ...
    "my_section": {
        "my_setting": "default_value",
    }
}
```

**环境变量** (在 `hermes_cli/config.py`):

```python
OPTIONAL_ENV_VARS = {
    "MY_API_KEY": {
        "description": "我的 API 密钥",
        "prompt": "My API Key",
        "url": "https://example.com/api",
        "password": True,
        "category": "tool",
    }
}
```

### 6.2 配置文件位置

| 类型 | 路径 |
|------|------|
| 用户配置 | `~/.hermes/config.yaml` |
| 环境变量 | `~/.hermes/.env` |
| 日志 | `~/.hermes/logs/` |
| 会话 | `~/.hermes/sessions/` |
| 技能 | `~/.hermes/skills/` |

## 7. 构建和发布

### 7.1 版本管理

版本在 `pyproject.toml` 中定义：

```toml
[project]
name = "hermes-agent"
version = "0.11.0"
```

### 7.2 发布流程

```bash
# 构建
python -m build

# 发布到 PyPI
twine upload dist/*
```

### 7.3 安装方式

```bash
# 从 PyPI
pip install hermes-agent

# 从源码
pip install -e ".[all]"

# 使用 uv
uv pip install hermes-agent
```

## 8. 调试

### 8.1 日志查看

```bash
# 查看日志
hermes logs

# 跟随日志
hermes logs --follow

# 按级别过滤
hermes logs --level error

# 按会话过滤
hermes logs --session <session_id>
```

### 8.2 调试工具

```bash
# 运行诊断
hermes doctor

# 调试模式
hermes debug --verbose
```

### 8.3 IDE 集成

推荐使用 VS Code 或 PyCharm，配合 `debugpy`:

```bash
# 启动调试服务器
python -m debugpy --listen 5678 ./hermes
```

## 9. 性能优化

### 9.1 缓存

- 使用 `agent/prompt_caching.py` 的缓存控制
- 启用上下文压缩减少 token 使用

### 9.2 并行处理

```python
# 使用 batch_runner.py
from batch_runner import BatchRunner

runner = BatchRunner(max_workers=4)
results = runner.run_batch(tasks)
```

### 9.3 内存管理

- 定期清理过期会话
- 使用 `enforce_turn_budget` 限制工具结果大小

## 10. 安全

### 10.1 凭证管理

- API 密钥存储在 `~/.hermes/.env`
- 不要提交密钥到代码库
- 使用凭证池管理多个 API 密钥

### 10.2 路径安全

```python
from tools.path_security import is_safe_path

if not is_safe_path(user_path):
    raise PermissionError("路径不安全")
```

### 10.3 URL 安全

```python
from tools.url_safety import is_safe_url

if not is_safe_url(user_url):
    raise ValueError("URL 不安全")
```

## 11. 贡献指南

### 11.1 Pull Request 流程

1. Fork 仓库
2. 创建功能分支
3. 进行开发
4. 运行测试
5. 提交 PR

### 11.2 代码审查

- 确保测试通过
- 遵循代码规范
- 更新文档

### 11.3 报告问题

- 使用 GitHub Issues
- 提供复现步骤
- 包含相关日志
