# 模块十五：后端 API 与 WebSocket 协议详细设计

> 本文档详细描述系统的 REST API、WebSocket 实时通信协议、认证机制、错误码体系。

---

## 1. API 架构总览

```
客户端 (Web App)
    │
    ├── REST API (HTTPS)
    │   ├── /api/v1/sessions     会话管理
    │   ├── /api/v1/secrets      密钥管理
    │   ├── /api/v1/artifacts    产物管理
    │   ├── /api/v1/repos        仓库管理
    │   └── /api/v1/users        用户管理
    │
    └── WebSocket (WSS)
        └── /ws/{session_id}     实时流
```

---

## 2. 认证机制

### 2.1 JWT Token 流程

```
登录/OAuth → 获取 access_token (短期) + refresh_token (长期)
    │
    ├── REST API: Authorization: Bearer {access_token}
    └── WebSocket: ?token={access_token} 或 first message auth

Token 结构:
{
  "sub": "user_123",
  "email": "user@example.com",
  "org_id": "org_456",
  "plan": "pro",
  "iat": 1709420000,
  "exp": 1709423600,  // 1小时
  "scope": ["agent:execute", "secrets:read", "repos:read"]
}
```

---

## 3. REST API 详细定义

### 3.1 会话管理 API

```yaml
# 创建新会话
POST /api/v1/sessions
  Request:
    {
      "repo_id": "repo_abc",
      "branch": "main",
      "message": "帮我修复登录页面的样式问题",
      "task_type": "general",        # general | env_setup | bug_fix
      "attachments": [               # 可选: 附件
        { "type": "file", "path": "screenshot.png", "content": "base64..." }
      ],
      "model_preference": null       # 可选: 模型偏好
    }
  Response: 201 Created
    {
      "session_id": "sess_xyz",
      "status": "creating",
      "vm_id": "vm_001",
      "created_at": "2026-03-03T10:30:00Z",
      "ws_url": "wss://api.example.com/ws/sess_xyz"
    }

# 获取会话状态
GET /api/v1/sessions/{session_id}
  Response: 200 OK
    {
      "session_id": "sess_xyz",
      "status": "running",           # creating|initializing|running|waiting|completed|failed|cancelled
      "repo_id": "repo_abc",
      "branch": "main",
      "vm_id": "vm_001",
      "created_at": "2026-03-03T10:30:00Z",
      "updated_at": "2026-03-03T10:35:00Z",
      "usage": {
        "input_tokens": 45000,
        "output_tokens": 12000,
        "tool_calls": 32,
        "duration_seconds": 300
      },
      "artifacts": [
        {
          "filename": "demo.mp4",
          "size": 5242880,
          "mime_type": "video/mp4",
          "url": "https://cdn.example.com/artifacts/sess_xyz/demo.mp4"
        }
      ],
      "commits": [
        {
          "hash": "abc123",
          "message": "Fix login page styling",
          "timestamp": "2026-03-03T10:34:00Z"
        }
      ]
    }

# 发送跟进消息
POST /api/v1/sessions/{session_id}/messages
  Request:
    {
      "content": "颜色改成蓝色而不是绿色",
      "attachments": []
    }
  Response: 200 OK
    {
      "message_id": "msg_789",
      "status": "accepted"
    }

# 取消会话
POST /api/v1/sessions/{session_id}/cancel
  Response: 200 OK
    {
      "status": "cancelled"
    }

# 列出用户会话
GET /api/v1/sessions?repo_id={repo_id}&status={status}&limit=20&offset=0
  Response: 200 OK
    {
      "sessions": [...],
      "total": 42,
      "has_more": true
    }
```

### 3.2 密钥管理 API

```yaml
# 创建 Secret
POST /api/v1/secrets
  Request:
    {
      "name": "DATABASE_URL",
      "value": "postgres://...",
      "type": "secret",              # secret | redacted
      "scope": "personal",           # personal | team | repo
      "repo_id": null                # scope=repo 时必填
    }
  Response: 201 Created
    {
      "id": "sec_001",
      "name": "DATABASE_URL",
      "type": "secret",
      "scope": "personal",
      "created_at": "2026-03-03T10:00:00Z"
    }

# 列出 Secrets
GET /api/v1/secrets?scope={scope}&repo_id={repo_id}
  Response: 200 OK
    {
      "secrets": [
        {
          "id": "sec_001",
          "name": "DATABASE_URL",
          "type": "secret",
          "scope": "personal",
          "created_at": "2026-03-03T10:00:00Z",
          "updated_at": "2026-03-03T10:00:00Z"
          # 注意: 不返回 value
        }
      ]
    }

# 更新 Secret
PUT /api/v1/secrets/{secret_id}
  Request:
    {
      "value": "postgres://new-url...",
      "type": "redacted"
    }
  Response: 200 OK

# 删除 Secret
DELETE /api/v1/secrets/{secret_id}
  Response: 204 No Content
```

### 3.3 产物管理 API

```yaml
# 获取会话产物列表
GET /api/v1/sessions/{session_id}/artifacts
  Response: 200 OK
    {
      "artifacts": [
        {
          "filename": "screenshot_login.webp",
          "size": 124000,
          "mime_type": "image/webp",
          "url": "https://cdn.example.com/artifacts/sess_xyz/screenshot_login.webp",
          "created_at": "2026-03-03T10:33:00Z"
        }
      ]
    }

# 下载产物
GET /api/v1/sessions/{session_id}/artifacts/{filename}
  Response: 302 Redirect → CDN URL
    或
  Response: 200 OK (直接流式下载)
```

### 3.4 仓库管理 API

```yaml
# 列出用户仓库
GET /api/v1/repos
  Response: 200 OK
    {
      "repos": [
        {
          "id": "repo_abc",
          "full_name": "user/project",
          "default_branch": "main",
          "is_public": false,
          "has_snapshot": true,
          "last_session_at": "2026-03-02T15:00:00Z"
        }
      ]
    }

# 获取仓库环境配置
GET /api/v1/repos/{repo_id}/environment
  Response: 200 OK
    {
      "update_script": "npm install",
      "snapshot_id": "snap_001",
      "required_secrets": ["DATABASE_URL", "API_KEY"],
      "agents_md_exists": true
    }
```

---

## 4. WebSocket 协议

### 4.1 连接建立

```
客户端:
  ws = new WebSocket("wss://api.example.com/ws/sess_xyz?token={jwt}")

服务端:
  验证 JWT → 验证 session 权限 → 接受连接

心跳:
  客户端每 30 秒发送 ping
  服务端回复 pong
  60 秒无心跳 → 断开

重连:
  客户端断开后自动重连 (指数退避)
  重连后服务端发送丢失的事件 (基于 last_event_id)
```

### 4.2 消息格式

```typescript
// 所有 WebSocket 消息的基础格式
interface WSMessage {
  type: string;
  event_id: string;        // 全局递增, 用于重连恢复
  session_id: string;
  timestamp: number;       // Unix ms
  payload: any;
}
```

### 4.3 服务端 → 客户端事件

```typescript
// === Agent 文本输出 ===

// 思考开始
{ type: "thinking_start", payload: {} }

// 思考增量 (不展示给用户, 可选展示指示器)
{ type: "thinking_delta", payload: { text: "让我分析..." } }

// 思考结束
{ type: "thinking_end", payload: {} }

// 文本增量 (实时展示)
{ type: "text_delta", payload: { text: "我来帮你" } }

// 文本结束
{ type: "text_end", payload: {} }

// === 工具调用 ===

// 工具调用开始
{
  type: "tool_call_start",
  payload: {
    tool_call_id: "tc_001",
    tool_name: "Shell",
    description: "查看项目结构"  // 来自 Shell 的 description 参数
  }
}

// 工具参数增量
{
  type: "tool_call_input_delta",
  payload: {
    tool_call_id: "tc_001",
    input_delta: "{\"command\": \"ls -la\"}"
  }
}

// 工具执行中 (状态更新)
{
  type: "tool_call_executing",
  payload: {
    tool_call_id: "tc_001"
  }
}

// 工具结果
{
  type: "tool_call_result",
  payload: {
    tool_call_id: "tc_001",
    output: "Exit code: 0\n\nCommand output:\n```\n...\n```",
    duration_ms: 450,
    is_error: false
  }
}

// === TODO 更新 ===
{
  type: "todo_update",
  payload: {
    todos: [
      { id: "t1", content: "安装依赖", status: "completed" },
      { id: "t2", content: "配置环境", status: "in_progress" },
      { id: "t3", content: "运行测试", status: "pending" }
    ]
  }
}

// === 产物就绪 ===
{
  type: "artifact_ready",
  payload: {
    filename: "demo.mp4",
    size: 5242880,
    mime_type: "video/mp4",
    url: "https://cdn.example.com/..."
  }
}

// === 会话状态变更 ===
{
  type: "session_status",
  payload: {
    status: "running",          // creating|initializing|running|waiting|completed|failed
    message: "Agent 正在执行"
  }
}

// === 会话结束 ===
{
  type: "session_end",
  payload: {
    status: "completed",
    usage: {
      input_tokens: 45000,
      output_tokens: 12000,
      tool_calls: 32,
      duration_seconds: 300
    },
    commits: [
      { hash: "abc123", message: "Fix login page styling" }
    ]
  }
}

// === 错误 ===
{
  type: "error",
  payload: {
    code: "VM_INIT_FAILED",
    message: "VM 初始化失败, 请重试"
  }
}
```

### 4.4 客户端 → 服务端事件

```typescript
// 发送消息
{
  type: "user_message",
  payload: {
    content: "请修改颜色为蓝色",
    attachments: []
  }
}

// 提交 Secret
{
  type: "secret_submit",
  payload: {
    secrets: {
      "DATABASE_URL": "postgres://...",
      "API_KEY": "sk-..."
    }
  }
}

// 标记外部操作完成
{
  type: "action_complete",
  payload: {
    action_id: "create_oauth_app"
  }
}

// 取消会话
{
  type: "session_cancel",
  payload: {}
}

// 心跳
{
  type: "ping",
  payload: {}
}
```

### 4.5 重连与事件恢复

```typescript
// 客户端重连时附带 last_event_id
ws = new WebSocket("wss://.../ws/sess_xyz?token=...&last_event_id=42")

// 服务端: 从 event_id=43 开始重放所有事件
// 保证消息不丢失
```

---

## 5. 错误码体系

### 5.1 HTTP 错误码

| HTTP Code | 错误码 | 说明 |
|-----------|--------|------|
| 400 | INVALID_REQUEST | 请求参数无效 |
| 401 | UNAUTHORIZED | 未认证 |
| 403 | FORBIDDEN | 无权限 |
| 404 | NOT_FOUND | 资源不存在 |
| 409 | CONFLICT | 冲突 (如重复创建) |
| 422 | VALIDATION_ERROR | 校验失败 |
| 429 | RATE_LIMITED | 超出速率限制 |
| 500 | INTERNAL_ERROR | 内部错误 |
| 503 | SERVICE_UNAVAILABLE | 服务不可用 |

### 5.2 业务错误码

| 错误码 | 说明 |
|--------|------|
| SESSION_LIMIT_EXCEEDED | 超出并发会话限制 |
| VM_PROVISION_FAILED | VM 分配失败 |
| VM_INIT_FAILED | VM 初始化失败 |
| SNAPSHOT_NOT_FOUND | 快照不存在 |
| SNAPSHOT_RESTORE_FAILED | 快照恢复失败 |
| SECRET_NOT_FOUND | Secret 不存在 |
| REPO_ACCESS_DENIED | 仓库访问被拒 |
| AGENT_TIMEOUT | Agent 执行超时 |
| AGENT_ERROR | Agent 运行时错误 |
| UPDATE_SCRIPT_FAILED | 更新脚本执行失败 |
| QUOTA_EXCEEDED | 配额超限 |

### 5.3 错误响应格式

```json
{
  "error": {
    "code": "SESSION_LIMIT_EXCEEDED",
    "message": "You have reached the maximum number of concurrent sessions (5)",
    "details": {
      "current": 5,
      "max": 5,
      "plan": "pro"
    },
    "request_id": "req_abc123"
  }
}
```

---

## 6. 速率限制

```yaml
Rate Limits:
  # 按用户
  session_creation: 10/hour
  messages: 60/minute
  secret_operations: 30/minute

  # 按全局
  total_active_sessions: capacity-based

  # 响应头
  X-RateLimit-Limit: 60
  X-RateLimit-Remaining: 45
  X-RateLimit-Reset: 1709424000
```
