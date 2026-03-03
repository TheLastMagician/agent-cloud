# 模块二十五：实现路线图（MVP → V1 → V2）

> 本文档提供从零到一实现 Cloud Agent 产品的分阶段路线图，
> 每个阶段的交付物、技术选型、团队配置、时间估计。

---

## 1. 阶段总览

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Phase 0        Phase 1         Phase 2        Phase 3          │
│  PoC            MVP             V1             V2               │
│  2-4 周         6-8 周          8-12 周        12-16 周         │
│                                                                 │
│  ┌──────┐      ┌──────┐       ┌──────┐       ┌──────┐         │
│  │ 验证  │ ──▶ │ 可用  │ ──▶  │ 可靠  │ ──▶  │ 规模  │         │
│  │ 概念  │      │ 产品  │       │ 产品  │       │ 产品  │         │
│  └──────┘      └──────┘       └──────┘       └──────┘         │
│                                                                 │
│  单用户          内部测试        公测           GA               │
│  本地运行        10用户          100用户        1000+用户        │
│  核心循环        基础功能        完整功能       企业功能          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Phase 0: PoC 概念验证 (2-4 周)

### 2.1 目标

验证核心技术可行性：LLM + 工具调用循环能否在隔离环境中自主完成编程任务。

### 2.2 交付物

```
✅ 最小 Agent 循环
   ├── LLM API 调用 (Claude/GPT)
   ├── 3 个基础工具: Shell, Read, Write
   ├── 工具调用解析与执行
   └── 多轮循环直到完成

✅ 最小隔离环境
   ├── Docker 容器 (非 VM)
   └── 工作目录挂载

✅ 最小交互
   ├── CLI 界面 (非 Web)
   └── 终端输入/输出

❌ 不做:
   ├── Web UI
   ├── 多用户
   ├── Firecracker VM
   ├── 快照
   ├── 子代理
   └── 产物系统
```

### 2.3 技术选型

| 组件 | 选择 | 原因 |
|------|------|------|
| Agent 运行时 | Python | 最快原型开发 |
| LLM | Claude API | 工具调用能力强 |
| 隔离 | Docker | PoC 阶段足够 |
| 交互 | CLI (stdin/stdout) | 最简单 |

### 2.4 示例代码骨架

```python
# poc/agent.py — 最小 Agent 循环

import anthropic

client = anthropic.Anthropic()
tools = [shell_tool, read_tool, write_tool]

def agent_loop(task: str):
    messages = [{"role": "user", "content": task}]

    while True:
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            system="You are a coding agent...",
            tools=tools,
            messages=messages,
        )

        # 处理文本输出
        for block in response.content:
            if block.type == "text":
                print(block.text)

        # 处理工具调用
        if response.stop_reason == "tool_use":
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result,
                    })
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": tool_results})
        else:
            break  # end_turn
```

---

## 3. Phase 1: MVP (6-8 周)

### 3.1 目标

可供内部团队日常使用的最小可用产品。

### 3.2 交付物清单

```
功能模块                     优先级    预计工时
────────────────────────────────────────────
Agent 核心引擎 (完整工具集)    P0       2 周
├── 15 个工具实现
├── 流式响应
├── 并行工具调用
└── 错误处理

Web UI (基础)                 P0       2 周
├── Chat Panel
├── 消息渲染 (Markdown)
├── 工具调用卡片
└── 输入框

WebSocket 实时通信             P0       1 周
├── 事件流
├── 重连机制
└── 心跳

VM 隔离 (Docker)              P0       1 周
├── Docker 容器管理
├── Git 仓库挂载
└── 环境变量注入

会话管理                      P0       1 周
├── 创建/查看/取消
├── 基础状态机
└── PostgreSQL 存储

认证                          P0       0.5 周
├── GitHub OAuth
└── JWT

Secret 管理 (基础)            P1       0.5 周
├── 创建/读取/删除
├── 环境变量注入
└── 输出脱敏

────────────────────────────────────────────
总计                                   ~8 周
```

### 3.3 不做 (推迟到 V1)

```
❌ Firecracker VM (用 Docker 代替)
❌ VM 快照与恢复
❌ 子代理系统
❌ Computer Use (GUI 测试)
❌ 屏幕录制
❌ Desktop Pane
❌ 多租户/团队
❌ 计费系统
❌ 产物系统 (产物直接在终端输出)
❌ 环境设置工作流
❌ TODO 列表
```

### 3.4 MVP 架构

```
浏览器 ←→ Next.js Frontend ←→ Node.js API ←→ Docker Container
                                   │              (Agent)
                                   │
                              PostgreSQL + Redis
```

---

## 4. Phase 2: V1 (8-12 周)

### 4.1 目标

面向公测的可靠产品，支持核心差异化功能。

### 4.2 交付物清单

```
功能模块                         优先级    预计工时
───────────────────────────────────────────────
Firecracker VM 迁移              P0       3 周
├── Firecracker 集成
├── Docker-in-Docker
├── fuse-overlayfs
└── iptables-legacy

VM 快照与恢复                    P0       2 周
├── 内存 + 磁盘快照
├── S3 存储
├── 恢复流程
└── Update Script 机制

子代理系统                       P0       2 周
├── explore 子代理
├── generalPurpose 子代理
├── 并发控制 (max 4)
└── 状态管理

Computer Use                    P0       2 周
├── computerUse 子代理
├── Chrome 自动化
├── VNC + noVNC
└── Desktop Pane UI

产物系统                         P1       1 周
├── 截图/视频/日志收集
├── S3 上传
├── CDN 分发
└── UI 展示

屏幕录制                         P1       1 周
├── ffmpeg 录制
├── 开始/保存/丢弃
└── videoReview 子代理

环境设置工作流                    P1       1 周
├── 并行发现
├── SetupVmEnvironment
├── AGENTS.md 生成
└── Hello World 验证

TODO 列表                        P2       0.5 周
debug 子代理                     P2       1 周
多租户基础                       P2       1 周
Secret Redacted 类型             P2       0.5 周

───────────────────────────────────────────────
总计                                     ~12 周 (部分并行)
```

---

## 5. Phase 3: V2 (12-16 周)

### 5.1 目标

企业级产品，支持大规模部署和高级功能。

### 5.2 交付物清单

```
功能模块                         优先级
───────────────────────────────────────
计费与配额系统                    P0
├── 套餐管理
├── 按量计费
├── 使用量仪表盘
└── Stripe 集成

多租户完整                       P0
├── 组织/团队管理
├── RBAC 权限
├── RLS 数据隔离
└── 团队 Secrets

企业功能                         P0
├── SSO (SAML/OIDC)
├── 审计日志
├── 合规报告
├── 专用 VM 池
└── SLA 保障

性能优化                         P1
├── Prompt 缓存
├── VM 预热池
├── 增量快照
├── 上下文压缩

高可用                           P1
├── 多区域部署
├── 自动故障转移
├── 灾备恢复

Skill 系统                       P2
MCP 集成                         P2
SDK / API                        P2
Webhook 集成                     P2
```

---

## 6. 团队配置建议

### 6.1 各阶段团队规模

| 阶段 | 人数 | 角色 |
|------|------|------|
| PoC | 1-2 | 全栈工程师 |
| MVP | 3-5 | 后端 2 + 前端 1 + 基础设施 1 + PM 1 |
| V1 | 6-10 | 后端 3 + 前端 2 + 基础设施 2 + AI/ML 1 + PM 1 + 设计 1 |
| V2 | 10-15 | 后端 4 + 前端 3 + 基础设施 3 + AI/ML 2 + PM 1 + 设计 1 + SRE 1 |

### 6.2 核心技能需求

| 角色 | 关键技能 |
|------|---------|
| 后端工程师 | Go/Rust/Node.js, 分布式系统, API 设计 |
| 前端工程师 | React/Next.js, WebSocket, 状态管理 |
| 基础设施 | Firecracker, Docker, Kubernetes, 网络 |
| AI/ML 工程师 | LLM API, Prompt 工程, Agent 架构 |
| SRE | 监控, 告警, 灾备, 性能调优 |

---

## 7. 技术风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| Firecracker 学习曲线高 | V1 延期 | Phase 0/1 用 Docker, 逐步迁移 |
| LLM API 成本不可控 | 毛利受压 | Prompt 缓存 + 上下文压缩 + 配额限制 |
| VM 快照恢复慢 | 用户体验差 | 增量快照 + 预热池 + 并行下载 |
| 子代理可靠性 | Agent 任务失败 | 重试机制 + 超时控制 + 降级策略 |
| Computer Use 精度 | GUI 测试失败 | 视觉模型优化 + 重试策略 + 等待优化 |
| 安全漏洞 | 用户数据泄露 | 三层隔离 + 安全审计 + 渗透测试 |
