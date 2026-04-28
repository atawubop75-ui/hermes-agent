# 工具系统

## 概述

Hermes Agent 的工具系统是一个模块化、可扩展的工具注册和执行框架，支持 40+ 内置工具。

## 架构

```
tools/registry.py  ←  中心注册表（无依赖）
        ↑
tools/*.py  ←  每个工具文件调用 registry.register()
        ↑
model_tools.py  ←  导入注册表并触发工具发现
        ↑
run_agent.py, cli.py, batch_runner.py
```

## 工具注册

### 注册流程

每个工具文件在模块级别调用 `registry.register()`：

```python
# tools/example_tool.py
import json
from tools.registry import registry

def check_requirements() -> bool:
    return bool(os.getenv("EXAMPLE_API_KEY"))

def example_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"success": True, "data": "..."})

registry.register(
    name="example_tool",
    toolset="example",
    schema={...},
    handler=lambda args, **kw: example_tool(...),
    check_fn=check_requirements,
    requires_env=["EXAMPLE_API_KEY"],
)
```

### 注册参数

| 参数 | 类型 | 说明 |
|------|------|------|
| name | str | 工具名称（唯一） |
| toolset | str | 工具集名称 |
| schema | dict | OpenAI 格式的 schema |
| handler | Callable | 处理函数 |
| check_fn | Callable | 需求检查函数 |
| requires_env | list | 所需环境变量 |
| is_async | bool | 是否异步工具 |
| description | str | 工具描述 |
| emoji | str | 表情符号 |
| max_result_size_chars | int | 最大结果大小 |

## 工具集

### 内置工具集

| 工具集 | 工具 |
|--------|------|
| core | todo, interrupt, clarify |
| file | file_read, file_write, file_search, ... |
| web | web_search, web_fetch, ... |
| terminal | bash, ssh, docker_run, ... |
| browser | browser_navigate, browser_click, ... |
| code | code_execution, code_search, ... |
| skills | skills_list, skills_install, ... |
| memory | memory_search, memory_write, ... |
| delegate | delegate_to_agent, ... |
| cron | cron_schedule, cron_list, ... |
| messaging | send_message, send_email, ... |

### 工具集配置

```yaml
tools:
  enabled: ["web", "terminal", "file"]
  disabled: ["browser"]
```

## 工具发现

### 自动发现

`discover_builtin_tools()` 扫描 `tools/` 目录：

```python
def discover_builtin_tools(tools_dir=None) -> List[str]:
    tools_path = Path(__file__).resolve().parent
    module_names = [
        f"tools.{path.stem}"
        for path in sorted(tools_path.glob("*.py"))
        if path.name not in {"__init__.py", "registry.py"}
        and _module_registers_tools(path)
    ]
```

### 手动发现

```python
from model_tools import get_tool_definitions, check_toolset_requirements

# 获取所有工具定义
tools = get_tool_definitions()

# 检查工具集需求
available, missing = check_toolset_requirements("web")
```

## 工具执行

### handle_function_call

```python
from model_tools import handle_function_call

result = handle_function_call(
    tool_name="example_tool",
    arguments={"param": "value"},
    task_id="optional_task_id"
)
```

### 执行流程

1. 根据名称查找工具
2. 检查需求（`check_fn`）
3. 调用处理器
4. 存储结果（可选）
5. 返回 JSON 字符串

## 主要工具

### 文件工具 (file_tools.py)

| 工具 | 说明 |
|------|------|
| file_read | 读取文件 |
| file_write | 写入文件 |
| file_search | 搜索文件内容 |
| directory_tree | 显示目录树 |
| glob | 文件名模式匹配 |

### 代码执行 (code_execution_tool.py)

| 工具 | 说明 |
|------|------|
| python_repl | 执行 Python 代码 |
| bash | 执行 Bash 命令 |

### Web 工具 (web_tools.py)

| 工具 | 说明 |
|------|------|
| web_search | 搜索网页 |
| web_fetch | 获取页面内容 |

### 浏览器工具 (browser_tool.py)

| 工具 | 说明 |
|------|------|
| browser_navigate | 导航到 URL |
| browser_click | 点击元素 |
| browser_type | 输入文本 |
| browser_screenshot | 截图 |

### 终端工具 (terminal_tool.py)

| 工具 | 说明 |
|------|------|
| terminal_run | 运行命令 |
| terminal_kill | 终止进程 |
| terminal_attach | 附加到终端 |

### 委托工具 (delegate_tool.py)

| 工具 | 说明 |
|------|------|
| delegate_to_agent | 委托给子 Agent |
| cancel_task | 取消任务 |

## 扩展工具

### MCP 工具

通过 Model Context Protocol 扩展工具：

```python
# 配置
mcp_servers:
  - name: "example"
    command: ["npx", "mcp-server-example"]
```

### 自定义工具

参考"添加新工具"指南创建自定义工具。
