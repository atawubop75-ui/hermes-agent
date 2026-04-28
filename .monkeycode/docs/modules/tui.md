# TUI 模块 (ui-tui/ + tui_gateway/)

## 概述

Hermes Agent 提供两种终端界面：
1. **经典 CLI** - 基于 prompt_toolkit 的交互式界面
2. **TUI** - 基于 React (Ink) 的现代终端 UI

## 架构

```
hermes --tui
    │
    ├─► Node (Ink) ◄──── stdio JSON-RPC ────► Python (tui_gateway)
    │      │                                              │
    │      │                                              │
    │      └─────── 渲染：transcript, composer, prompts ──┘
    │                                                      │
    └──────────────────────────────────► AIAgent + tools + sessions
```

## 目录结构

### ui-tui (React/Ink 前端)

```
ui-tui/
├── package.json
├── tsconfig.json
├── inkrc.js              # Ink 配置
├── src/
│   ├── entry.tsx         # 入口点
│   ├── app.tsx           # 主应用
│   ├── components/
│   │   ├── layout/
│   │   │   ├── composer.tsx      # 输入composer
│   │   │   ├── header.tsx        # 头部
│   │   │   ├── sidebar.tsx       # 侧边栏
│   │   │   └── statusBar.tsx     # 状态栏
│   │   ├── message/
│   │   │   ├── messageLine.tsx   # 消息行
│   │   │   ├── messageBox.tsx    # 消息框
│   │   │   └── thinking.tsx      # 思考动画
│   │   ├── prompts/
│   │   │   ├── prompts.tsx       # 提示
│   │   │   └── maskedPrompt.tsx  # 密码提示
│   │   └── ...
│   ├── hooks/
│   │   ├── useCompletion.ts      # 完成 hook
│   │   ├── useGateway.ts         # 网关 hook
│   │   └── ...
│   ├── lib/
│   │   ├── gatewayClient.ts      # 网关客户端
│   │   ├── theme.ts              # 主题
│   │   └── ...
│   └── utils/
│       └── ...
└── dist/                 # 构建输出
```

### tui_gateway (Python 后端)

```
tui_gateway/
├── __init__.py
├── server.py             # JSON-RPC 服务器
├── protocol.py           # 协议定义
├── session.py            # 会话管理
└── ...
```

## 通信协议

### 传输

- **协议**: Newline-delimited JSON-RPC 2.0
- **方向**: 双向 stdio
- **请求**: Ink → Python
- **事件**: Python → Ink

### 方法/事件目录

| 类型 | 名称 | 说明 |
|------|------|------|
| Method | `prompt.submit` | 提交提示 |
| Method | `session.list` | 列出会话 |
| Method | `session.resume` | 恢复会话 |
| Method | `approval.respond` | 响应审批 |
| Method | `clarify.respond` | 响应澄清 |
| Method | `sudo.respond` | 响应 sudo |
| Method | `secret.respond` | 响应密钥 |
| Method | `slash.exec` | 执行 slash 命令 |
| Method | `complete.slash` | Slash 命令补全 |
| Method | `complete.path` | 路径补全 |
| Event | `message.delta` | 消息增量 |
| Event | `message.complete` | 消息完成 |
| Event | `tool.start` | 工具开始 |
| Event | `tool.progress` | 工具进度 |
| Event | `tool.complete` | 工具完成 |
| Event | `approval.request` | 审批请求 |
| Event | `gateway.ready` | 网关就绪 |
| Event | `error` | 错误 |

## 组件

### Composer (输入框)

```tsx
// src/components/layout/composer.tsx
const Composer: React.FC<ComposerProps> = ({
    onSubmit,
    onInterrupt,
    disabled
}) => {
    // 多行编辑
    // Slash 命令自动补全
    // 快捷键支持
}
```

### MessageLine (消息行)

```tsx
// src/components/message/messageLine.tsx
const MessageLine: React.FC<MessageLineProps> = ({
    role,
    content,
    reasoning
}) => {
    // 角色图标
    // 内容渲染
    // Markdown 支持
    // 流式输出
}
```

### Thinking (思考动画)

```tsx
// src/components/message/thinking.tsx
const Thinking: React.FC<ThinkingProps> = ({
    verb,
    faces
}) => {
    // 旋转动画
    // 动词显示
    // 表情符号
}
```

### Prompts (提示)

```tsx
// src/components/prompts/prompts.tsx
const Prompts: React.FC<PromptsProps> = ({
    type,        // 'confirm' | 'input' | 'select'
    title,
    options,
    onRespond
}) => {
    // 确认提示
    // 输入提示
    // 选择提示
}
```

## 主题

### theme.ts

```typescript
interface Theme {
    colors: {
        background: string;
        foreground: string;
        accent: string;
        error: string;
        success: string;
        muted: string;
    };
    spinner: {
        frames: string[];
        verb?: string[];
    };
}

export const defaultTheme: Theme = {
    colors: {
        background: '#1a1a2e',
        foreground: '#eaeaea',
        accent: '#ffd700',
        error: '#ff6b6b',
        success: '#4ecdc4',
        muted: '#666666'
    },
    spinner: {
        frames: ['⠋', '⠙', '⠹', '⠸', '⠼', '⠴', '⠦', '⠧', '⠇', '⠏'],
        verb: ['thinking', 'processing', 'working']
    }
};
```

### branding.tsx

```tsx
// 品牌组件
const Branding: React.FC<{ skin: SkinConfig }> = ({ skin }) => {
    // Agent 名称
    // 欢迎消息
    // 响应标签
}
```

## 命令处理

### Built-in 命令

本地处理（不需要服务器）：

```typescript
const BUILT_IN_COMMANDS = [
    'help',
    'quit',
    'clear',
    'resume',
    'copy',
    'paste'
];
```

### Slash 命令流程

```typescript
async function handleSlashCommand(input: string) {
    // 本地命令？直接处理
    if (isBuiltIn(command)) {
        return handleBuiltIn(command);
    }

    // 发送到服务器
    const result = await gatewayClient.slashExec({
        command,
        args
    });

    return result;
}
```

## 开发

### 开发模式

```bash
cd ui-tui
npm install
npm run dev     # watch 模式，重建 + tsx --watch
npm start       # 生产模式
```

### 构建

```bash
npm run build   # 完整构建
npm run type-check  # 类型检查
npm run lint    # ESLint
npm run fmt     # Prettier
npm test        # Vitest
```

### 发布

```bash
# 构建 Ink 应用
npm run build

# 打包
tar -czf hermes-ink.tar.gz dist/
```

## Dashboard 集成

### PTY Bridge

```python
# hermes_cli/pty_bridge.py
class PtyBridge:
    """PTY 桥接到 Web"""

    async def handle_websocket(self, websocket: WebSocket):
        """处理 WebSocket 连接"""
        # 启动 hermes --tui
        # 转发 PTY 数据
```

### Web 端点

```
GET /api/pty?token=<session_token>
Upgrade: websocket
```

### ChatPage

```tsx
// web/src/pages/ChatPage.tsx
const ChatPage: React.FC = () => {
    return (
        <Terminal
            renderer={WebGLRenderer}
            addons={[fit, unicode11]}
        />
    );
};
```

## 配置

### 启用 TUI

```bash
# 命令行
hermes --tui

# 环境变量
HERMES_TUI=1 hermes
```

### TUI 配置

```yaml
display:
  tui:
    theme: "default"
    show_thinking: true
    streaming: true
```

## 与经典 CLI 的区别

| 特性 | 经典 CLI | TUI |
|------|----------|-----|
| 渲染引擎 | prompt_toolkit | React/Ink |
| 流式输出 | 支持 | 支持 |
| 组件化 | 否 | 是 |
| 状态管理 | 单一 | 组件化 |
| 主题系统 | 外部皮肤 | 内部主题 |
| Dashboard 集成 | 有限 | 完整 |
