# Hermes Agent 项目文档

本文档由 AI 自动生成，提供 Hermes Agent 项目的完整结构、架构和开发指南。

## 文档结构

### 主文档

- [ARCHITECTURE.md](./ARCHITECTURE.md) - 系统架构文档
- [INTERFACES.md](./INTERFACES.md) - 接口文档
- [DEVELOPER_GUIDE.md](./DEVELOPER_GUIDE.md) - 开发者指南

### 核心概念 (concept/)

- [工具系统](./concept/tools.md) - 工具注册、执行和扩展
- [记忆系统](./concept/memory.md) - 持久化记忆和跨会话上下文
- [技能系统](./concept/skills.md) - 技能创建、加载和管理
- [消息网关](./concept/gateway.md) - 多平台消息集成架构

### 模块文档 (modules/)

- [Agent 模块](./modules/agent.md) - AIAgent 核心组件详解
- [CLI 模块](./modules/hermes_cli.md) - 命令行界面详解
- [Gateway 模块](./modules/gateway.md) - 消息网关详解
- [TUI 模块](./modules/tui.md) - React/Ink 终端 UI 详解
- [插件系统](./modules/plugins.md) - 插件架构和开发

## 项目概览

Hermes Agent 是由 [Nous Research](https://nousresearch.com) 开发的一款自改进 AI Agent，主要特点包括：

- **自改进学习循环**：从经验中创建技能、在使用中改进技能
- **多平台支持**：CLI、Telegram、Discord、Slack、WhatsApp、Signal 等
- **40+ 内置工具**：文件操作、代码执行、浏览器自动化、Web 搜索等
- **多种终端后端**：本地、Docker、SSH、Daytona、Singularity、Modal
- **插件式记忆系统**：支持 Honcho、Mem0、Supermemory 等
- **技能系统**：内置技能和可选技能市场

## 快速导航

### 核心模块

| 模块 | 路径 | 说明 |
|------|------|------|
| AIAgent | `run_agent.py` | 核心对话循环 (~13k LOC) |
| CLI | `cli.py` | 交互式 CLI 界面 (~11k LOC) |
| Gateway | `gateway/run.py` | 消息网关，支持多平台 |
| 工具系统 | `tools/` | 40+ 工具，自动注册 |
| Agent | `agent/` | 提供者适配器、记忆、压缩等 |
| 插件系统 | `plugins/` | 通用插件和记忆提供者插件 |

### 配置与常量

| 文件 | 说明 |
|------|------|
| `hermes_constants.py` | 路径管理，支持多配置文件 |
| `hermes_state.py` | SQLite 会话存储，FTS5 搜索 |
| `hermes_logging.py` | 日志系统 |
| `hermes_cli/config.py` | CLI 配置管理 |

## 技术栈

- **语言**：Python 3.11+
- **AI 提供者**：OpenAI、Anthropic、Google Gemini、Nous Portal、OpenRouter 等
- **CLI 界面**：prompt_toolkit + Rich
- **TUI**：React (Ink) + Python JSON-RPC
- **消息网关**：异步架构，支持多种即时通讯平台
- **数据存储**：SQLite (会话) + 文件系统 (技能/记忆)

## 项目统计

- **代码行数**：约 100,000+ 行 Python 代码
- **测试覆盖**：15,000+ 测试用例，700+ 测试文件
- **工具数量**：40+ 内置工具
- **平台支持**：15+ 消息平台
- **技能数量**：27 个内置技能 + 多个可选技能

## 相关信息

- [官方文档](https://hermes-agent.nousresearch.com/docs/)
- [GitHub 仓库](https://github.com/NousResearch/hermes-agent)
- [Discord 社区](https://discord.gg/NousResearch)
