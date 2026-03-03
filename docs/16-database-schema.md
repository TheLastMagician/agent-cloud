# 模块十六：数据库 Schema 与数据模型设计

> 本文档详细描述系统所有持久化数据的表结构、索引策略、关系图、查询模式。

---

## 1. 数据库选型

| 用途 | 数据库 | 说明 |
|------|--------|------|
| 核心业务数据 | PostgreSQL | 会话、用户、仓库、Secret 元数据 |
| 实时事件流 | Redis Streams | WebSocket 事件分发、事件重放 |
| 快照/产物存储 | S3 (对象存储) | VM 快照、产物文件 |
| 缓存 | Redis | 速率限制、会话状态缓存 |
| 全文搜索 (可选) | Elasticsearch | 审计日志搜索 |

---

## 2. 核心表结构

### 2.1 ER 图

```
┌──────────┐     ┌──────────────┐     ┌─────────────┐
│  users   │──┬──│  sessions    │──┬──│  messages   │
└──────────┘  │  └──────────────┘  │  └─────────────┘
              │         │          │
              │         │          └──│  tool_calls  │
              │         │             └──────────────┘
              │         │
              │         ├──────────│  artifacts     │
              │         │          └────────────────┘
              │         │
              │         ├──────────│  commits       │
              │         │          └────────────────┘
              │         │
              │         └──────────│  snapshots     │
              │                    └────────────────┘
              │
              ├──────────────────│  secrets        │
              │                  └─────────────────┘
              │
              ├──────────────────│  repos          │
              │                  └─────────────────┘
              │
              └──────────────────│  audit_logs     │
                                 └─────────────────┘
```

### 2.2 users 表

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(255),
    avatar_url      TEXT,
    github_id       VARCHAR(100) UNIQUE,
    github_token    TEXT,                    -- 加密存储
    org_id          UUID REFERENCES orgs(id),
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
                    -- free | hobby | pro | enterprise
    settings        JSONB DEFAULT '{}',
                    -- 用户配置: 语言、规则等
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_github_id ON users(github_id);
CREATE INDEX idx_users_org_id ON users(org_id);
```

### 2.3 orgs 表

```sql
CREATE TABLE orgs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 2.4 repos 表

```sql
CREATE TABLE repos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    github_repo_id  BIGINT UNIQUE,
    full_name       VARCHAR(255) NOT NULL,  -- "owner/repo"
    default_branch  VARCHAR(100) DEFAULT 'main',
    is_public       BOOLEAN DEFAULT FALSE,
    clone_url       TEXT NOT NULL,
    owner_id        UUID REFERENCES users(id),
    org_id          UUID REFERENCES orgs(id),

    -- 环境配置
    update_script   TEXT,                    -- VM 启动更新脚本
    env_config      JSONB DEFAULT '{}',      -- .cursor/environment.json 的内容
    allow_secret_injection BOOLEAN DEFAULT TRUE,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_repos_github_id ON repos(github_repo_id);
CREATE INDEX idx_repos_full_name ON repos(full_name);
CREATE INDEX idx_repos_owner ON repos(owner_id);
```

### 2.5 sessions 表

```sql
CREATE TABLE sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    repo_id         UUID NOT NULL REFERENCES repos(id),
    branch          VARCHAR(255) NOT NULL,

    -- 状态
    status          VARCHAR(50) NOT NULL DEFAULT 'creating',
                    -- creating | initializing | running | waiting
                    -- completing | completed | failed | cancelled | timed_out

    -- VM 信息
    vm_id           VARCHAR(100),
    vm_host         VARCHAR(255),            -- 宿主机标识

    -- Agent 配置
    model           VARCHAR(100) DEFAULT 'claude-4.6-opus',
    task_type       VARCHAR(50) DEFAULT 'general',
                    -- general | env_setup | bug_fix | feature

    -- Git 状态
    git_commit_start VARCHAR(40),            -- 会话开始时的 commit
    git_commit_end   VARCHAR(40),            -- 会话结束时的 commit

    -- 使用量
    input_tokens    BIGINT DEFAULT 0,
    output_tokens   BIGINT DEFAULT 0,
    thinking_tokens BIGINT DEFAULT 0,
    tool_calls_count INTEGER DEFAULT 0,
    subagent_count  INTEGER DEFAULT 0,

    -- 时间
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    started_at      TIMESTAMPTZ,             -- Agent 开始执行时间
    completed_at    TIMESTAMPTZ,             -- 完成时间
    timeout_at      TIMESTAMPTZ,             -- 预期超时时间

    -- 快照
    snapshot_id     UUID REFERENCES snapshots(id),
    restored_from_snapshot_id UUID REFERENCES snapshots(id)
);

CREATE INDEX idx_sessions_user ON sessions(user_id);
CREATE INDEX idx_sessions_repo ON sessions(repo_id, branch);
CREATE INDEX idx_sessions_status ON sessions(status);
CREATE INDEX idx_sessions_active ON sessions(user_id, status)
    WHERE status IN ('creating', 'initializing', 'running', 'waiting');
CREATE INDEX idx_sessions_created ON sessions(created_at DESC);
```

### 2.6 messages 表

```sql
CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES sessions(id),
    role            VARCHAR(20) NOT NULL,    -- user | assistant | system | tool
    content         TEXT,
    thinking        TEXT,                    -- Agent 的思考内容

    -- 关联
    tool_call_id    VARCHAR(100),            -- tool 消息的关联
    parent_message_id UUID REFERENCES messages(id),

    -- 元数据
    input_tokens    INTEGER DEFAULT 0,
    output_tokens   INTEGER DEFAULT 0,
    model           VARCHAR(100),

    -- 时间
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    sequence_num    SERIAL                   -- 消息在会话中的顺序
);

CREATE INDEX idx_messages_session ON messages(session_id, sequence_num);
CREATE INDEX idx_messages_role ON messages(session_id, role);
```

### 2.7 tool_calls 表

```sql
CREATE TABLE tool_calls (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES sessions(id),
    message_id      UUID NOT NULL REFERENCES messages(id),
    tool_call_id    VARCHAR(100) NOT NULL,   -- LLM 生成的 tool_call ID

    -- 工具信息
    tool_name       VARCHAR(100) NOT NULL,
    input           JSONB NOT NULL,          -- 工具参数
    output          TEXT,                    -- 工具输出 (脱敏后)
    is_error        BOOLEAN DEFAULT FALSE,

    -- 执行信息
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
                    -- pending | running | completed | failed | timeout
    duration_ms     INTEGER,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tool_calls_session ON tool_calls(session_id);
CREATE INDEX idx_tool_calls_message ON tool_calls(message_id);
CREATE INDEX idx_tool_calls_name ON tool_calls(tool_name);
```

### 2.8 secrets 表

```sql
CREATE TABLE secrets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    encrypted_value BYTEA NOT NULL,          -- AES-256-GCM 加密
    type            VARCHAR(20) NOT NULL DEFAULT 'secret',
                    -- secret | redacted
    scope           VARCHAR(20) NOT NULL,    -- personal | team | repo
    owner_id        UUID NOT NULL,           -- user_id 或 org_id
    repo_id         UUID REFERENCES repos(id),
    kms_key_id      VARCHAR(255) NOT NULL,   -- KMS 密钥 ID

    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(name, scope, owner_id, repo_id)
);

CREATE INDEX idx_secrets_owner ON secrets(owner_id, scope);
CREATE INDEX idx_secrets_repo ON secrets(repo_id) WHERE repo_id IS NOT NULL;
```

### 2.9 artifacts 表

```sql
CREATE TABLE artifacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES sessions(id),
    filename        VARCHAR(500) NOT NULL,
    original_path   TEXT,                    -- VM 内路径
    mime_type       VARCHAR(100),
    size_bytes      BIGINT,

    -- 存储
    storage_key     TEXT NOT NULL,           -- S3 key
    cdn_url         TEXT,                    -- CDN URL

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_artifacts_session ON artifacts(session_id);
```

### 2.10 snapshots 表

```sql
CREATE TABLE snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID REFERENCES sessions(id),
    repo_id         UUID NOT NULL REFERENCES repos(id),
    branch          VARCHAR(255) NOT NULL,

    -- 快照数据
    memory_key      TEXT NOT NULL,           -- S3 key
    rootfs_key      TEXT NOT NULL,           -- S3 key
    metadata        JSONB NOT NULL,

    -- 配置快照
    update_script   TEXT,
    git_commit      VARCHAR(40),
    vm_config       JSONB,

    -- 状态
    status          VARCHAR(20) DEFAULT 'active',
                    -- active | expired | deleted
    size_bytes      BIGINT,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ             -- 过期时间
);

CREATE INDEX idx_snapshots_repo ON snapshots(repo_id, branch);
CREATE INDEX idx_snapshots_active ON snapshots(repo_id, branch, status)
    WHERE status = 'active';
```

### 2.11 commits 表

```sql
CREATE TABLE commits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES sessions(id),
    commit_hash     VARCHAR(40) NOT NULL,
    message         TEXT NOT NULL,
    files_changed   INTEGER,
    insertions      INTEGER,
    deletions       INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_commits_session ON commits(session_id);
```

### 2.12 audit_logs 表

```sql
CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type      VARCHAR(100) NOT NULL,
    user_id         UUID REFERENCES users(id),
    session_id      UUID REFERENCES sessions(id),
    resource_type   VARCHAR(50),             -- session | secret | repo | vm
    resource_id     UUID,
    details         JSONB DEFAULT '{}',
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_user ON audit_logs(user_id, created_at DESC);
CREATE INDEX idx_audit_session ON audit_logs(session_id);
CREATE INDEX idx_audit_type ON audit_logs(event_type, created_at DESC);
CREATE INDEX idx_audit_time ON audit_logs(created_at DESC);

-- 分区 (按月)
-- 用于大规模部署时的性能优化
```

---

## 3. Redis 数据结构

```python
# === 会话状态缓存 ===
# Key: session:{session_id}:status
# Type: Hash
HSET session:sess_xyz:status \
    status "running" \
    vm_id "vm_001" \
    last_heartbeat "1709420000" \
    input_tokens "45000" \
    output_tokens "12000"

# === WebSocket 事件流 ===
# Key: session:{session_id}:events
# Type: Stream
XADD session:sess_xyz:events * \
    type "text_delta" \
    payload '{"text":"Hello"}'

# === 速率限制 ===
# Key: ratelimit:{user_id}:{action}
# Type: String (with TTL)
# 使用令牌桶或滑动窗口算法
SET ratelimit:user_123:session_create 8 EX 3600

# === VM 池状态 ===
# Key: vm:warm_pool
# Type: List
LPUSH vm:warm_pool "vm_available_001" "vm_available_002"

# Key: vm:active
# Type: Hash
HSET vm:active vm_001 "sess_xyz"

# === 子代理状态 ===
# Key: subagent:{agent_id}:state
# Type: Hash (with TTL)
HSET subagent:agent_abc:state \
    type "debug" \
    session_id "sess_xyz" \
    history_key "subagent:agent_abc:history"

# Key: subagent:{agent_id}:history
# Type: List
RPUSH subagent:agent_abc:history \
    '{"role":"user","content":"..."}' \
    '{"role":"assistant","content":"..."}'
```

---

## 4. 查询模式

### 4.1 高频查询

```sql
-- 获取用户活跃会话
SELECT * FROM sessions
WHERE user_id = $1
  AND status IN ('creating', 'initializing', 'running', 'waiting')
ORDER BY created_at DESC;

-- 获取仓库最新快照
SELECT * FROM snapshots
WHERE repo_id = $1 AND branch = $2 AND status = 'active'
ORDER BY created_at DESC
LIMIT 1;

-- 解析 Secrets (优先级)
SELECT DISTINCT ON (name) name, encrypted_value, scope
FROM secrets
WHERE (owner_id = $1 AND scope = 'personal')
   OR (owner_id = $2 AND scope = 'team')
   OR (repo_id = $3 AND scope = 'repo')
ORDER BY name,
    CASE scope
        WHEN 'personal' THEN 1
        WHEN 'team' THEN 2
        WHEN 'repo' THEN 3
    END;

-- 获取会话完整消息历史
SELECT m.*, array_agg(tc.*) as tool_calls
FROM messages m
LEFT JOIN tool_calls tc ON tc.message_id = m.id
WHERE m.session_id = $1
GROUP BY m.id
ORDER BY m.sequence_num;
```

### 4.2 分析查询

```sql
-- 用户使用量统计
SELECT
    date_trunc('day', created_at) as day,
    COUNT(*) as session_count,
    SUM(input_tokens) as total_input_tokens,
    SUM(output_tokens) as total_output_tokens,
    AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) as avg_duration_secs
FROM sessions
WHERE user_id = $1
  AND created_at >= NOW() - INTERVAL '30 days'
GROUP BY day
ORDER BY day DESC;

-- 工具使用频率
SELECT
    tool_name,
    COUNT(*) as call_count,
    AVG(duration_ms) as avg_duration_ms,
    SUM(CASE WHEN is_error THEN 1 ELSE 0 END) as error_count
FROM tool_calls
WHERE session_id = $1
GROUP BY tool_name
ORDER BY call_count DESC;
```

---

## 5. 数据生命周期管理

| 数据类型 | 保留期 | 清理策略 |
|---------|--------|---------|
| 会话记录 | 永久 | 不删除, 归档 |
| 消息历史 | 90 天 | 定期归档到冷存储 |
| 工具调用 | 90 天 | 随消息归档 |
| 产物文件 | 30 天 | S3 生命周期策略 |
| VM 快照 | 30 天 | 定期清理旧快照 |
| 审计日志 | 365 天 | 分区+归档 |
| Secrets | 永久 | 用户手动删除 |
| Redis 缓存 | 按 TTL | 自动过期 |
