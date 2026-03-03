# Cursor Cloud Agent 设计文档索引

> 基于 Cursor Cloud Agent 运行时逆向分析的完整产品设计文档。
> 适用于从零实现一个类似的云端 AI 编程代理产品。

---

## 文档结构

### 总览文档

| 文档 | 说明 | 行数(约) |
|------|------|---------|
| [cursor-cloud-agent-design.md](cursor-cloud-agent-design.md) | 系统总览设计（23章全景） | 1600 |

### 分模块详细设计 (第一批: 基础模块)

| 编号 | 文档 | 模块 | 关键内容 |
|------|------|------|---------|
| 01 | [01-infrastructure-layer.md](01-infrastructure-layer.md) | 基础设施层 | Firecracker VM、Docker-in-Docker、网络、存储、资源管理、高可用 |
| 02 | [02-agent-core-engine.md](02-agent-core-engine.md) | Agent 核心引擎 | LLM 调用编排、上下文构建、流式响应、工具执行循环、错误处理 |
| 03 | [03-tool-system.md](03-tool-system.md) | 工具系统 | 15 个工具的完整实现（Shell/Read/Write/Grep/Glob/Task 等） |
| 04 | [04-subagent-system.md](04-subagent-system.md) | 子代理系统 | 6 种子代理类型、调度器、并发控制、状态管理、通信协议 |
| 05 | [05-terminal-session-management.md](05-terminal-session-management.md) | 终端会话管理 | 终端即文件模型、会话状态同步、ANSI 处理、输出缓冲 |
| 06 | [06-secrets-environment-management.md](06-secrets-environment-management.md) | Secrets 与环境管理 | 加密机制、注入流程、作用域优先级、提交扫描、环境配置 |
| 07 | [07-artifacts-screen-recording.md](07-artifacts-screen-recording.md) | 产物与屏幕录制 | 产物生命周期、录制引擎、视频处理、上传管线、UI 渲染 |
| 08 | [08-ui-frontend-design.md](08-ui-frontend-design.md) | UI 与前端 | 组件架构、WebSocket 实时通信、状态管理、Desktop Pane、交互式任务 |
| 09 | [09-task-orchestration.md](09-task-orchestration.md) | 任务编排与调度 | 路由器、会话状态机、队列管理、快照恢复、事件流、计量计费 |
| 10 | [10-git-integration.md](10-git-integration.md) | Git 集成 | 操作权限、提交策略、分支管理、GitHub CLI、PR 自动化、冲突处理 |
| 11 | [11-testing-debugging-workflow.md](11-testing-debugging-workflow.md) | 测试与调试工作流 | 测试策略引擎、变更分析、证据收集、迭代控制、调试插桩 |
| 12 | [12-security-design.md](12-security-design.md) | 安全体系 | 威胁模型、沙箱隔离、输入/输出安全、权限控制、审计日志 |

### 分模块详细设计 (第二批: 深化模块)

| 编号 | 文档 | 模块 | 关键内容 |
|------|------|------|---------|
| 13 | [13-core-e2e-flows.md](13-core-e2e-flows.md) | 核心端到端流程 | 任务全生命周期时序(T0-T11)、跟进消息流程、工具调用时序、错误恢复4级、手动测试+录屏全流程、Bug 修复全流程、超时层级表 |
| 14 | [14-prompt-engineering.md](14-prompt-engineering.md) | Prompt 工程 | 系统提示词10层架构、User Message 注入结构、行为规则完整定义、Skill 系统、AGENTS.md 集成、系统提醒机制、上下文压缩策略 |
| 15 | [15-api-websocket-protocol.md](15-api-websocket-protocol.md) | API 与 WebSocket 协议 | 完整 REST API (5大资源)、WebSocket 20种事件类型、认证 JWT 流程、错误码体系、速率限制 |
| 16 | [16-database-schema.md](16-database-schema.md) | 数据库 Schema | 11张核心表 SQL DDL、索引策略、Redis 数据结构、ER 图、高频/分析查询模式、数据生命周期管理 |
| 17 | [17-deployment-operations.md](17-deployment-operations.md) | 部署与运维 | 生产架构图、10个微服务拆分、K8s 资源定义、监控栈(Prometheus+Loki+Jaeger)、告警规则、CI/CD 管线、灾备策略 |
| 18 | [18-env-setup-deep-dive.md](18-env-setup-deep-dive.md) | 环境设置深度 | 并行发现4任务编排、TODO生成逻辑、运行时版本管理、包管理器选择、Update Script 设计哲学、Hello World 任务完整设计、各类项目处理策略 |
| 19 | [19-computer-use-protocol.md](19-computer-use-protocol.md) | Computer Use 协议 | 操作原语完整定义、坐标系统、视觉感知循环、Chrome 管理、智能等待策略、主代理协作协议、错误恢复、录屏协调 |
| 20 | [20-billing-quota.md](20-billing-quota.md) | 计费与配额 | 成本构成分析、4级套餐设计、按量计费定价、配额检查流程、使用量追踪、仪表盘 API、成本优化策略 |

---

## 阅读建议

### 如果你是产品经理

建议阅读顺序：
1. `cursor-cloud-agent-design.md` — 理解全局架构
2. `08-ui-frontend-design.md` — 理解用户界面
3. `09-task-orchestration.md` — 理解任务流程

### 如果你是后端工程师

建议阅读顺序：
1. `01-infrastructure-layer.md` — 理解基础设施
2. `02-agent-core-engine.md` — 理解核心引擎
3. `04-subagent-system.md` — 理解子代理调度
4. `09-task-orchestration.md` — 理解编排层
5. `12-security-design.md` — 理解安全约束

### 如果你是前端工程师

建议阅读顺序：
1. `08-ui-frontend-design.md` — 组件架构和状态管理
2. `07-artifacts-screen-recording.md` — 产物渲染
3. `06-secrets-environment-management.md` — Secrets UI

### 如果你是 AI/ML 工程师

建议阅读顺序：
1. `02-agent-core-engine.md` — LLM 集成和上下文管理
2. `03-tool-system.md` — 工具定义和执行
3. `04-subagent-system.md` — 多代理协作
4. `11-testing-debugging-workflow.md` — Agent 测试策略

---

## 技术栈建议

| 层级 | 推荐技术 |
|------|---------|
| 虚拟化 | Firecracker MicroVM |
| 容器化 | Docker-in-Docker + fuse-overlayfs |
| 后端 API | Go / Rust / Node.js |
| 实时通信 | WebSocket |
| 前端 | React + Next.js + Tailwind |
| 状态管理 | Zustand / Jotai |
| LLM 集成 | Anthropic Claude API / OpenAI API |
| 对象存储 | AWS S3 / MinIO |
| 密钥管理 | AWS KMS / HashiCorp Vault |
| 消息队列 | Redis / SQS |
| 数据库 | PostgreSQL |
| 监控 | Prometheus + Grafana |
| 日志 | ELK / Loki |

---

*文档版本: 1.0*
*生成日期: 2026-03-03*
