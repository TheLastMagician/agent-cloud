# 模块二十六：SDK 与第三方集成设计

> 本文档详细描述客户端 SDK、Webhook 集成、CI/CD 集成、IDE 插件接口等。

---

## 1. 客户端 SDK

### 1.1 SDK 架构

```
┌──────────────────────────────────────────┐
│             Client SDK                    │
│                                          │
│  ┌──────────────────────────────────┐    │
│  │  High-Level API                   │    │
│  │  agent.run("Fix the login bug")   │    │
│  │  agent.follow_up("Change color")  │    │
│  └──────────┬───────────────────────┘    │
│             │                            │
│  ┌──────────▼───────────────────────┐    │
│  │  Session Manager                  │    │
│  │  - 会话创建/恢复/取消              │    │
│  │  - 跟进消息                       │    │
│  └──────────┬───────────────────────┘    │
│             │                            │
│  ┌──────────▼───────────────────────┐    │
│  │  Transport Layer                  │    │
│  │  ├── REST Client (CRUD)           │    │
│  │  └── WebSocket Client (实时)      │    │
│  └──────────┬───────────────────────┘    │
│             │                            │
│  ┌──────────▼───────────────────────┐    │
│  │  Auth & Config                    │    │
│  │  ├── API Key 认证                 │    │
│  │  ├── OAuth Token 管理             │    │
│  │  └── 配置管理                     │    │
│  └──────────────────────────────────┘    │
└──────────────────────────────────────────┘
```

### 1.2 TypeScript/JavaScript SDK

```typescript
import { CursorAgent } from '@cursor/agent-sdk';

// 初始化
const agent = new CursorAgent({
  apiKey: 'sk-...',
  baseUrl: 'https://api.cursor.com',
});

// 创建会话并运行任务
const session = await agent.run({
  repo: 'owner/repo',
  branch: 'main',
  task: '修复登录页面的样式问题',
  onTextDelta: (text) => process.stdout.write(text),
  onToolCall: (call) => console.log(`[Tool] ${call.name}`),
  onTodoUpdate: (todos) => console.log('TODOs:', todos),
  onArtifact: (artifact) => console.log(`Artifact: ${artifact.filename}`),
});

// 等待完成
const result = await session.waitForCompletion();
console.log('Status:', result.status);
console.log('Commits:', result.commits);
console.log('Artifacts:', result.artifacts);

// 跟进
const followUp = await session.followUp('颜色改成蓝色');
await followUp.waitForCompletion();
```

### 1.3 Python SDK

```python
from cursor_agent import CursorAgent

agent = CursorAgent(api_key="sk-...")

# 同步模式
result = agent.run(
    repo="owner/repo",
    branch="main",
    task="修复登录页面的样式问题",
)
print(f"Status: {result.status}")
print(f"Commits: {result.commits}")

# 流式模式
async for event in agent.run_stream(
    repo="owner/repo",
    task="添加暗色模式支持",
):
    match event.type:
        case "text_delta":
            print(event.text, end="", flush=True)
        case "tool_call_start":
            print(f"\n[Tool] {event.tool_name}")
        case "session_end":
            print(f"\nDone: {event.status}")
```

### 1.4 CLI 工具

```bash
# 安装
npm install -g @cursor/agent-cli

# 运行任务
cursor-agent run --repo owner/repo --task "修复 bug #42"

# 查看会话
cursor-agent sessions list
cursor-agent sessions view sess_xyz

# 管理 Secrets
cursor-agent secrets set DATABASE_URL "postgres://..."
cursor-agent secrets list

# 查看产物
cursor-agent artifacts list sess_xyz
cursor-agent artifacts download sess_xyz demo.mp4
```

---

## 2. Webhook 集成

### 2.1 Webhook 事件

```python
WEBHOOK_EVENTS = {
    "session.completed": "会话完成",
    "session.failed": "会话失败",
    "session.waiting": "等待用户操作",
    "commit.pushed": "代码已推送",
    "artifact.ready": "产物已就绪",
    "quota.warning": "配额预警 (80%)",
    "quota.exceeded": "配额超限",
}
```

### 2.2 Webhook 配置

```yaml
# 通过 API 配置
POST /api/v1/webhooks
{
  "url": "https://your-server.com/webhook",
  "events": ["session.completed", "session.failed"],
  "secret": "whsec_...",  # 签名验证密钥
  "active": true
}
```

### 2.3 Webhook Payload

```json
{
  "id": "evt_abc123",
  "type": "session.completed",
  "timestamp": "2026-03-03T10:35:00Z",
  "data": {
    "session_id": "sess_xyz",
    "repo": "owner/repo",
    "branch": "main",
    "status": "completed",
    "commits": [
      {"hash": "abc123", "message": "Fix login page styling"}
    ],
    "artifacts": [
      {"filename": "demo.mp4", "url": "https://..."}
    ],
    "usage": {
      "input_tokens": 45000,
      "output_tokens": 12000,
      "duration_seconds": 300
    }
  },
  "signature": "sha256=..."
}
```

### 2.4 签名验证

```python
import hmac
import hashlib

def verify_webhook(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)
```

---

## 3. CI/CD 集成

### 3.1 GitHub Actions

```yaml
# .github/workflows/cursor-agent.yml
name: Cursor Agent Task
on:
  issues:
    types: [labeled]

jobs:
  agent-task:
    if: contains(github.event.label.name, 'cursor-agent')
    runs-on: ubuntu-latest
    steps:
      - name: Run Cursor Agent
        uses: cursor/agent-action@v1
        with:
          api-key: ${{ secrets.CURSOR_API_KEY }}
          task: ${{ github.event.issue.body }}
          repo: ${{ github.repository }}
          branch: main
          wait-for-completion: true
          timeout: 1800  # 30 minutes

      - name: Comment Result
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              body: `Agent completed: ${process.env.AGENT_RESULT}`
            })
```

### 3.2 GitLab CI

```yaml
# .gitlab-ci.yml
cursor-agent:
  stage: automate
  image: cursor/agent-cli:latest
  script:
    - cursor-agent run
        --repo $CI_PROJECT_PATH
        --branch $CI_COMMIT_REF_NAME
        --task "$TASK_DESCRIPTION"
        --wait
  rules:
    - if: $CI_PIPELINE_SOURCE == "trigger"
```

---

## 4. IDE 插件接口

### 4.1 VS Code 扩展 API

```typescript
// VS Code Extension API 集成点

interface CursorAgentExtension {
  // 启动 Cloud Agent 会话
  startSession(options: {
    task: string;
    repo: string;
    branch: string;
    context?: {
      currentFile?: string;
      selectedText?: string;
      diagnostics?: vscode.Diagnostic[];
    };
  }): Promise<AgentSession>;

  // 查看会话状态
  getSession(sessionId: string): Promise<SessionInfo>;

  // 在编辑器中应用 Agent 的变更
  applyChanges(sessionId: string): Promise<void>;

  // 事件订阅
  onSessionEvent(
    sessionId: string,
    callback: (event: AgentEvent) => void
  ): vscode.Disposable;
}
```

---

## 5. MCP (Model Context Protocol) 扩展

### 5.1 自定义 MCP 服务器集成

```python
# 用户可以连接自定义 MCP 服务器来扩展 Agent 能力

class MCPServerConfig:
    """MCP 服务器配置"""
    servers: List[MCPServer] = [
        MCPServer(
            name="internal-api",
            url="http://internal.company.com/mcp",
            auth={"type": "bearer", "token": "..."},
            resources=["database-schema", "api-docs"],
        ),
        MCPServer(
            name="design-system",
            url="http://figma-mcp.company.com",
            resources=["components", "tokens"],
        ),
    ]

# Agent 可以通过 ListMcpResources / FetchMcpResource
# 访问这些自定义资源
```

### 5.2 自定义工具注册 (未来)

```python
# 未来可能支持用户注册自定义工具

class CustomTool:
    name = "deploy_staging"
    description = "Deploy to staging environment"
    parameters = {
        "service": {"type": "string", "required": True},
        "version": {"type": "string", "required": True},
    }

    def execute(self, service: str, version: str) -> str:
        # 调用内部部署系统
        return deploy(service, version)
```
