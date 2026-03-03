# 模块二十三：事件驱动架构与消息总线设计

> 本文档详细描述系统的事件流转机制、消息总线、事件溯源、CQRS 模式。

---

## 1. 事件驱动架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                   事件驱动架构                                │
│                                                             │
│  生产者 (Producers)           消息总线              消费者    │
│                                                             │
│  ┌──────────┐                                               │
│  │ Agent    │──▶ text_delta ──▶ ┌─────────┐ ──▶ WebSocket  │
│  │ Runtime  │──▶ tool_call ──▶  │  Redis   │ ──▶ Service   │
│  │          │──▶ todo_update─▶  │  Streams │                │
│  └──────────┘                   │          │ ──▶ Metrics    │
│                                 │  / NATS  │     Collector  │
│  ┌──────────┐                   │          │                │
│  │ VM       │──▶ vm_health ──▶  │  / Kafka │ ──▶ Audit     │
│  │ Manager  │──▶ vm_created──▶  │          │     Service   │
│  └──────────┘                   │          │                │
│                                 │          │ ──▶ Billing    │
│  ┌──────────┐                   │          │     Service   │
│  │ API      │──▶ session_new─▶  │          │                │
│  │ Service  │──▶ secret_op ──▶  └─────────┘ ──▶ Snapshot   │
│  └──────────┘                                    Service   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 事件分类

### 2.1 事件目录

```python
class EventCatalog:
    """完整事件目录"""

    # === Session 生命周期事件 ===
    SESSION_CREATED = "session.created"
    SESSION_INITIALIZING = "session.initializing"
    SESSION_STARTED = "session.started"
    SESSION_WAITING = "session.waiting"
    SESSION_RESUMED = "session.resumed"
    SESSION_COMPLETING = "session.completing"
    SESSION_COMPLETED = "session.completed"
    SESSION_FAILED = "session.failed"
    SESSION_CANCELLED = "session.cancelled"
    SESSION_TIMED_OUT = "session.timed_out"

    # === Agent 输出事件 (高频) ===
    AGENT_THINKING_START = "agent.thinking.start"
    AGENT_THINKING_DELTA = "agent.thinking.delta"
    AGENT_THINKING_END = "agent.thinking.end"
    AGENT_TEXT_DELTA = "agent.text.delta"
    AGENT_TEXT_END = "agent.text.end"

    # === 工具调用事件 ===
    TOOL_CALL_START = "tool.call.start"
    TOOL_CALL_INPUT_DELTA = "tool.call.input_delta"
    TOOL_CALL_EXECUTING = "tool.call.executing"
    TOOL_CALL_COMPLETED = "tool.call.completed"
    TOOL_CALL_FAILED = "tool.call.failed"

    # === 子代理事件 ===
    SUBAGENT_CREATED = "subagent.created"
    SUBAGENT_COMPLETED = "subagent.completed"
    SUBAGENT_FAILED = "subagent.failed"

    # === TODO 事件 ===
    TODO_UPDATED = "todo.updated"

    # === 产物事件 ===
    ARTIFACT_CREATED = "artifact.created"
    ARTIFACT_UPLOADED = "artifact.uploaded"

    # === VM 事件 ===
    VM_PROVISIONED = "vm.provisioned"
    VM_HEALTH_CHECK = "vm.health_check"
    VM_SNAPSHOT_SAVED = "vm.snapshot.saved"
    VM_SNAPSHOT_RESTORED = "vm.snapshot.restored"
    VM_TERMINATED = "vm.terminated"

    # === Git 事件 ===
    GIT_COMMIT = "git.commit"
    GIT_PUSH = "git.push"

    # === Secret 事件 ===
    SECRET_CREATED = "secret.created"
    SECRET_ACCESSED = "secret.accessed"
    SECRET_DELETED = "secret.deleted"

    # === 用户事件 ===
    USER_MESSAGE = "user.message"
    USER_SECRET_SUBMITTED = "user.secret.submitted"
    USER_ACTION_COMPLETED = "user.action.completed"
```

### 2.2 事件结构

```python
@dataclass
class Event:
    id: str                    # 全局唯一 ID
    type: str                  # 事件类型 (EventCatalog)
    source: str                # 来源服务
    session_id: Optional[str]  # 关联会话
    user_id: Optional[str]     # 关联用户
    timestamp: datetime        # 事件时间
    payload: dict             # 事件数据
    metadata: dict            # 元数据 (trace_id, span_id 等)

    # 排序保证
    sequence: int              # 会话内序号 (严格递增)
```

---

## 3. 消息总线实现

### 3.1 Redis Streams 方案

```python
class RedisEventBus:
    """基于 Redis Streams 的事件总线"""

    def publish(self, event: Event):
        """发布事件"""
        stream_key = f"events:{event.session_id}" if event.session_id \
                     else f"events:global:{event.type.split('.')[0]}"

        self.redis.xadd(
            stream_key,
            {
                "id": event.id,
                "type": event.type,
                "source": event.source,
                "session_id": event.session_id or "",
                "user_id": event.user_id or "",
                "payload": json.dumps(event.payload),
                "timestamp": event.timestamp.isoformat(),
            },
            maxlen=10000,  # 保留最近 10000 条
        )

    def subscribe(self, pattern: str, handler: Callable, group: str):
        """订阅事件 (消费者组)"""
        # 创建消费者组
        try:
            self.redis.xgroup_create(pattern, group, id="0", mkstream=True)
        except Exception:
            pass  # 组已存在

        while True:
            messages = self.redis.xreadgroup(
                group, consumer_id,
                {pattern: ">"},
                count=10,
                block=5000,
            )

            for stream, entries in messages:
                for entry_id, data in entries:
                    event = self.deserialize(data)
                    handler(event)
                    self.redis.xack(stream, group, entry_id)

    def replay(self, session_id: str, after_id: str) -> List[Event]:
        """重放事件 (用于 WebSocket 重连)"""
        stream_key = f"events:{session_id}"
        entries = self.redis.xrange(stream_key, min=after_id, max="+")
        return [self.deserialize(data) for _, data in entries]
```

### 3.2 事件路由表

```
┌────────────────────────────────────────────────────────────┐
│                     事件路由表                               │
│                                                            │
│  事件类型              │ 消费者                │ 处理方式   │
│ ──────────────────────┼──────────────────────┼──────────│
│  agent.text.delta     │ WebSocket Service    │ 实时转发   │
│  agent.thinking.*     │ WebSocket Service    │ 实时转发   │
│  tool.call.*          │ WebSocket Service    │ 实时转发   │
│  todo.updated         │ WebSocket Service    │ 实时转发   │
│  artifact.*           │ WebSocket Service    │ 实时转发   │
│                       │ Artifact Service     │ 触发上传   │
│  session.*            │ WebSocket Service    │ 状态通知   │
│                       │ Metrics Collector    │ 指标更新   │
│                       │ Billing Service      │ 计量记录   │
│  vm.*                 │ VM Manager           │ 池管理     │
│                       │ Snapshot Service     │ 快照触发   │
│  git.*                │ Audit Service        │ 审计记录   │
│  secret.*             │ Audit Service        │ 审计记录   │
│  user.*               │ Orchestration        │ 消息路由   │
└────────────────────────────────────────────────────────────┘
```

---

## 4. WebSocket 事件分发

```
Agent Runtime → Redis Stream → WebSocket Service → Client

详细流程:

1. Agent 产生 text_delta
2. Agent Runtime 发布到 Redis: XADD events:{session_id}
3. WebSocket Service 消费: XREADGROUP
4. 查找 session_id 对应的 WebSocket 连接
5. 通过 WebSocket 发送给客户端
6. ACK Redis 消息

重连场景:
1. 客户端断开 → 记录 last_event_id
2. 客户端重连 → 带上 last_event_id
3. WebSocket Service → Redis XRANGE 重放丢失事件
4. 客户端收到所有错过的事件
```

---

## 5. 事件持久化

```python
class EventStore:
    """事件持久化存储"""

    # 两级存储策略:
    # Level 1: Redis Streams (热数据, 最近 24h)
    # Level 2: PostgreSQL (冷数据, 永久)

    async def persist(self, event: Event):
        """持久化事件到数据库"""
        await self.db.execute("""
            INSERT INTO events (id, type, source, session_id, user_id,
                              payload, metadata, created_at, sequence)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        """, event.id, event.type, event.source, event.session_id,
             event.user_id, json.dumps(event.payload),
             json.dumps(event.metadata), event.timestamp, event.sequence)

    # 批量持久化 (提高写入性能)
    async def batch_persist(self, events: List[Event]):
        async with self.db.transaction():
            for event in events:
                await self.persist(event)
```

---

## 6. 事件驱动的副作用

```python
class EventHandlers:
    """事件触发的副作用"""

    @on_event("session.completed")
    async def handle_session_completed(self, event: Event):
        """会话完成后的处理"""
        session_id = event.session_id

        # 1. 上传产物
        await self.artifact_service.upload_all(session_id)

        # 2. 保存快照
        await self.snapshot_service.save(session_id)

        # 3. 记录计费
        await self.billing_service.finalize(session_id)

        # 4. 更新统计
        await self.metrics_service.update(session_id)

    @on_event("tool.call.completed")
    async def handle_tool_completed(self, event: Event):
        """工具调用完成后的处理"""
        # 更新实时使用量
        await self.billing_service.record_tool_call(
            session_id=event.session_id,
            tool_name=event.payload["tool_name"],
            duration_ms=event.payload["duration_ms"],
        )

    @on_event("secret.accessed")
    async def handle_secret_accessed(self, event: Event):
        """Secret 被访问时记录审计"""
        await self.audit_service.log(
            event_type="secret.access",
            user_id=event.user_id,
            details={"secret_name": event.payload["name"]},
        )

    @on_event("vm.health_check")
    async def handle_health_check(self, event: Event):
        """VM 健康检查"""
        if not event.payload["healthy"]:
            await self.vm_manager.handle_unhealthy(
                vm_id=event.payload["vm_id"],
                alerts=event.payload["alerts"],
            )
```
