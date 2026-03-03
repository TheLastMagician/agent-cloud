# 模块二十四：缓存与性能优化策略设计

> 本文档详细描述系统各层级的缓存策略、LLM 调用优化、启动延迟优化、资源池化等。

---

## 1. 多级缓存架构

```
┌─────────────────────────────────────────────────────────┐
│                    缓存层级                               │
│                                                         │
│  L1: 进程内缓存 (最快, 最小)                              │
│  ├── LRU Cache: 工具定义, 系统提示模板                    │
│  ├── TTL: 30 秒                                         │
│  └── 命中率目标: >95%                                    │
│                                                         │
│  L2: Redis 缓存 (快, 共享)                               │
│  ├── 会话状态, 用户配额, Secret 元数据                    │
│  ├── TTL: 5-60 分钟                                     │
│  └── 命中率目标: >80%                                    │
│                                                         │
│  L3: CDN 缓存 (快, 边缘)                                │
│  ├── 产物文件 (图片/视频/日志)                            │
│  ├── 静态资源 (前端 JS/CSS)                              │
│  └── TTL: 24 小时                                       │
│                                                         │
│  L4: 数据库 (慢, 持久)                                   │
│  ├── PostgreSQL 查询缓存                                 │
│  ├── 连接池 (PgBouncer)                                  │
│  └── 索引优化                                            │
└─────────────────────────────────────────────────────────┘
```

---

## 2. LLM 调用优化

### 2.1 Prompt 缓存 (最大优化点)

```
LLM API 调用中, 系统提示占大量输入 tokens:

未缓存:
  每次调用: System Prompt (20K tokens) + Context (变化)
  成本: 每次都付 20K 输入 tokens 的费用

有缓存 (Anthropic Prompt Caching):
  首次调用: 缓存 System Prompt (20K tokens)
  后续调用: 缓存命中, 只需传变化部分
  节省: ~90% 的系统提示 token 成本

实现:
  1. System Prompt 设为 cache_control: ephemeral
  2. 工具定义设为 cache_control: ephemeral
  3. 静态上下文 (AGENTS.md) 设为 cache_control: ephemeral
  4. 只有对话历史和用户消息每次都完整传送
```

```python
class PromptCacheOptimizer:
    """LLM Prompt 缓存优化"""

    def build_cached_request(self, context: List[Message]) -> dict:
        """构建带缓存控制的请求"""
        return {
            "model": "claude-4.6-opus",
            "system": [
                {
                    "type": "text",
                    "text": self.system_prompt,
                    "cache_control": {"type": "ephemeral"}
                }
            ],
            "tools": [
                {**tool, "cache_control": {"type": "ephemeral"}}
                for tool in self.tools[:4]  # 缓存前几个大工具定义
            ] + self.tools[4:],
            "messages": context,  # 不缓存 (每次变化)
        }
```

### 2.2 上下文窗口效率

```python
class ContextOptimizer:
    """上下文窗口效率优化"""

    # 1. 工具输出截断
    MAX_TOOL_OUTPUT_TOKENS = 10000

    def truncate_output(self, output: str) -> str:
        tokens = self.count_tokens(output)
        if tokens > self.MAX_TOOL_OUTPUT_TOKENS:
            truncated = self.truncate_to_tokens(output, self.MAX_TOOL_OUTPUT_TOKENS)
            return f"{truncated}\n\n[... output truncated ({tokens} total tokens) ...]"
        return output

    # 2. 文件读取优化
    def optimize_file_read(self, path: str, content: str) -> str:
        """大文件只返回相关部分"""
        if self.count_tokens(content) > 5000:
            # 建议使用 offset/limit
            return f"File is large ({self.count_tokens(content)} tokens). Consider using offset/limit."
        return content

    # 3. 重复文件读取去重
    def deduplicate_reads(self, messages: List[Message]) -> List[Message]:
        """去除重复的文件读取"""
        seen_files = {}
        for msg in messages:
            if msg.role == "tool" and msg.tool_name == "Read":
                file_path = msg.tool_input.get("path")
                if file_path in seen_files:
                    # 只保留最后一次读取
                    seen_files[file_path].content = "[Earlier read removed - see latest version below]"
                seen_files[file_path] = msg
        return messages
```

---

## 3. VM 启动延迟优化

### 3.1 预热池策略

```
冷启动路径 (无优化):
  创建 VM → 启动 OS → 安装依赖 → 克隆仓库 → 总计: 30-60秒

预热池路径 (有优化):
  从池中取出预热 VM → 克隆仓库 → 总计: 5-15秒

快照恢复路径 (最优):
  下载快照 → 恢复 VM → 拉取代码增量 → 总计: 10-30秒

预热池管理:
  ├── 池大小: 按需动态调整
  │   低峰: 5 个
  │   中峰: 15 个
  │   高峰: 30 个
  │
  ├── 补充策略: 取出一个立即补充一个
  │
  └── 预热内容:
      ├── OS 已启动
      ├── Docker daemon 已运行
      ├── Chrome 已安装
      ├── 常见运行时已安装 (Node 20, Python 3.12)
      └── 系统包已更新
```

### 3.2 快照增量优化

```python
class IncrementalSnapshot:
    """增量快照优化"""

    # 问题: 完整快照可能很大 (数 GB)
    # 解决: 使用 Copy-on-Write (CoW) 增量

    def save_incremental(self, base_snapshot_id: str, vm: VM) -> Snapshot:
        """保存增量快照"""
        base = self.get_snapshot(base_snapshot_id)

        # 只保存变化的磁盘块
        delta = vm.get_disk_delta(base.rootfs_checksum)

        # 内存总是全量保存 (无法增量)
        memory = vm.dump_memory()

        return Snapshot(
            base_id=base_snapshot_id,
            delta_key=self.upload(delta),      # 小: 只有变化
            memory_key=self.upload(memory),     # 大: 全量
            delta_size=len(delta),
        )

    # 典型大小:
    # 全量快照: 2-8 GB
    # 增量快照: 50-500 MB (仅变更)
    # 内存快照: 1-4 GB (始终全量)
```

---

## 4. 数据库性能优化

### 4.1 连接池

```python
# PgBouncer 配置
PGBOUNCER_CONFIG = {
    "pool_mode": "transaction",    # 事务级连接池
    "max_client_conn": 1000,       # 最大客户端连接
    "default_pool_size": 25,       # 每个数据库的连接池大小
    "reserve_pool_size": 5,        # 预留连接
    "reserve_pool_timeout": 3,     # 预留连接超时
}
```

### 4.2 热点查询优化

```sql
-- 1. 活跃会话查询 (最高频)
-- 优化: 部分索引
CREATE INDEX idx_sessions_active ON sessions(user_id, repo_id)
    WHERE status IN ('creating', 'initializing', 'running', 'waiting');

-- 2. 最新快照查询
-- 优化: 覆盖索引
CREATE INDEX idx_snapshots_latest ON snapshots(repo_id, branch, created_at DESC)
    WHERE status = 'active'
    INCLUDE (id, memory_key, rootfs_key, update_script);

-- 3. 消息历史查询
-- 优化: 按 session_id 分区 + 序号索引
CREATE INDEX idx_messages_history ON messages(session_id, sequence_num);

-- 4. 物化视图: 使用量统计
CREATE MATERIALIZED VIEW mv_daily_usage AS
SELECT
    user_id,
    date_trunc('day', created_at) as day,
    COUNT(*) as sessions,
    SUM(input_tokens) as input_tokens,
    SUM(output_tokens) as output_tokens,
    SUM(EXTRACT(EPOCH FROM (completed_at - started_at))) as total_seconds
FROM sessions
WHERE created_at >= NOW() - INTERVAL '90 days'
GROUP BY user_id, date_trunc('day', created_at);

-- 每小时刷新
-- REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_usage;
```

---

## 5. 网络优化

### 5.1 WebSocket 压缩

```python
# WebSocket permessage-deflate 压缩
# 减少 30-50% 的流量

WEBSOCKET_CONFIG = {
    "compression": "permessage-deflate",
    "compression_level": 6,
    "max_message_size": 1_000_000,  # 1MB
}
```

### 5.2 产物分发优化

```
产物文件分发:

1. Agent 生成产物 → 保存到 VM 本地
2. 会话结束 → 上传到 S3
3. 生成 CloudFront 签名 URL (24h 有效)
4. 客户端通过 CDN 边缘节点下载

优化:
├── 图片: WebP 格式 (比 PNG 小 30-50%)
├── 视频: H.264 CRF=28 (质量与大小平衡)
├── 日志: gzip 压缩
└── CDN: 全球边缘缓存
```

---

## 6. 性能基准

### 6.1 延迟目标

| 操作 | P50 | P99 | 目标 |
|------|-----|-----|------|
| 首字节响应 (流式) | 1s | 3s | < 5s |
| 工具执行 (Read/Write) | 20ms | 100ms | < 200ms |
| 工具执行 (Shell, 简单) | 200ms | 2s | < 5s |
| 工具执行 (Grep) | 50ms | 500ms | < 1s |
| 子代理启动 | 500ms | 2s | < 5s |
| VM 冷启动 | 15s | 45s | < 60s |
| VM 快照恢复 | 10s | 30s | < 60s |
| 产物上传 | 2s | 10s | < 30s |
| WebSocket 事件延迟 | 5ms | 50ms | < 100ms |

### 6.2 吞吐量目标

| 指标 | 目标 |
|------|------|
| 并发活跃会话 | 1000+ |
| WebSocket 并发连接 | 10000+ |
| API QPS | 5000+ |
| 事件吞吐量 | 50000 events/s |
