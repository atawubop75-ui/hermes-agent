# Hermes Agent 系统架构

## 1. 架构概述

Hermes Agent 采用模块化架构，主要由以下几个核心组件构成：

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLI / TUI                               │
│                   (cli.py / ui-tui)                            │
├─────────────────────────────────────────────────────────────────┤
│                     Gateway Runner                               │
│                    (gateway/run.py)                             │
├───────────────────┬─────────────────────────────────────────────┤
│   AIAgent         │           消息平台适配器                      │
│  (run_agent.py)   │    (telegram.py, discord.py, slack.py...)  │
├───────────────────┴─────────────────────────────────────────────┤
│                      工具系统                                    │
│              (tools/registry.py + tools/*.py)                   │
├─────────────────────────────────────────────────────────────────┤
│     Agent 内部组件 (agent/)                                       │
│  provider adapters, memory, compression, credential pool...       │
├─────────────────────────────────────────────────────────────────┤
│     插件系统 (plugins/)                                           │
│  memory providers, context engines, image-gen...                 │
└─────────────────────────────────────────────────────────────────┘
```

## 2. 核心组件

### 2.1 AIAgent (run_agent.py)

**核心对话循环**，处理与 LLM 的交互和工具调用。

```
用户消息 → 构建消息 → LLM API 调用 → 工具调用循环 → 返回结果
```

**主要职责**：
- 管理对话上下文和消息历史
- 处理工具调用循环直到完成
- 管理迭代预算和中断请求
- 协调记忆系统和技能系统
- 处理错误分类和重试逻辑

**关键类**：`AIAgent`
- `chat(message: str) -> str`：简单接口
- `run_conversation(...) -> dict`：完整接口

### 2.2 CLI (cli.py)

**交互式命令行界面**，使用 prompt_toolkit 构建。

**主要职责**：
- 交互式 REPL 界面
- 命令处理和自动补全
- 富文本格式化和动画效果
- Slash 命令注册和分发

### 2.3 Gateway (gateway/run.py)

**消息网关**，连接多个即时通讯平台。

**主要职责**：
- 管理多个平台适配器生命周期
- 消息路由和会话管理
- 跨平台命令处理
- WebSocket 和 Webhook 处理

### 2.4 工具系统 (tools/)

**40+ 内置工具**，通过注册表自动发现。

#### 工具注册流程

```
tools/registry.py  (无依赖)
        ↑
tools/*.py  (每个文件调用 registry.register())
        ↑
model_tools.py  (导入注册表 + 触发工具发现)
        ↑
run_agent.py, cli.py, batch_runner.py
```

#### 主要工具分类

| 类别 | 工具 | 说明 |
|------|------|------|
| 文件操作 | `file_tools.py`, `file_operations.py` | 文件读写、搜索、移动 |
| 代码执行 | `code_execution_tool.py` | Python/Bash 代码执行 |
| 浏览器自动化 | `browser_tool.py`, `browser_supervisor.py` | Web 浏览和交互 |
| Web 搜索 | `web_tools.py` | Exa 搜索、Firecrawl 爬取 |
| 终端 | `terminal_tool.py` | 终端后端管理 |
| 技能 | `skills_tool.py`, `skill_manager_tool.py` | 技能管理 |
| 记忆 | `memory_tool.py` | 持久化记忆 |
| 委托 | `delegate_tool.py` | 子 Agent 委托 |
| 消息 | `send_message_tool.py` | 跨平台消息发送 |
| 定时任务 | `cronjob_tools.py` | Cron 调度 |
| MCP | `mcp_tool.py` | Model Context Protocol |
| 语音 | `tts_tool.py`, `transcription_tools.py` | TTS/STT |
| 图像生成 | `image_generation_tool.py` | AI 图像生成 |
| Discord | `discord_tool.py` | Discord 特定功能 |
| 飞书 | `feishu_doc_tool.py`, `feishu_drive_tool.py` | 飞书文档/云盘 |
| 元宝 | `yuanbao_tools.py` | 腾讯元宝集成 |
| 记忆家庭 | `homeassistant_tool.py` | Home Assistant 集成 |

### 2.5 Agent 内部组件 (agent/)

| 文件 | 职责 |
|------|------|
| `memory_manager.py` | 记忆上下文构建和清理 |
| `context_compressor.py` | 上下文压缩 |
| `prompt_builder.py` | 系统提示构建 |
| `credential_pool.py` | 凭证池管理 |
| `model_metadata.py` | 模型元数据缓存 |
| `usage_pricing.py` | 用量计费估算 |
| `display.py` | UI 显示工具 |
| `error_classifier.py` | 错误分类和恢复 |

### 2.6 提供者适配器 (agent/)

支持多种 LLM 提供者：

| 适配器 | 文件 | 支持模型 |
|--------|------|----------|
| OpenAI | `anthropic_adapter.py` | GPT-4, GPT-3.5 |
| Anthropic | `anthropic_adapter.py` | Claude 3.5, Claude 3 |
| Google | `gemini_native_adapter.py`, `gemini_cloudcode_adapter.py` | Gemini Pro, Gemini Ultra |
| Bedrock | `bedrock_adapter.py` | AWS Bedrock 模型 |
| Nous Portal | `auxiliary_client.py` | Nous 自有模型 |
| OpenRouter | `auxiliary_client.py` | 200+ 模型 |
| Moonshot | `auxiliary_client.py` | Kimi 模型 |
| MiniMax | `auxiliary_client.py` | MiniMax 模型 |
| xAI | `auxiliary_client.py` | Grok 模型 |
| Codex | `codex_responses_adapter.py` | OpenAI Codex |

## 3. 数据流

### 3.1 消息处理流程

```
用户输入 → CLI/Gateway → AIAgent.run_conversation()
                                    ↓
                            构建消息列表
                                    ↓
                            LLM API 调用
                                    ↓
                    ┌───────────────┴───────────────┐
                    ↓                               ↓
              工具调用                          返回结果
                    ↓
            handle_function_call()
                    ↓
            ┌───────┴───────┐
            ↓               ↓
      工具执行           委托子 Agent
            ↓               ↓
        返回结果 → 添加到消息 → 继续循环
```

### 3.2 会话管理

```
SessionDB (hermes_state.py)
    ├── SQLite + FTS5 全文搜索
    ├── 会话历史存储
    ├── 消息压缩状态
    └── 跨会话记忆
```

### 3.3 技能加载

```
skills/                    # 内置技能 (默认加载)
    ├── github/
    ├── software-development/
    └── ...
optional-skills/           # 可选技能 (按需安装)
    ├── mlops/
    ├── devops/
    └── ...
~/.hermes/skills/          # 用户技能
```

## 4. 终端后端

`tools/environments/` 支持多种执行环境：

| 后端 | 说明 |
|------|------|
| local | 本地终端 |
| docker | Docker 容器 |
| ssh | 远程 SSH |
| daytona | Daytona 云端环境 |
| singularity | Singularity 容器 |
| modal | Modal 无服务器 |

## 5. 消息平台

`gateway/platforms/` 支持：

| 平台 | 文件大小 | 特点 |
|------|----------|------|
| Telegram | ~138KB | Bot API, Webhook |
| Discord | ~172KB | Bot API, Slash Commands |
| Slack | ~102KB | Bolt SDK |
| WhatsApp | ~45KB | WhatsApp Business API |
| Signal | ~39KB | Signal Messenger |
| Matrix | ~106KB | Matrix protocol |
| Mattermost | ~28KB | Mattermost webhook |
| Email | ~23KB | SMTP/IMAP |
| SMS | ~14KB | Twilio 等 |
| 飞书 | ~192KB | 文档、评论、回调 |
| 钉钉 | ~56KB | 钉钉群消息 |
| 企业微信 | ~77KB | 企业微信应用 |
| 微信 | ~165KB | 微信消息 |
| Home Assistant | ~16KB | Home Assistant 集成 |
| QQBot | ~? | QQ 机器人 |
| BlueBubbles | ~34KB | iMessage 桥接 |
| Webhook | ~31KB | 通用 Webhook |
| API Server | ~118KB | REST API |

## 6. 插件系统

### 6.1 通用插件 (plugins/)

通过 `PluginManager` 发现和管理：

```python
class PluginManager:
    def register(self, ctx):
        # 注册生命周期钩子
        # pre_tool_call, post_tool_call
        # pre_llm_call, post_llm_call
        # on_session_start, on_session_end
        # 注册工具
        # 注册 CLI 命令
```

### 6.2 记忆提供者插件 (plugins/memory/)

实现 `MemoryProvider` ABC：

| 提供者 | 说明 |
|--------|------|
| honcho | Honcho AI dialectic user modeling |
| mem0 | Mem0 记忆系统 |
| supermemory | Supermemory 记忆系统 |
| byterover | Byterover 记忆 |
| hindsight | Hindsight 记忆 |
| holographic | Holographic 记忆 |
| openviking | OpenViking 记忆 |
| retaindb | RetainDB 记忆 |

### 6.3 其他插件

- `context_engine/` - 上下文引擎插件
- `image_gen/` - 图像生成插件
- `dashboard/` - 仪表板插件

## 7. 配置系统

### 7.1 配置文件

- `~/.hermes/config.yaml` - 用户配置
- `~/.hermes/.env` - API 密钥（仅密钥）

### 7.2 配置加载

| 加载器 | 使用场景 | 位置 |
|--------|----------|------|
| `load_cli_config()` | CLI 模式 | cli.py |
| `load_config()` | 工具、设置、大多数子命令 | hermes_cli/config.py |
| 直接 YAML 加载 | Gateway 运行时 | gateway/run.py + gateway/config.py |

### 7.3 多配置文件支持

通过 `hermes_constants.py` 的 `get_hermes_home()` 支持：

```
~/.hermes/                      # 默认配置
~/.hermes/profiles/<name>/      # 命名配置文件
```

## 8. 关键技术特性

### 8.1 提示缓存

- Anthropic cache control 支持
- 自动应用 `CacheControl` 头部
- 上下文压缩时保留缓存标记

### 8.2 上下文压缩

- `ContextCompressor` 类处理
- LLM 辅助摘要
- 选择性保留关键信息

### 8.3 预算控制

- 迭代预算 (`iteration_budget`)
- Token 预算
- 工具结果大小限制

### 8.4 错误处理

- `ErrorClassifier` 分类 API 错误
- `FailoverReason` 枚举
- 自动重试 (`jittered_backoff`)

### 8.5 安全特性

- 路径安全检查 (`path_security.py`)
- URL 安全检查 (`url_safety.py`)
- 技能守卫 (`skills_guard.py`)
- TIRITH 安全 (`tirith_security.py`)

## 9. 依赖关系图

```
run_agent.py
├── model_tools.py
│   └── tools/registry.py
├── tools/terminal_tool.py
├── tools/interrupt.py
├── tools/browser_tool.py
├── agent/memory_manager.py
├── agent/error_classifier.py
├── agent/prompt_builder.py
├── agent/context_compressor.py
└── agent/display.py

cli.py
├── hermes_cli/commands.py
├── hermes_cli/skin_engine.py
├── hermes_cli/config.py
└── agent/usage_pricing.py

gateway/run.py
├── gateway/session.py
├── gateway/platforms/*.py
├── hermes_cli/config.py
└── agent/account_usage.py
```

## 10. 文件大小统计

| 文件 | 大小 | LOC (估算) |
|------|------|-----------|
| run_agent.py | 677KB | ~13,000 |
| cli.py | 515KB | ~11,000 |
| hermes_cli/main.py | 381KB | ~8,000 |
| toolsets.py | 25KB | - |
| model_tools.py | 28KB | - |
| hermes_state.py | 82KB | - |
| gateway/run.py | 118KB | ~2,500 |
| gateway/base.py | 118KB | ~2,500 |
| gateway/telegram.py | 138KB | ~3,000 |
| gateway/discord.py | 172KB | ~3,500 |
