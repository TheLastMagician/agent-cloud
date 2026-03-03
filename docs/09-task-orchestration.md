# 模块九：任务编排与调度详细设计

> 本文档详细描述任务从用户提交到 Agent 执行完成的完整编排流程，包括会话管理、队列调度、资源分配、状态机等。

---

## 1. 编排层架构

### 1.1 组件架构

```
┌──────────────────────────────────────────────────────────────┐
│                   Orchestration Layer                          │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ API Gateway  │  │ Task Router  │  │ Session Manager  │   │
│  │              │──▶              │──▶                   │   │
│  └──────────────┘  └──────────────┘  └────────┬─────────┘   │
│                                                │             │
│  ┌──────────────┐  ┌──────────────┐  ┌────────▼─────────┐   │
│  │ Queue        │  │ Snapshot     │  │ VM               │   │
│  │ Manager      │  │ Manager      │  │ Provisioner      │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ Rate         │  │ Billing      │  │ Metrics &        │   │
│  │ Limiter      │  │ Tracker      │  │ Logging          │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. 任务路由器（Task Router）

### 2.1 路由决策流程

```python
class TaskRouter:
    """任务路由器"""

    def route(self, request: TaskRequest) -> TaskRouteResult:
        """路由任务到合适的执行环境"""

        # 1. 认证与授权
        user = self.auth.verify(request.token)
        self.auth.check_permissions(user, request.repo)

        # 2. 限流检查
        if not self.rate_limiter.allow(user.id):
            raise RateLimitExceeded(user.id)

        # 3. 查找现有会话
        existing_session = self.session_manager.find_active(
            user_id=user.id,
            repo_id=request.repo_id,
            branch=request.branch,
        )

        if existing_session and request.is_followup:
            # 跟进消息: 复用现有会话
            return TaskRouteResult(
                action="RESUME",
                session=existing_session,
            )

        # 4. 查找可恢复快照
        snapshot = self.snapshot_manager.find_latest(
            repo_id=request.repo_id,
            branch=request.branch,
        )

        if snapshot:
            # 从快照恢复
            return TaskRouteResult(
                action="RESTORE",
                snapshot=snapshot,
            )

        # 5. 创建新会话
        return TaskRouteResult(
            action="CREATE",
            vm_config=self.resolve_vm_config(request),
        )
```

### 2.2 路由策略

| 场景 | 路由决策 | 说明 |
|------|---------|------|
| 新任务，无快照 | CREATE | 全新创建 VM |
| 新任务，有快照 | RESTORE | 从快照恢复，跳过环境设置 |
| 跟进消息 | RESUME | 复用活跃会话 |
| 会话超时后跟进 | RESTORE | 从最近快照恢复 |
| 并发会话上限 | QUEUE | 排队等待 |

---

## 3. 会话管理器（Session Manager）

### 3.1 会话数据模型

```python
@dataclass
class Session:
    """Agent 会话"""
    id: str
    user_id: str
    repo_id: str
    branch: str
    vm_id: str
    status: SessionStatus
    created_at: datetime
    updated_at: datetime
    timeout_at: datetime

    # 配置
    update_script: Optional[str] = None
    model: str = "claude-4.6-opus"

    # 运行时状态
    agent_pid: Optional[int] = None
    git_commit_at_start: Optional[str] = None
    git_commit_current: Optional[str] = None

    # 产物
    artifacts: List[ArtifactMeta] = field(default_factory=list)

    # 计量
    input_tokens: int = 0
    output_tokens: int = 0
    tool_calls_count: int = 0
    duration_seconds: int = 0


class SessionStatus(Enum):
    CREATING = "creating"          # VM 正在创建
    INITIALIZING = "initializing"  # 运行 update_script
    RUNNING = "running"            # Agent 正在执行
    WAITING = "waiting"            # 等待用户输入
    PAUSED = "paused"              # 已暂停
    COMPLETING = "completing"      # 正在完成（上传产物等）
    COMPLETED = "completed"        # 已完成
    FAILED = "failed"              # 失败
    CANCELLED = "cancelled"        # 已取消
    TIMED_OUT = "timed_out"        # 超时
```

### 3.2 会话状态机

```
                          ┌──────────┐
               创建请求 ──▶│ CREATING │
                          └────┬─────┘
                               │ VM ready
                               ▼
                          ┌──────────────┐
                          │ INITIALIZING │  运行 update_script
                          └────┬─────────┘
                               │ 初始化完成
                               ▼
              ┌───────────────────────────────────┐
              │                                   │
              ▼                                   │
         ┌─────────┐     跟进消息            ┌─────────┐
    ┌───▶│ RUNNING │◀────────────────────────│ WAITING │
    │    └────┬────┘                         └─────────┘
    │         │                                   ▲
    │         │ Agent 发送交互式任务请求             │
    │         └───────────────────────────────────┘
    │         │
    │         │ Agent 完成
    │         ▼
    │    ┌────────────┐
    │    │ COMPLETING  │  上传产物、保存快照
    │    └────┬───────┘
    │         │
    │         ▼
    │    ┌───────────┐
    │    │ COMPLETED  │
    │    └───────────┘
    │
    │    ┌───────────┐
    └────│ 新跟进     │  (从 COMPLETED 恢复)
         └───────────┘

  异常路径:
    任意状态 ──▶ FAILED (错误)
    任意状态 ──▶ CANCELLED (用户取消)
    任意状态 ──▶ TIMED_OUT (超时)
```

### 3.3 会话生命周期管理

```python
class SessionManager:
    """会话生命周期管理"""

    SESSION_TIMEOUT = 1800       # 30 分钟空闲超时
    MAX_SESSION_DURATION = 7200  # 2 小时最大时长
    MAX_CONCURRENT_SESSIONS = 5  # 每用户最大并发

    async def create_session(self, request: TaskRequest) -> Session:
        """创建新会话"""

        # 1. 并发检查
        active_count = await self.count_active(request.user_id)
        if active_count >= self.MAX_CONCURRENT_SESSIONS:
            raise ConcurrencyLimitError(request.user_id)

        # 2. 创建会话记录
        session = Session(
            id=generate_uuid(),
            user_id=request.user_id,
            repo_id=request.repo_id,
            branch=request.branch,
            status=SessionStatus.CREATING,
            created_at=datetime.utcnow(),
            timeout_at=datetime.utcnow() + timedelta(seconds=self.SESSION_TIMEOUT),
        )
        await self.store.save(session)

        # 3. 分配 VM
        vm = await self.vm_provisioner.provision(session)
        session.vm_id = vm.id
        session.status = SessionStatus.INITIALIZING

        # 4. 初始化环境
        await self.initialize_environment(session, vm)
        session.status = SessionStatus.RUNNING

        # 5. 启动 Agent
        await self.start_agent(session, vm, request)

        await self.store.save(session)
        return session

    async def initialize_environment(self, session: Session, vm: VM):
        """初始化 VM 环境"""

        # 克隆/更新仓库
        await vm.exec(f"cd /workspace && git fetch && git checkout {session.branch}")

        # 注入 Secrets
        secrets = await self.secret_resolver.resolve(
            user_id=session.user_id,
            repo_id=session.repo_id,
        )
        for name, value in secrets.items():
            await vm.set_env(name, value)

        # 运行更新脚本
        if session.update_script:
            result = await vm.exec(
                session.update_script,
                cwd="/workspace",
                timeout=300,  # 5 分钟
            )
            if result.exit_code != 0:
                raise InitializationError(f"Update script failed: {result.stderr}")

    async def handle_timeout(self, session_id: str):
        """处理会话超时"""
        session = await self.store.get(session_id)

        # 保存快照
        await self.snapshot_manager.save(session)

        # 标记超时
        session.status = SessionStatus.TIMED_OUT
        await self.store.save(session)

        # 释放 VM
        await self.vm_provisioner.release(session.vm_id)
```

---

## 4. 队列管理器（Queue Manager）

### 4.1 任务队列架构

```
┌─────────────┐     ┌─────────────────┐     ┌──────────────┐
│ Task Submit  │────▶│  Priority Queue  │────▶│  Worker Pool │
│ (API)       │     │  (Redis/SQS)    │     │  (VM Pool)   │
└─────────────┘     └─────────────────┘     └──────────────┘
```

### 4.2 优先级策略

```python
class TaskPriority:
    """任务优先级计算"""

    # 优先级因素权重
    WEIGHTS = {
        "plan_tier": 0.4,       # 付费计划等级
        "queue_time": 0.3,      # 等待时间（避免饥饿）
        "task_type": 0.2,       # 任务类型
        "concurrency": 0.1,     # 当前并发使用量
    }

    PLAN_TIERS = {
        "enterprise": 100,
        "pro": 70,
        "hobby": 40,
        "free": 10,
    }

    TASK_TYPES = {
        "env_setup": 90,        # 环境设置优先
        "bug_fix": 80,          # Bug 修复次之
        "feature": 60,          # 新功能
        "general": 50,          # 通用任务
    }

    def calculate(self, task: TaskRequest) -> float:
        score = 0
        score += self.PLAN_TIERS.get(task.plan, 10) * self.WEIGHTS["plan_tier"]
        score += min(task.queue_time_seconds / 300, 1.0) * 100 * self.WEIGHTS["queue_time"]
        score += self.TASK_TYPES.get(task.type, 50) * self.WEIGHTS["task_type"]
        concurrent_ratio = task.active_sessions / task.max_sessions
        score += (1 - concurrent_ratio) * 100 * self.WEIGHTS["concurrency"]
        return score
```

---

## 5. 会话恢复流程

### 5.1 跟进消息处理

```python
async def handle_followup(self, session_id: str, message: str):
    """处理跟进消息"""
    session = await self.store.get(session_id)

    match session.status:
        case SessionStatus.RUNNING:
            # 直接追加消息
            await self.agent_client.send_message(session.vm_id, message)

        case SessionStatus.WAITING:
            # Agent 在等待用户输入
            session.status = SessionStatus.RUNNING
            await self.agent_client.send_message(session.vm_id, message)

        case SessionStatus.COMPLETED | SessionStatus.TIMED_OUT:
            # 从快照恢复
            snapshot = await self.snapshot_manager.get_latest(session_id)
            if snapshot:
                new_session = await self.restore_from_snapshot(snapshot, message)
                return new_session
            else:
                # 无快照，创建新会话
                return await self.create_new_session(session, message)

        case SessionStatus.FAILED | SessionStatus.CANCELLED:
            # 创建新会话
            return await self.create_new_session(session, message)
```

---

## 6. 实时事件流

### 6.1 事件总线

```python
class EventBus:
    """会话事件总线"""

    def __init__(self):
        self.subscribers: Dict[str, List[Callable]] = {}

    async def publish(self, session_id: str, event: Event):
        """发布事件"""
        # 持久化事件
        await self.event_store.save(session_id, event)

        # 通知订阅者
        for callback in self.subscribers.get(session_id, []):
            await callback(event)

    def subscribe(self, session_id: str, callback: Callable):
        """订阅会话事件"""
        if session_id not in self.subscribers:
            self.subscribers[session_id] = []
        self.subscribers[session_id].append(callback)


class Event:
    TEXT_DELTA = "text_delta"
    THINKING_DELTA = "thinking_delta"
    TOOL_CALL_START = "tool_call_start"
    TOOL_CALL_INPUT = "tool_call_input"
    TOOL_CALL_RESULT = "tool_call_result"
    TODO_UPDATE = "todo_update"
    ARTIFACT_READY = "artifact_ready"
    SESSION_STATUS_CHANGE = "session_status_change"
    ERROR = "error"
```

### 6.2 Agent → 编排层 → UI 事件流

```
Agent Runtime                编排层                    UI
    │                          │                       │
    │  stdout: "Let me..."     │                       │
    │─────────────────────────▶│  text_delta           │
    │                          │──────────────────────▶│
    │                          │                       │
    │  tool_call: Shell(...)   │                       │
    │─────────────────────────▶│  tool_call_start      │
    │                          │──────────────────────▶│
    │                          │                       │
    │  tool_result: "..."      │                       │
    │─────────────────────────▶│  tool_call_result     │
    │                          │──────────────────────▶│
    │                          │                       │
    │  file: screenshot.png    │                       │
    │─────────────────────────▶│  artifact_ready       │
    │                          │──────────────────────▶│
    │                          │                       │
    │  [END]                   │                       │
    │─────────────────────────▶│  session_end          │
    │                          │──────────────────────▶│
```

---

## 7. 计量与计费

### 7.1 使用量追踪

```python
class UsageTracker:
    """使用量追踪"""

    def track(self, session: Session, event_type: str, data: dict):
        """记录使用量"""
        match event_type:
            case "llm_call":
                session.input_tokens += data["input_tokens"]
                session.output_tokens += data["output_tokens"]

            case "tool_call":
                session.tool_calls_count += 1

            case "vm_time":
                session.duration_seconds += data["seconds"]

    def get_usage_summary(self, user_id: str, period: str) -> UsageSummary:
        """获取使用量摘要"""
        sessions = self.store.get_sessions(user_id, period)
        return UsageSummary(
            total_sessions=len(sessions),
            total_input_tokens=sum(s.input_tokens for s in sessions),
            total_output_tokens=sum(s.output_tokens for s in sessions),
            total_tool_calls=sum(s.tool_calls_count for s in sessions),
            total_vm_minutes=sum(s.duration_seconds for s in sessions) / 60,
        )
```

---

## 8. 监控与告警

### 8.1 关键指标

| 指标 | 说明 | 告警阈值 |
|------|------|---------|
| session_create_latency | 会话创建延迟 | > 30s |
| vm_provision_latency | VM 分配延迟 | > 60s |
| snapshot_restore_latency | 快照恢复延迟 | > 120s |
| queue_depth | 等待队列深度 | > 50 |
| queue_wait_time | 队列等待时间 | > 300s |
| active_sessions | 活跃会话数 | > capacity * 0.9 |
| error_rate | 会话错误率 | > 5% |
| timeout_rate | 会话超时率 | > 20% |
