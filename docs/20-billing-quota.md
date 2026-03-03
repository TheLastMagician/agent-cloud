# 模块二十：计费模型与配额管理设计

> 本文档详细描述系统的计费维度、配额管理、套餐设计、使用量追踪等。

---

## 1. 计费维度

### 1.1 成本构成分析

```
用户使用 Cloud Agent 的成本来源:

┌────────────────────────────────────────────┐
│              成本构成                        │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │  1. LLM API 调用 (最大成本, ~60-70%)  │  │
│  │     ├── 输入 tokens (context)         │  │
│  │     ├── 输出 tokens (response)        │  │
│  │     └── 思考 tokens (thinking)        │  │
│  └──────────────────────────────────────┘  │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │  2. 计算资源 (~20-25%)                │  │
│  │     ├── VM 运行时间 (CPU + Memory)    │  │
│  │     ├── 存储 (磁盘 + 快照)            │  │
│  │     └── 网络流量                      │  │
│  └──────────────────────────────────────┘  │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │  3. 对象存储 (~5-10%)                 │  │
│  │     ├── 快照存储                      │  │
│  │     ├── 产物存储                      │  │
│  │     └── CDN 分发                     │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

### 1.2 计量指标

| 计量维度 | 单位 | 精度 | 说明 |
|---------|------|------|------|
| 输入 tokens | tokens | 精确 | LLM 上下文 tokens |
| 输出 tokens | tokens | 精确 | LLM 生成 tokens |
| 思考 tokens | tokens | 精确 | 扩展思考 tokens |
| VM 时间 | 分钟 | 按分钟计 | VM 活跃运行时间 |
| 存储量 | GB | 按 GB 计 | 快照 + 产物 |
| 会话数 | 次 | 精确 | 每次 Agent 执行 |
| 工具调用数 | 次 | 精确 | 内部参考指标 |

---

## 2. 套餐设计

### 2.1 套餐层级

```
┌─────────────────────────────────────────────────────────────────┐
│                        套餐对比                                  │
│                                                                 │
│  功能/限制         │  Free    │  Hobby   │  Pro     │ Enterprise│
│ ──────────────────┼─────────┼─────────┼─────────┼───────────│
│  月用量 (tokens)   │  50K    │  500K   │  5M     │  无限      │
│  并发会话          │  1      │  2      │  5      │  20       │
│  最大会话时长      │  15min  │  30min  │  60min  │  120min   │
│  VM 规格          │  2C/4G  │  2C/4G  │  4C/8G  │  8C/16G   │
│  快照保留          │  7天    │  14天   │  30天   │  90天     │
│  产物保留          │  7天    │  14天   │  30天   │  90天     │
│  团队成员          │  1      │  5      │  20     │  无限      │
│  私有仓库          │  3      │  10     │  无限   │  无限      │
│  优先队列          │  ❌     │  ❌     │  ✅     │  ✅       │
│  专用 VM 池        │  ❌     │  ❌     │  ❌     │  ✅       │
│  SLA              │  ❌     │  ❌     │  99.5%  │  99.9%    │
│  支持              │  社区   │  邮件   │  优先   │  专属     │
│                                                                 │
│  月价格            │  $0     │  $20    │  $100   │  联系销售  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 按量付费 (超额)

```python
class UsagePricing:
    """按量付费定价"""

    PRICES = {
        "input_tokens": {
            "unit": "per 1M tokens",
            "price": {
                "free": None,     # 不允许超额
                "hobby": 3.00,    # $3/1M tokens
                "pro": 2.50,      # $2.5/1M tokens
                "enterprise": 2.00,  # 协商价
            },
        },
        "output_tokens": {
            "unit": "per 1M tokens",
            "price": {
                "free": None,
                "hobby": 15.00,
                "pro": 12.50,
                "enterprise": 10.00,
            },
        },
        "vm_minutes": {
            "unit": "per minute",
            "price": {
                "free": None,
                "hobby": 0.01,
                "pro": 0.008,
                "enterprise": 0.006,
            },
        },
        "storage_gb": {
            "unit": "per GB/month",
            "price": {
                "free": None,
                "hobby": 0.10,
                "pro": 0.08,
                "enterprise": 0.05,
            },
        },
    }
```

---

## 3. 配额管理

### 3.1 配额检查流程

```
任务提交
    │
    ▼
┌─────────────────┐
│ 1. 检查套餐状态  │  是否过期/取消?
└────────┬────────┘
         ▼
┌─────────────────┐
│ 2. 检查月用量    │  是否超过套餐限额?
└────────┬────────┘
         │
    ┌────┤
    │    │
  超额  未超额
    │    │
    ▼    ▼
┌──────┐ ┌─────────────┐
│允许  │ │ 允许执行     │
│超额? │ └─────────────┘
└──┬───┘
   │
  ┌┤
  ││
 是否
  ││
  ▼▼
继续 拒绝
(按量) (429)
```

### 3.2 实时配额追踪

```python
class QuotaManager:
    """配额管理器"""

    def check_quota(self, user_id: str) -> QuotaCheckResult:
        """检查用户配额"""
        user = self.get_user(user_id)
        plan = self.get_plan(user.plan)
        usage = self.get_current_usage(user_id)

        checks = []

        # 1. 并发会话
        active_sessions = self.count_active_sessions(user_id)
        if active_sessions >= plan.max_concurrent_sessions:
            checks.append(QuotaCheck(
                type="concurrent_sessions",
                allowed=False,
                current=active_sessions,
                limit=plan.max_concurrent_sessions,
            ))

        # 2. 月 token 用量
        monthly_tokens = usage.input_tokens + usage.output_tokens
        if monthly_tokens >= plan.monthly_token_limit:
            if plan.allows_overage:
                checks.append(QuotaCheck(
                    type="tokens",
                    allowed=True,
                    overage=True,
                    current=monthly_tokens,
                    limit=plan.monthly_token_limit,
                ))
            else:
                checks.append(QuotaCheck(
                    type="tokens",
                    allowed=False,
                    current=monthly_tokens,
                    limit=plan.monthly_token_limit,
                ))

        # 3. 存储配额
        storage_gb = self.get_storage_usage(user_id)
        if storage_gb >= plan.max_storage_gb:
            checks.append(QuotaCheck(
                type="storage",
                allowed=False,
                current=storage_gb,
                limit=plan.max_storage_gb,
            ))

        return QuotaCheckResult(checks=checks)

    def record_usage(self, session_id: str, usage: UsageRecord):
        """记录使用量 (实时)"""
        # Redis 原子递增
        key = f"usage:{self.user_id}:{current_month()}"
        self.redis.hincrby(key, "input_tokens", usage.input_tokens)
        self.redis.hincrby(key, "output_tokens", usage.output_tokens)
        self.redis.hincrby(key, "thinking_tokens", usage.thinking_tokens)
        self.redis.hincrby(key, "vm_minutes", usage.vm_minutes)
        self.redis.hincrby(key, "tool_calls", usage.tool_calls)

        # 异步写入 PostgreSQL (批量)
        self.usage_queue.push(usage)
```

---

## 4. 使用量仪表盘

### 4.1 用户可见指标

```
┌─────────────────────────────────────────┐
│           本月使用量                      │
│                                         │
│  Token 用量      ████████░░  3.2M / 5M  │
│  会话数          ████░░░░░░  42 次       │
│  VM 运行时间     ██████░░░░  280 / 500分 │
│  存储            ███░░░░░░░  12 / 50 GB  │
│                                         │
│  本月费用: $87.50                        │
│  ├── 套餐基础: $100.00                   │
│  └── 超额费用: $0.00                     │
│                                         │
│  [查看明细]  [升级套餐]                   │
└─────────────────────────────────────────┘
```

### 4.2 使用明细 API

```yaml
GET /api/v1/usage?period=2026-03
  Response:
    {
      "period": "2026-03",
      "plan": "pro",
      "summary": {
        "total_sessions": 42,
        "total_input_tokens": 2800000,
        "total_output_tokens": 420000,
        "total_thinking_tokens": 350000,
        "total_vm_minutes": 280,
        "total_storage_gb": 12.3,
        "total_cost_usd": 87.50
      },
      "daily_breakdown": [
        {
          "date": "2026-03-01",
          "sessions": 5,
          "input_tokens": 120000,
          "output_tokens": 18000,
          "vm_minutes": 25,
          "cost_usd": 8.50
        }
      ],
      "top_sessions": [
        {
          "session_id": "sess_xyz",
          "repo": "user/project",
          "input_tokens": 85000,
          "output_tokens": 12000,
          "vm_minutes": 15,
          "cost_usd": 5.20,
          "created_at": "2026-03-03T10:30:00Z"
        }
      ]
    }
```

---

## 5. 成本优化策略

### 5.1 平台侧优化

| 策略 | 节省比例 | 说明 |
|------|---------|------|
| Prompt 缓存 | ~30% tokens | 缓存相同的系统提示前缀 |
| VM 快照复用 | ~40% 启动时间 | 避免重复环境设置 |
| 工具输出截断 | ~15% tokens | 减少不必要的 token 消耗 |
| 上下文压缩 | ~20% tokens | 摘要旧轮次 |
| VM 预热池 | ~60% 启动延迟 | 减少冷启动 |
| 智能路由 | ~10% LLM成本 | 简单任务用更快的模型 |

### 5.2 用户侧建议

```
帮助用户降低成本:

1. 使用 AGENTS.md: 减少 Agent 探索时间
2. 提供清晰的任务描述: 减少来回沟通
3. 合理设置 update_script: 加速 VM 启动
4. 及时清理旧快照: 减少存储费用
5. 使用 Skills: 标准化操作流程
```
