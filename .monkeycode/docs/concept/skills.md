# 技能系统

## 概述

Hermes Agent 的技能系统允许创建、共享和自动执行复杂任务的工作流。

## 技能类型

### 内置技能 (skills/)

默认加载的技能，位于项目 `skills/` 目录：

| 类别 | 技能 |
|------|------|
| apple | Apple 设备控制 |
| autonomous-ai-agents | 自主 AI Agent 相关 |
| creative | 创意写作、设计 |
| data-science | 数据科学 |
| devops | DevOps 工具 |
| github | GitHub 集成 |
| mlops | MLOps 工具 |
| productivity | 生产力工具 |
| software-development |软件开发 |

### 可选技能 (optional-skills/)

按需安装的技能，位于项目 `optional-skills/` 目录：

| 类别 | 技能 |
|------|------|
| mlops | 高级 MLOps 技能 |
| devops | 高级 DevOps 技能 |
| security | 安全工具 |
| communication | 通讯工具 |

## 技能结构

### SKILL.md 格式

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
      api_key:
        type: string
        required: true
---

# 技能内容

技能的详细实现...
```

### 技能文件结构

```
skills/
└── category/
    └── skill-name/
        ├── SKILL.md          # 技能定义
        ├── src/              # 源代码（可选）
        ├── prompts/          # 提示模板（可选）
        └── tests/            # 测试（可选）
```

## 技能命令

```bash
# 列出技能
hermes skills list

# 安装技能
hermes skills install <skill-name>

# 卸载技能
hermes skills uninstall <skill-name>

# 查看技能详情
hermes skills info <skill-name>
```

## 技能执行

### Slash 命令

```bash
/<skill-name>           # 直接执行
/<skill-name> --help    # 查看帮助
```

### 技能加载

技能在需要时加载到系统提示：

```python
# agent/prompt_builder.py
def build_skills_system_prompt(enabled_skills):
    """构建技能相关的系统提示"""
```

## 技能市场

### agentskills.io

官方技能市场：[https://agentskills.io](https://agentskills.io)

### 安装市场技能

```bash
hermes skills install official/<category>/<skill>
```

## 技能开发

### 创建新技能

1. 创建技能目录结构
2. 编写 SKILL.md
3. 实现技能逻辑
4. 测试技能
5. 发布（可选）

### 技能模板

```markdown
---
name: my-skill
description: 我的自定义技能
version: "1.0.0"
platforms: [linux, macos]
metadata:
  hermes:
    tags: [custom, example]
    category: "custom"
---

# My Skill

这个技能实现了...

## 使用方法

```
/my-skill <参数>
```

## 实现细节

技能的具体实现...
```

## 技能守卫

技能执行前进行安全检查：

```python
# tools/skills_guard.py
class SkillsGuard:
    def check_skill_safety(self, skill_name: str) -> bool: ...
    def check_command_safety(self, command: str) -> bool: ...
```

## 技能同步

### 同步配置

```yaml
skills:
  sync:
    enabled: true
    interval: 3600  # 秒
```

### 手动同步

```bash
hermes skills sync
```
