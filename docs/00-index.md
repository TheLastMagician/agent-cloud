# Cursor Cloud Agent 设计文档索引

> 基于 Cursor Cloud Agent 运行时逆向分析的完整产品设计文档。
> 适用于从零实现一个类似的云端 AI 编程代理产品。

---

## 文档结构

### 总览文档

| 文档 | 说明 | 行数(约) |
|------|------|---------|
| [cursor-cloud-agent-design.md](cursor-cloud-agent-design.md) | 系统总览设计（23章全景） | 1600 |

### 分模块详细设计

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
