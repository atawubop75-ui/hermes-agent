# CLI 模块 (hermes_cli/)

## 概述

`hermes_cli/` 目录包含 Hermes Agent 的命令行界面实现，提供交互式 REPL、命令处理、配置管理等功能。

## 目录结构

```
hermes_cli/
├── __init__.py
├── main.py              # 主入口 (~8k LOC)
├── commands.py          # 命令注册表 (~1.5k LOC)
├── config.py            # 配置管理 (~3k LOC)
├── banner.py            # 横幅显示
├── display.py           # 显示工具
├── skin_engine.py       # 皮肤引擎 (~1k LOC)
├── auth.py              # 认证管理 (~4k LOC)
├── auth_commands.py     # 认证命令
├── setup.py             # 设置向导 (~3k LOC)
├── gateway.py           # 网关命令 (~4k LOC)
├── doctor.py            # 诊断工具 (~1.5k LOC)
├── models.py            # 模型管理 (~2.5k LOC)
├── model_switch.py      # 模型切换
├── model_catalog.py     # 模型目录
├── model_normalize.py   # 模型名称规范化
├── providers.py         # 提供者管理
├── runtime_provider.py  # 运行时提供者
├── tools_config.py      # 工具配置 (~2k LOC)
├── profiles.py          # 配置文件管理 (~1k LOC)
├── plugins_cmd.py       # 插件命令 (~1k LOC)
├── plugins.py           # 插件管理 (~1k LOC)
├── skills_hub.py        # 技能市场 (~1.5k LOC)
├── skills_config.py     # 技能配置
├── memory_setup.py      # 记忆设置
├── cron.py              # Cron 命令
├── logs.py              # 日志命令
├── status.py            # 状态命令
├── clipboard.py         # 剪贴板
├── completion.py        # 自动补全
├── hook.py              # 钩子管理
├── webhook.py           # Webhook
├── voice.py             # 语音命令
├── dingtalk_auth.py     # 钉钉认证
├── copilot_auth.py      # Copilot 认证
├── azure_detect.py      # Azure 检测
├── codex_models.py      # Codex 模型
├── fallback_cmd.py      # 后备命令
├── tips.py              # 提示
├── uninstall.py         # 卸载
├── oneshot.py           # 单次执行
├── pairing.py           # 配对
├── platforms.py         # 平台
├── timeouts.py          # 超时配置
├── env_loader.py        # 环境加载
├── default_soul.py      # 默认人格
├── claw.py              # OpenClaw 迁移
├── backup.py            # 备份
├── pty_bridge.py        # PTY 桥接
├── curses_ui.py         # Curses UI
├── web_server.py        # Web 服务器 (~2.5k LOC)
└── ...
```

## 主入口 (main.py)

```python
def main():
    """Hermes CLI 主入口"""
    # 解析命令行参数
    # 初始化配置
    # 启动 CLI 或 Gateway
```

### 命令行参数

```bash
hermes [options] [command] [args]

Options:
  --model MODEL          # 指定模型
  --provider PROVIDER    # 指定提供者
  --toolsets TOOLSETS   # 指定工具集
  --quiet, -q           # 静默模式
  --session SESSION      # 会话 ID

Commands:
  help                   # 显示帮助
  model                  # 模型管理
  tools                  # 工具管理
  config                 # 配置管理
  gateway                # 网关管理
  skills                 # 技能管理
  plugins                # 插件管理
```

## 命令注册表 (commands.py)

### CommandDef 结构

```python
@dataclass
class CommandDef:
    name: str                    # 命令名称
    description: str             # 描述
    category: str                 # 分类
    aliases: tuple = ()          # 别名
    args_hint: str = None         # 参数提示
    cli_only: bool = False        # 仅 CLI
    gateway_only: bool = False    # 仅 Gateway
    gateway_config_gate: str = None  # Gateway 配置门控
```

### 类别

| 类别 | 说明 |
|------|------|
| Session | 会话命令 |
| Configuration | 配置命令 |
| Tools & Skills | 工具和技能命令 |
| Info | 信息命令 |
| Exit | 退出命令 |

### 注册命令

```python
# 新增命令
CommandDef(
    "mycommand",
    "Description",
    "Session",
    aliases=("mc",),
    args_hint="[arg]"
)
```

## 配置管理 (config.py)

### DEFAULT_CONFIG

默认配置结构：

```python
DEFAULT_CONFIG = {
    "_config_version": 21,
    "model": "claude-sonnet-4-20250514",
    "provider": "anthropic",
    "display": {
        "skin": "default",
        "color": True,
        "spinner": True,
    },
    "tools": {
        "enabled": ["core", "file", "web", "terminal"],
        "disabled": [],
    },
    "memory": {
        "provider": "honcho",
    },
    "platforms": {...},
    "terminal": {...},
}
```

### 配置加载

```python
def load_config() -> dict:
    """加载用户配置"""

def load_cli_config() -> dict:
    """加载 CLI 配置"""

def save_config_value(key: str, value: any):
    """保存配置值"""
```

### 环境变量

```python
OPTIONAL_ENV_VARS = {
    "ANTHROPIC_API_KEY": {...},
    "OPENAI_API_KEY": {...},
    # ...
}
```

## 皮肤引擎 (skin_engine.py)

### SkinConfig

```python
@dataclass
class SkinConfig:
    name: str
    description: str
    colors: dict
    spinner: dict
    branding: dict
    tool_prefix: str
    tool_emojis: dict
```

### 内置皮肤

| 皮肤 | 说明 |
|------|------|
| default | 经典金色/可爱风格 |
| ares | 红色/青铜战争主题 |
| mono | 灰度单色 |
| slate | 冷蓝色开发者主题 |

### 用户皮肤

```yaml
# ~/.hermes/skins/myskin.yaml
name: myskin
description: My custom skin
colors:
  banner_border: "#FF00FF"
  banner_title: "#00FFFF"
```

## 设置向导 (setup.py)

```python
class SetupWizard:
    """交互式设置向导"""

    def run(self):
        """运行设置流程"""

    def configure_api_keys(self): ...
    def configure_model(self): ...
    def configure_memory(self): ...
    def configure_gateway(self): ...
```

## 诊断工具 (doctor.py)

```python
class Doctor:
    """诊断工具"""

    def check_installation(self): ...
    def check_configuration(self): ...
    def check_dependencies(self): ...
    def check_network(self): ...
    def check_api_keys(self): ...
```

## Web 服务器 (web_server.py)

### Dashboard

```python
# FastAPI 应用
app = FastAPI()

@app.get("/")
async def index(): ...

@app.websocket("/api/pty")
async def pty_websocket(token: str, websocket: WebSocket): ...
```

### 端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/` | GET | Dashboard 首页 |
| `/api/pty` | WS | PTY WebSocket |
| `/api/sessions` | GET | 会话列表 |
| `/api/sessions/{id}` | GET | 会话详情 |
| `/chat` | GET | 聊天页面 |

## 工具配置 (tools_config.py)

```python
class ToolsConfig:
    """工具配置管理"""

    def list_tools(self) -> list: ...
    def enable_tool(self, name: str): ...
    def disable_tool(self, name: str): ...
    def get_tool_config(self, name: str) -> dict: ...
```

## 插件管理 (plugins.py)

```python
class PluginManager:
    """插件管理器"""

    def discover_plugins(self): ...
    def load_plugin(self, name: str): ...
    def unload_plugin(self, name: str): ...
    def get_plugin_info(self, name: str) -> dict: ...
```

## 配置文件管理 (profiles.py)

```python
class ProfileManager:
    """多配置文件管理"""

    def list_profiles(self) -> list: ...
    def create_profile(self, name: str): ...
    def switch_profile(self, name: str): ...
    def delete_profile(self, name: str): ...
```
