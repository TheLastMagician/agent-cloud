# 模块八：UI 与前端详细设计

> 本文档详细描述 Cursor Cloud Agent 的 Web UI 设计，包括组件架构、状态管理、实时通信、交互流程等。

---

## 1. UI 架构总览

### 1.1 技术栈（推荐）

| 层级 | 技术选型 | 说明 |
|------|---------|------|
| 框架 | React 18+ / Next.js | 组件化 + SSR |
| 状态管理 | Zustand / Jotai | 轻量级状态管理 |
| 样式 | Tailwind CSS + CSS Modules | 实用优先 + 组件隔离 |
| 实时通信 | WebSocket | 流式输出 + 状态同步 |
| Markdown 渲染 | react-markdown + remark/rehype | 支持 GFM + HTML |
| 视频播放 | 原生 HTML5 Video | 无需额外库 |
| 图标 | Lucide React | 一致的图标系统 |
| 代码高亮 | Shiki / Prism | 语法高亮 |

### 1.2 页面层级结构

```
App
├── Header (全局导航)
│   ├── Logo
│   ├── ProjectSelector
│   ├── SessionList
│   └── UserMenu
│
├── MainLayout (三栏布局)
│   ├── ChatPanel (左栏/主栏)
│   │   ├── MessageList
│   │   │   ├── UserMessage
│   │   │   ├── AssistantMessage
│   │   │   │   ├── TextContent
│   │   │   │   ├── ThinkingIndicator
│   │   │   │   ├── ToolCallCard
│   │   │   │   ├── InlineArtifact
│   │   │   │   └── TodoList
│   │   │   └── SystemMessage
│   │   ├── TypingIndicator
│   │   └── InputArea
│   │       ├── TextInput
│   │       ├── FileAttachment
│   │       └── SendButton
│   │
│   ├── DesktopPane (中栏, 可折叠)
│   │   ├── VNCViewer
│   │   ├── ControlBar
│   │   └── StatusIndicator
│   │
│   └── SidePanel (右栏, 可折叠)
│       ├── SecretsManager
│       ├── ArtifactsViewer
│       ├── TerminalViewer
│       └── SessionInfo
│
└── Modals
    ├── SecretInputModal
    ├── TestLoginModal
    ├── ExternalActionModal
    ├── FullscreenImageModal
    └── SettingsModal
```

---

## 2. 核心组件设计

### 2.1 ChatPanel（聊天面板）

```tsx
interface ChatPanelProps {
  sessionId: string;
}

function ChatPanel({ sessionId }: ChatPanelProps) {
  const messages = useMessages(sessionId);
  const streamingMessage = useStreamingMessage(sessionId);
  const [input, setInput] = useState('');

  return (
    <div className="chat-panel">
      {/* 消息列表 */}
      <div className="message-list" ref={scrollRef}>
        {messages.map(msg => (
          <MessageRenderer key={msg.id} message={msg} />
        ))}

        {/* 流式消息 */}
        {streamingMessage && (
          <StreamingMessage message={streamingMessage} />
        )}
      </div>

      {/* 输入区域 */}
      <div className="input-area">
        <textarea
          value={input}
          onChange={e => setInput(e.target.value)}
          placeholder="描述你的任务..."
          onKeyDown={handleKeyDown}
        />
        <button onClick={handleSend} disabled={isProcessing}>
          发送
        </button>
      </div>
    </div>
  );
}
```

### 2.2 MessageRenderer（消息渲染器）

```tsx
interface Message {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  toolCalls?: ToolCall[];
  thinking?: string;
  timestamp: Date;
}

function MessageRenderer({ message }: { message: Message }) {
  switch (message.role) {
    case 'user':
      return <UserMessage content={message.content} />;

    case 'assistant':
      return (
        <AssistantMessage>
          {/* 思考指示器 */}
          {message.thinking && (
            <ThinkingIndicator content={message.thinking} />
          )}

          {/* 文本内容（Markdown 渲染） */}
          <MarkdownRenderer content={message.content} />

          {/* 工具调用卡片 */}
          {message.toolCalls?.map(tc => (
            <ToolCallCard key={tc.id} toolCall={tc} />
          ))}

          {/* 交互式任务（env_setup_actions XML） */}
          <EnvSetupActionsParser content={message.content} />
        </AssistantMessage>
      );

    case 'system':
      return <SystemMessage content={message.content} />;
  }
}
```

### 2.3 ToolCallCard（工具调用卡片）

```tsx
interface ToolCall {
  id: string;
  name: string;
  input: Record<string, any>;
  output?: string;
  status: 'pending' | 'running' | 'completed' | 'error';
  duration_ms?: number;
}

function ToolCallCard({ toolCall }: { toolCall: ToolCall }) {
  const [expanded, setExpanded] = useState(false);

  const icon = getToolIcon(toolCall.name);
  const description = toolCall.input.description || toolCall.name;

  return (
    <div className={`tool-call-card ${toolCall.status}`}>
      {/* 头部 */}
      <div className="header" onClick={() => setExpanded(!expanded)}>
        <span className="icon">{icon}</span>
        <span className="name">{toolCall.name}</span>
        <span className="description">{description}</span>
        <StatusBadge status={toolCall.status} />
        {toolCall.duration_ms && (
          <span className="duration">{toolCall.duration_ms}ms</span>
        )}
        <ChevronIcon direction={expanded ? 'up' : 'down'} />
      </div>

      {/* 展开内容 */}
      {expanded && (
        <div className="details">
          <div className="input">
            <h4>参数</h4>
            <CodeBlock language="json">
              {JSON.stringify(toolCall.input, null, 2)}
            </CodeBlock>
          </div>

          {toolCall.output && (
            <div className="output">
              <h4>输出</h4>
              <CodeBlock>{toolCall.output}</CodeBlock>
            </div>
          )}
        </div>
      )}
    </div>
  );
}

function getToolIcon(toolName: string): string {
  const icons: Record<string, string> = {
    Shell: '⌨️',
    Read: '📄',
    Write: '✏️',
    StrReplace: '🔄',
    Delete: '🗑️',
    Glob: '🔍',
    Grep: '🔎',
    Task: '🤖',
    RecordScreen: '🎬',
    WebSearch: '🌐',
    TodoWrite: '📋',
  };
  return icons[toolName] || '🔧';
}
```

### 2.4 MarkdownRenderer（Markdown 渲染器）

```tsx
import ReactMarkdown from 'react-markdown';
import remarkGfm from 'remark-gfm';
import rehypeRaw from 'rehype-raw';

function MarkdownRenderer({ content }: { content: string }) {
  return (
    <ReactMarkdown
      remarkPlugins={[remarkGfm]}
      rehypePlugins={[rehypeRaw]}
      components={{
        // 自定义图片渲染 → 产物图片查看器
        img: ({ src, alt }) => <ArtifactImage src={src!} alt={alt || ''} />,

        // 自定义视频渲染 → 产物视频播放器
        video: ({ src }) => <ArtifactVideo src={src!} />,

        // 代码块 → 语法高亮
        code: ({ className, children }) => (
          <SyntaxHighlighter language={parseLanguage(className)}>
            {String(children)}
          </SyntaxHighlighter>
        ),

        // 表格 → 响应式表格
        table: ({ children }) => (
          <div className="table-wrapper">
            <table>{children}</table>
          </div>
        ),
      }}
    />
  );
}
```

---

## 3. Desktop Pane（桌面面板）

### 3.1 VNC 查看器

```tsx
interface DesktopPaneProps {
  sessionId: string;
}

function DesktopPane({ sessionId }: DesktopPaneProps) {
  const vncRef = useRef<HTMLDivElement>(null);
  const [connected, setConnected] = useState(false);
  const [interactiveMode, setInteractiveMode] = useState(false);

  useEffect(() => {
    // 建立 VNC WebSocket 连接
    const rfb = new RFB(vncRef.current!, vncWebSocketUrl, {
      credentials: { password: vncPassword },
      wsProtocols: ['binary'],
    });

    rfb.addEventListener('connect', () => setConnected(true));
    rfb.addEventListener('disconnect', () => setConnected(false));

    // 默认只读模式（观察 Agent 操作）
    rfb.viewOnly = !interactiveMode;

    return () => rfb.disconnect();
  }, [sessionId, interactiveMode]);

  return (
    <div className="desktop-pane">
      <div className="control-bar">
        <StatusIndicator connected={connected} />

        <button onClick={() => setInteractiveMode(!interactiveMode)}>
          {interactiveMode ? '🔓 交互模式' : '🔒 观察模式'}
        </button>

        <button onClick={handleFullscreen}>
          全屏
        </button>
      </div>

      <div className="vnc-container" ref={vncRef} />
    </div>
  );
}
```

### 3.2 VNC 服务端设置

```python
class VNCServer:
    """VM 内的 VNC 服务"""

    def setup(self):
        # 启动虚拟帧缓冲 (Xvfb)
        subprocess.Popen([
            "Xvfb", ":0",
            "-screen", "0", "1920x1080x24",
            "-ac",
        ])

        # 启动窗口管理器
        subprocess.Popen(["fluxbox"], env={"DISPLAY": ":0"})

        # 启动 VNC 服务器 (noVNC WebSocket)
        subprocess.Popen([
            "websockify",
            "--web", "/usr/share/novnc/",
            "6080",
            "localhost:5900",
        ])

        # 启动 x11vnc
        subprocess.Popen([
            "x11vnc",
            "-display", ":0",
            "-forever",
            "-nopw",
            "-rfbport", "5900",
            "-shared",
        ])
```

---

## 4. Secrets Manager（密钥管理面板）

```tsx
function SecretsManager({ sessionId }: { sessionId: string }) {
  const [secrets, setSecrets] = useState<SecretEntry[]>([]);
  const [activeTab, setActiveTab] = useState<'personal' | 'team' | 'repo'>('personal');

  return (
    <div className="secrets-manager">
      <h3>🔑 Secrets</h3>

      {/* Tab 切换 */}
      <div className="tabs">
        <Tab active={activeTab === 'personal'} onClick={() => setActiveTab('personal')}>
          个人
        </Tab>
        <Tab active={activeTab === 'team'} onClick={() => setActiveTab('team')}>
          团队
        </Tab>
        <Tab active={activeTab === 'repo'} onClick={() => setActiveTab('repo')}>
          仓库
        </Tab>
      </div>

      {/* Secret 列表 */}
      <div className="secret-list">
        {secrets
          .filter(s => s.scope === activeTab)
          .map(secret => (
            <SecretEntry key={secret.name} secret={secret} />
          ))}
      </div>

      {/* 添加新 Secret */}
      <AddSecretForm scope={activeTab} onAdd={handleAdd} />
    </div>
  );
}

function SecretEntry({ secret }: { secret: SecretEntry }) {
  const [showValue, setShowValue] = useState(false);

  return (
    <div className="secret-entry">
      <span className="name">{secret.name}</span>
      <span className="value">
        {showValue ? secret.value : '••••••••'}
      </span>
      <button onClick={() => setShowValue(!showValue)}>
        {showValue ? '🙈' : '👁️'}
      </button>
      <select value={secret.type} onChange={handleTypeChange}>
        <option value="secret">Secret</option>
        <option value="redacted">Redacted Secret</option>
      </select>
      <button onClick={handleDelete}>🗑️</button>
    </div>
  );
}
```

---

## 5. 交互式任务渲染（env_setup_actions）

### 5.1 XML 解析器

```tsx
function EnvSetupActionsParser({ content }: { content: string }) {
  // 从消息内容中提取 XML
  const xmlMatch = content.match(
    /<env_setup_actions[\s\S]*?<\/env_setup_actions>/
  );
  if (!xmlMatch) return null;

  const actions = parseXml(xmlMatch[0]);

  return (
    <div className="env-setup-actions">
      <h3>⚠️ 需要您的操作</h3>

      {actions.addSecrets && (
        <AddSecretsAction
          secrets={actions.addSecrets.secrets}
          reason={actions.addSecrets.reason}
        />
      )}

      {actions.addTestLogin && (
        <TestLoginAction
          config={actions.addTestLogin}
        />
      )}

      {actions.externalActions?.map(action => (
        <ExternalActionCard
          key={action.id}
          action={action}
        />
      ))}
    </div>
  );
}
```

### 5.2 添加 Secrets 交互

```tsx
function AddSecretsAction({ secrets, reason }: AddSecretsActionProps) {
  const [values, setValues] = useState<Record<string, string>>({});

  return (
    <div className="action-card">
      <h4>🔑 添加 Secrets</h4>
      {reason && <p className="reason">{reason}</p>}

      {secrets.map(secret => (
        <div key={secret.name} className="secret-input">
          <label>{secret.name}</label>
          <input
            type="password"
            value={values[secret.name] || ''}
            onChange={e => setValues({
              ...values,
              [secret.name]: e.target.value,
            })}
            placeholder={`输入 ${secret.name}`}
          />
        </div>
      ))}

      <button onClick={() => handleSubmit(values)} className="primary">
        保存并继续
      </button>
    </div>
  );
}
```

### 5.3 外部操作卡片

```tsx
function ExternalActionCard({ action }: { action: ExternalAction }) {
  const [completed, setCompleted] = useState(false);

  return (
    <div className={`action-card ${completed ? 'completed' : ''}`}>
      <h4>🔧 {action.title}</h4>

      <div className="instructions">
        <ReactMarkdown>{action.instructions}</ReactMarkdown>
      </div>

      <button
        onClick={() => setCompleted(true)}
        className={completed ? 'success' : 'primary'}
        disabled={completed}
      >
        {completed ? '✅ 已完成' : '标记为已完成'}
      </button>
    </div>
  );
}
```

---

## 6. 实时通信

### 6.1 WebSocket 协议

```typescript
interface WebSocketMessage {
  type: MessageType;
  sessionId: string;
  payload: any;
  timestamp: number;
}

enum MessageType {
  // Agent → UI
  TEXT_DELTA = 'text_delta',           // 流式文本增量
  THINKING_DELTA = 'thinking_delta',   // 思考内容增量
  TOOL_CALL_START = 'tool_call_start', // 工具调用开始
  TOOL_CALL_INPUT = 'tool_call_input', // 工具参数增量
  TOOL_CALL_RESULT = 'tool_call_result', // 工具结果
  TODO_UPDATE = 'todo_update',         // TODO 列表更新
  SESSION_END = 'session_end',         // 会话结束
  ARTIFACT_READY = 'artifact_ready',   // 产物就绪
  VNC_FRAME = 'vnc_frame',            // VNC 帧数据

  // UI → Server
  USER_MESSAGE = 'user_message',       // 用户发送消息
  SECRET_SUBMIT = 'secret_submit',     // 提交 Secret
  ACTION_COMPLETE = 'action_complete', // 标记外部操作完成
  SESSION_CANCEL = 'session_cancel',   // 取消会话
}
```

### 6.2 WebSocket Hook

```tsx
function useWebSocket(sessionId: string) {
  const [ws, setWs] = useState<WebSocket | null>(null);
  const [status, setStatus] = useState<'connecting' | 'connected' | 'disconnected'>('connecting');

  useEffect(() => {
    const socket = new WebSocket(`wss://api.example.com/ws/${sessionId}`);

    socket.onopen = () => setStatus('connected');
    socket.onclose = () => {
      setStatus('disconnected');
      // 自动重连
      setTimeout(() => reconnect(), 3000);
    };

    socket.onmessage = (event) => {
      const msg: WebSocketMessage = JSON.parse(event.data);
      handleMessage(msg);
    };

    setWs(socket);
    return () => socket.close();
  }, [sessionId]);

  const handleMessage = (msg: WebSocketMessage) => {
    switch (msg.type) {
      case MessageType.TEXT_DELTA:
        appendToStreamingMessage(msg.payload.text);
        break;

      case MessageType.TOOL_CALL_START:
        addToolCall(msg.payload);
        break;

      case MessageType.TOOL_CALL_RESULT:
        updateToolCallResult(msg.payload);
        break;

      case MessageType.TODO_UPDATE:
        updateTodos(msg.payload.todos);
        break;

      case MessageType.ARTIFACT_READY:
        addArtifact(msg.payload);
        break;

      case MessageType.SESSION_END:
        finalizeSession(msg.payload);
        break;
    }
  };

  return { ws, status };
}
```

---

## 7. 状态管理

### 7.1 全局状态结构

```typescript
interface AppState {
  // 会话状态
  sessions: Record<string, Session>;
  activeSessionId: string | null;

  // 消息状态
  messages: Record<string, Message[]>;
  streamingMessage: Record<string, Partial<Message> | null>;

  // TODO 状态
  todos: Record<string, TodoItem[]>;

  // Secrets 状态
  secrets: SecretEntry[];

  // 产物状态
  artifacts: Record<string, ArtifactMeta[]>;

  // 连接状态
  wsStatus: 'connecting' | 'connected' | 'disconnected';

  // UI 状态
  ui: {
    desktopPaneVisible: boolean;
    sidePanelVisible: boolean;
    sidePanelTab: 'secrets' | 'artifacts' | 'terminals';
  };
}
```

### 7.2 Zustand Store

```typescript
import { create } from 'zustand';

const useStore = create<AppState>((set, get) => ({
  sessions: {},
  activeSessionId: null,
  messages: {},
  streamingMessage: {},
  todos: {},
  secrets: [],
  artifacts: {},
  wsStatus: 'disconnected',
  ui: {
    desktopPaneVisible: false,
    sidePanelVisible: true,
    sidePanelTab: 'secrets',
  },

  // Actions
  appendTextDelta: (sessionId: string, text: string) =>
    set(state => ({
      streamingMessage: {
        ...state.streamingMessage,
        [sessionId]: {
          ...(state.streamingMessage[sessionId] || {}),
          content: (state.streamingMessage[sessionId]?.content || '') + text,
        },
      },
    })),

  finalizeMessage: (sessionId: string) =>
    set(state => {
      const streaming = state.streamingMessage[sessionId];
      if (!streaming) return state;

      return {
        messages: {
          ...state.messages,
          [sessionId]: [
            ...(state.messages[sessionId] || []),
            streaming as Message,
          ],
        },
        streamingMessage: {
          ...state.streamingMessage,
          [sessionId]: null,
        },
      };
    }),

  updateTodos: (sessionId: string, todos: TodoItem[]) =>
    set(state => ({
      todos: { ...state.todos, [sessionId]: todos },
    })),
}));
```

---

## 8. 响应式布局

### 8.1 断点设计

| 断点 | 宽度 | 布局 |
|------|------|------|
| Mobile | < 768px | 单栏（Chat only） |
| Tablet | 768-1024px | 双栏（Chat + Side） |
| Desktop | 1024-1440px | 三栏（Chat + Desktop + Side） |
| Wide | > 1440px | 三栏加宽 |

### 8.2 面板折叠

```tsx
function MainLayout() {
  const { desktopPaneVisible, sidePanelVisible } = useStore(s => s.ui);

  const gridTemplate = useMemo(() => {
    if (desktopPaneVisible && sidePanelVisible) {
      return '1fr 1fr 300px';
    } else if (desktopPaneVisible) {
      return '1fr 1fr';
    } else if (sidePanelVisible) {
      return '1fr 300px';
    } else {
      return '1fr';
    }
  }, [desktopPaneVisible, sidePanelVisible]);

  return (
    <div
      className="main-layout"
      style={{ display: 'grid', gridTemplateColumns: gridTemplate }}
    >
      <ChatPanel />
      {desktopPaneVisible && <DesktopPane />}
      {sidePanelVisible && <SidePanel />}
    </div>
  );
}
```

---

## 9. TODO 列表 UI

```tsx
function TodoList({ todos }: { todos: TodoItem[] }) {
  const statusColors = {
    pending: '#9ca3af',
    in_progress: '#3b82f6',
    completed: '#10b981',
    cancelled: '#ef4444',
  };

  const statusIcons = {
    pending: '⬜',
    in_progress: '🔵',
    completed: '✅',
    cancelled: '❌',
  };

  return (
    <div className="todo-list">
      {todos.map(todo => (
        <div
          key={todo.id}
          className="todo-item"
          style={{ borderLeft: `3px solid ${statusColors[todo.status]}` }}
        >
          <span className="icon">{statusIcons[todo.status]}</span>
          <span className="content">{todo.content}</span>
          <span className="status">{todo.status}</span>
        </div>
      ))}
    </div>
  );
}
```

---

## 10. 主题与设计系统

### 10.1 颜色系统

```css
:root {
  /* 主色调 */
  --color-primary: #3b82f6;
  --color-primary-hover: #2563eb;

  /* 状态色 */
  --color-success: #10b981;
  --color-warning: #f59e0b;
  --color-error: #ef4444;
  --color-info: #3b82f6;

  /* 灰阶 */
  --color-gray-50: #f9fafb;
  --color-gray-100: #f3f4f6;
  --color-gray-200: #e5e7eb;
  --color-gray-300: #d1d5db;
  --color-gray-700: #374151;
  --color-gray-800: #1f2937;
  --color-gray-900: #111827;

  /* 背景 */
  --bg-primary: #ffffff;
  --bg-secondary: #f9fafb;
  --bg-chat: #ffffff;
  --bg-code: #1e1e1e;

  /* 文本 */
  --text-primary: #111827;
  --text-secondary: #6b7280;
  --text-muted: #9ca3af;

  /* 圆角 */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;

  /* 阴影 */
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.1);
}

/* 暗色主题 */
[data-theme="dark"] {
  --bg-primary: #111827;
  --bg-secondary: #1f2937;
  --bg-chat: #111827;
  --text-primary: #f9fafb;
  --text-secondary: #9ca3af;
}
```

### 10.2 排版系统

```css
:root {
  --font-sans: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
  --font-mono: 'JetBrains Mono', 'Fira Code', monospace;

  --text-xs: 0.75rem;
  --text-sm: 0.875rem;
  --text-base: 1rem;
  --text-lg: 1.125rem;
  --text-xl: 1.25rem;

  --line-height-tight: 1.25;
  --line-height-normal: 1.5;
  --line-height-relaxed: 1.75;
}
```
