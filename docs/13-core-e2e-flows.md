# 模块十三：核心端到端流程详细设计

> 本文档以极致细粒度描述系统中所有核心流程的完整时序、状态流转、异常分支与恢复策略。
> 每个流程都包含：准确的时序图、每一步的输入/输出、错误处理、超时策略。

---

## 1. 任务全生命周期流程

### 1.1 完整时序（从用户点击发送到收到最终回复）

```
时间线 ──────────────────────────────────────────────────────────────────▶

[T0] 用户在 Web UI 输入任务文本, 点击「发送」
  │
  ▼
[T1] 前端 WebSocket 发送 USER_MESSAGE 事件
  │   payload: { sessionId?, content, attachments?, repoId, branch }
  │
  ▼
[T2] API Gateway 接收
  │   ├── 认证: 验证 JWT Token
  │   ├── 限流: 检查用户速率限制 (令牌桶算法)
  │   └── 参数校验: content 长度, 附件大小
  │   ⚠ 失败 → 返回 4xx 错误, 前端展示
  │
  ▼
[T3] Task Router 路由决策
  │   ├── 查询活跃会话表: SELECT * FROM sessions
  │   │     WHERE user_id=? AND repo_id=? AND branch=?
  │   │     AND status IN ('running','waiting')
  │   │
  │   ├── CASE A: 找到活跃会话 (跟进消息)
  │   │     → 直接转发消息到现有 VM
  │   │     → 跳到 [T7]
  │   │
  │   ├── CASE B: 无活跃会话, 但有快照
  │   │     → 从对象存储下载快照
  │   │     → 跳到 [T4-RESTORE]
  │   │
  │   └── CASE C: 完全新建
  │         → 跳到 [T4-CREATE]
  │
  ▼
[T4-CREATE] VM 分配 (新建路径, ~5-15秒)
  │   ├── 从 Warm Pool 获取预热 VM (优先)
  │   │     └── 如果 Warm Pool 为空 → 创建新 Firecracker VM
  │   │         ├── 分配 TAP 网络设备
  │   │         ├── 挂载 rootfs 镜像
  │   │         ├── 配置 CPU/内存限制
  │   │         └── 启动 VM (Firecracker API)
  │   │
  │   ├── 克隆仓库到 /workspace
  │   │     git clone --branch {branch} {repo_url} /workspace
  │   │
  │   ├── 注入 Secrets 为环境变量
  │   │     ├── 解析作用域优先级 (Personal > Team)
  │   │     ├── 从 KMS 解密
  │   │     └── 设置环境变量
  │   │
  │   └── 运行 update_script (如果存在)
  │         ├── 命令: bash -c "{update_script}"
  │         ├── 工作目录: /workspace
  │         ├── 超时: 300秒
  │         └── ⚠ 失败 → 记录警告但继续 (尽力策略)
  │
  ▼
[T4-RESTORE] VM 恢复 (快照路径, ~10-30秒)
  │   ├── 从 S3 下载快照
  │   │     ├── memory.snapshot (~2-8GB)
  │   │     ├── rootfs.delta (~500MB-5GB)
  │   │     └── metadata.json
  │   │
  │   ├── 创建 VM 并应用快照
  │   │     ├── 恢复磁盘状态
  │   │     └── 恢复内存状态
  │   │
  │   ├── 拉取最新代码
  │   │     cd /workspace && git fetch origin && git checkout {branch} && git pull
  │   │
  │   ├── 运行 update_script
  │   │     └── (同 T4-CREATE)
  │   │
  │   └── 注入/更新 Secrets
  │
  ▼
[T5] 启动 Agent 运行时
  │   ├── 构建 System Prompt (见模块14)
  │   ├── 构建 User Message (注入上下文)
  │   │     ├── <user_info> (OS, Shell, 日期, 路径)
  │   │     ├── <git_status> (分支, 状态快照)
  │   │     ├── <rules> (用户规则)
  │   │     ├── AGENTS.md 内容 (如果存在)
  │   │     ├── Skills 列表 (如果配置)
  │   │     └── <user_query> (实际任务)
  │   │
  │   ├── 初始化工具注册表 (15个工具)
  │   └── 初始化会话状态
  │
  ▼
[T6] 前端收到 SESSION_READY 事件, 显示 Agent 正在思考
  │
  ▼
[T7] ═══════════════ Agent 主循环开始 ═══════════════
  │
  │   ┌─────────────────────────────────────────────┐
  │   │              Agent Turn N                    │
  │   │                                             │
  │   │  [T7.1] LLM API 调用                        │
  │   │    ├── 发送: messages[] + tools[]            │
  │   │    ├── 模型: claude-4.6-opus-high-thinking   │
  │   │    ├── temperature: 0                        │
  │   │    ├── max_tokens: 16000                     │
  │   │    └── thinking.budget_tokens: 10000         │
  │   │                                             │
  │   │  [T7.2] 流式响应处理                         │
  │   │    ├── thinking block                        │
  │   │    │     → 内部推理 (不展示给用户)             │
  │   │    │     → 前端显示 "正在思考..." 指示器       │
  │   │    │                                         │
  │   │    ├── text block                            │
  │   │    │     → 逐 token 发送 TEXT_DELTA 事件      │
  │   │    │     → 前端实时渲染 Markdown              │
  │   │    │                                         │
  │   │    └── tool_use blocks (可能多个, 并行)       │
  │   │          ├── TOOL_CALL_START 事件              │
  │   │          │     → 前端显示工具调用卡片           │
  │   │          ├── 参数增量传输                      │
  │   │          └── 参数完成 → 进入执行               │
  │   │                                             │
  │   │  [T7.3] 工具执行 (如果有 tool_use)            │
  │   │    ├── 并行执行独立的工具调用                  │
  │   │    │     ├── Tool A ───────▶ Result A        │
  │   │    │     ├── Tool B ───────▶ Result B        │
  │   │    │     └── Tool C ───────▶ Result C        │
  │   │    │                                         │
  │   │    ├── 对每个结果:                            │
  │   │    │     ├── Secret 脱敏 (值 → [REDACTED])   │
  │   │    │     ├── 输出截断 (如果过长)              │
  │   │    │     └── TOOL_CALL_RESULT 事件发送        │
  │   │    │                                         │
  │   │    └── 追加 tool results 到 messages[]       │
  │   │                                             │
  │   │  [T7.4] 决策: 是否继续?                      │
  │   │    ├── 有 tool_use → 继续 (回到 T7.1)        │
  │   │    ├── end_turn 标记 → 结束循环               │
  │   │    └── 无 tool_use 且有文本 → 结束循环        │
  │   │                                             │
  │   └─────────────────────────────────────────────┘
  │        ↑                              │
  │        │    循环 (通常 5-50 轮)         │
  │        └──────────────────────────────┘
  │
  ▼
[T8] Agent 主循环结束
  │   ├── 最终文本回复已生成
  │   ├── 包含: Summary, Testing, Walkthrough
  │   └── 可能包含: env_setup_actions XML (需要用户操作)
  │
  ▼
[T9] 会话完成处理
  │   ├── 上传产物到 S3
  │   │     扫描 /opt/cursor/artifacts/ 中的所有文件
  │   │     并行上传, 生成 CDN URL
  │   │
  │   ├── 保存 VM 快照 (异步)
  │   │     ├── 暂停 VM
  │   │     ├── 保存内存 + 磁盘增量
  │   │     └── 上传到 S3
  │   │
  │   ├── 更新会话状态
  │   │     status = COMPLETED
  │   │     记录: token 消耗, 工具调用次数, 时长
  │   │
  │   └── 发送 SESSION_END 事件
  │
  ▼
[T10] 前端渲染最终回复
  │   ├── Markdown 文本渲染
  │   ├── 产物 (图片/视频/日志) 内联展示
  │   ├── env_setup_actions → 交互式任务 UI
  │   └── TODO 列表最终状态
  │
  ▼
[T11] 用户查看结果
      ├── 满意 → 流程结束
      └── 需要调整 → 发送跟进消息 → 回到 [T1]
```

---

## 2. 跟进消息流程

### 2.1 活跃会话中的跟进

```
前提: 会话 status = WAITING (Agent 发送了 env_setup_actions)
     或 status = COMPLETED (Agent 已完成, VM 仍活跃)

用户发送跟进消息
    │
    ▼
[F1] 前端发送 USER_MESSAGE
    │   payload: { sessionId: "existing", content: "请修改..." }
    │
    ▼
[F2] Task Router: 发现活跃会话
    │   → action = RESUME
    │
    ▼
[F3] 会话状态更新
    │   status: WAITING → RUNNING
    │   或
    │   status: COMPLETED → RUNNING
    │
    ▼
[F4] 消息注入到 Agent
    │   ├── 新消息追加到 conversation_history
    │   ├── 刷新动态上下文 (git status 可能已变)
    │   └── 重新触发 Agent 主循环
    │
    ▼
[F5] Agent 主循环 (同 T7)
    │   Agent 拥有完整的历史上下文
    │   知道之前做了什么, 用户现在要什么
    │
    ▼
[F6] 完成 → 同 T8-T10
```

### 2.2 过期会话的跟进

```
前提: 会话已超时或完成, VM 已销毁, 但有快照

用户发送跟进消息
    │
    ▼
[E1] Task Router: 无活跃会话, 但有快照
    │   → action = RESTORE
    │
    ▼
[E2] 从快照恢复 VM (同 T4-RESTORE)
    │
    ▼
[E3] 重建 Agent 上下文
    │   ├── 恢复完整的 conversation_history
    │   ├── 追加新的用户消息
    │   └── 刷新动态上下文
    │
    ▼
[E4] Agent 主循环继续
```

---

## 3. 工具调用详细流程

### 3.1 单轮工具调用时序

```
LLM 生成响应
    │
    ├── text: "让我先查看项目结构。"
    │     → TEXT_DELTA 事件流 → 前端实时渲染
    │
    ├── tool_use[0]: Glob(pattern="package.json")
    │     → TOOL_CALL_START { id: "tc_01", name: "Glob" }
    │
    ├── tool_use[1]: Read(path="/workspace/README.md")
    │     → TOOL_CALL_START { id: "tc_02", name: "Read" }
    │
    └── tool_use[2]: Shell(command="git status")
          → TOOL_CALL_START { id: "tc_03", name: "Shell" }

工具执行引擎
    │
    ├── 依赖分析: 三个调用互相独立 → 并行执行
    │
    ├── [并行] Glob 执行 (~50ms)
    │     ├── 展开模式: **/package.json
    │     ├── 文件系统搜索
    │     ├── 按修改时间排序
    │     └── 输出: "Result of search: 3 files found\n..."
    │
    ├── [并行] Read 执行 (~20ms)
    │     ├── 检查文件存在
    │     ├── 检测文件类型 (文本)
    │     ├── 读取内容
    │     ├── 添加行号: "  1|# Project\n  2|..."
    │     └── 输出: 带行号的文件内容
    │
    └── [并行] Shell 执行 (~500ms)
          ├── 获取当前会话
          ├── 在 bash 中执行
          ├── 捕获 stdout+stderr
          ├── 格式化: "Exit code: 0\n\nCommand output:\n```\n..."
          └── 输出: 格式化的命令结果

工具结果组装
    │
    ├── 对每个结果执行 Secret 脱敏
    ├── 对每个结果执行截断检查
    │
    ├── TOOL_CALL_RESULT { id: "tc_01", output: "..." }
    ├── TOOL_CALL_RESULT { id: "tc_02", output: "..." }
    └── TOOL_CALL_RESULT { id: "tc_03", output: "..." }

追加到 messages 数组, 进入下一轮 LLM 调用
```

### 3.2 嵌套工具调用 (Task 启动子代理)

```
主 Agent LLM 生成
    │
    └── tool_use: Task(subagent_type="explore", prompt="...")
          │
          ▼
        Task 工具执行
          │
          ├── [T-1] 并发检查: active_subagents < 4?
          │     ├── 是 → 继续
          │     └── 否 → 等待信号量
          │
          ├── [T-2] 自动恢复检查 (debug/computerUse)
          │     ├── 找到最近实例 → resume
          │     └── 无 → 创建新的
          │
          ├── [T-3] 创建子代理进程
          │     ├── 分配 UUID
          │     ├── 配置工具集 (按类型限制)
          │     ├── 配置系统提示 (子代理专用)
          │     └── 传入 prompt (必须自包含所有上下文)
          │
          ├── [T-4] 子代理自主执行
          │     │
          │     │  ┌──── 子代理内部循环 ────┐
          │     │  │                        │
          │     │  │ LLM 调用 (继承父模型)   │
          │     │  │     ↓                  │
          │     │  │ 工具调用 (受限工具集)    │
          │     │  │     ↓                  │
          │     │  │ 结果处理               │
          │     │  │     ↓                  │
          │     │  │ 继续/完成?             │
          │     │  │                        │
          │     │  └────────────────────────┘
          │     │
          │     └── 子代理完成, 返回最终文本
          │
          ├── [T-5] 保存状态 (有状态子代理)
          │
          └── [T-6] 返回结果给主 Agent
                    "Agent ID: xxx\n\n{子代理返回的文本}"
```

---

## 4. 错误恢复流程

### 4.1 LLM 调用失败

```
LLM API 调用
    │
    ├── 成功 (200) → 正常处理
    │
    ├── 429 Rate Limited
    │     ├── 重试 1: 等待 1秒
    │     ├── 重试 2: 等待 5秒
    │     ├── 重试 3: 等待 15秒
    │     └── 超过 3 次 → 会话标记 FAILED, 通知用户
    │
    ├── 400 Context Too Long
    │     ├── 策略 1: 截断工具输出 (最长的优先)
    │     ├── 策略 2: 摘要早期对话轮次
    │     ├── 策略 3: 移除重复的文件读取
    │     └── 重试 LLM 调用
    │
    ├── 500 Server Error
    │     ├── 重试 (同 429 策略)
    │     └── 超过重试 → FAILED
    │
    └── 网络超时
          ├── 重试 (同 429 策略)
          └── 超过重试 → FAILED
```

### 4.2 工具执行失败

```
工具执行
    │
    ├── 文件不存在 (Read/StrReplace)
    │     → 返回错误消息给 LLM
    │     → LLM 自行决定下一步 (通常: 搜索正确路径)
    │
    ├── 权限拒绝
    │     → 返回错误消息给 LLM
    │     → LLM 自行决定下一步
    │
    ├── 命令超时 (Shell, 默认30s)
    │     → 杀死进程
    │     → 返回 "Command timed out" 给 LLM
    │     → LLM 可以: 调整命令/增加超时/改用后台模式
    │
    ├── StrReplace 非唯一匹配
    │     → 返回 "old_string appears N times"
    │     → LLM 需要: 提供更多上下文使其唯一
    │
    ├── 子代理崩溃
    │     → 返回错误消息
    │     → 主 Agent 可以: 重试/换方案
    │
    └── 所有情况: 错误消息作为 tool_result 反馈给 LLM
         LLM 基于错误信息自主调整策略
         (这是 Agent 自主性的核心: 遇到错误不停止, 而是适应)
```

### 4.3 VM 级别故障

```
VM 故障
    │
    ├── OOM Kill (进程被杀)
    │     ├── 健康监控检测到进程消失
    │     ├── 尝试重启 Agent 进程
    │     ├── 如果反复 OOM → 增加内存配额 → 重启
    │     └── 如果无法恢复 → 从快照恢复
    │
    ├── 磁盘满
    │     ├── 清理: /tmp, Docker 缓存, 旧日志
    │     ├── 如果仍满 → 扩展磁盘
    │     └── 通知 Agent 磁盘空间不足
    │
    ├── VM 进程崩溃
    │     ├── 自动从最近快照恢复
    │     ├── 重新注入上下文
    │     └── 继续执行
    │
    └── 宿主机故障
          ├── 集群健康检查发现节点不可用
          ├── 在新节点上从快照恢复
          └── 会话状态迁移
```

---

## 5. 手动测试 + 录屏完整流程

### 5.1 GUI 测试端到端流程

```
Agent 判断需要手动 GUI 测试
    │
    ▼
[M1] 确保开发服务器运行
    │   ├── Shell: "npm run dev" (后台)
    │   ├── 等待端口可用
    │   └── 确认服务器响应
    │
    ▼
[M2] 通过 computerUse 子代理导航到测试页面
    │   Task(subagent_type="computerUse",
    │        prompt="打开浏览器, 导航到 localhost:3000/login")
    │
    │   子代理执行:
    │   ├── 打开 Chrome
    │   ├── 导航到 URL
    │   ├── 等待页面加载
    │   ├── 截屏确认
    │   └── 返回截图路径
    │
    ▼
[M3] 开始屏幕录制
    │   RecordScreen(mode="START_RECORDING")
    │
    ▼
[M4] 通过 computerUse 子代理执行测试操作
    │   Task(subagent_type="computerUse",
    │        prompt="在页面上:
    │                1. 点击用户名输入框, 输入 test@example.com
    │                2. 点击密码输入框, 输入 password123
    │                3. 点击登录按钮
    │                4. 等待跳转到仪表盘
    │                5. 截屏确认")
    │
    │   子代理执行:
    │   ├── screenshot() → 分析页面布局
    │   ├── click(x, y) → 点击用户名框
    │   ├── type("test@example.com")
    │   ├── click(x, y) → 点击密码框
    │   ├── type("password123")
    │   ├── click(x, y) → 点击登录按钮
    │   ├── 等待 3 秒
    │   ├── screenshot() → 确认跳转成功
    │   └── 返回截图路径 + 描述
    │
    ▼
[M5] 停止录制并判断结果
    │
    ├── 测试成功?
    │   ├── 是 → RecordScreen(mode="SAVE_RECORDING",
    │   │         save_as_filename="demo_login_flow")
    │   │         → 保存到 /opt/cursor/artifacts/demo_login_flow.mp4
    │   │
    │   └── 否 → RecordScreen(mode="DISCARD_RECORDING")
    │             → 丢弃录屏
    │             → 调查失败原因
    │             → 修复问题
    │             → 回到 [M3] 重试
    │
    ▼
[M6] (可选) videoReview 子代理验证视频内容
    │   Task(subagent_type="videoReview",
    │        prompt="验证视频展示了完整的登录流程...",
    │        attachments=["/.../demo_login_flow_demo.mp4"])
    │
    ▼
[M7] 在最终回复中引用视频和截图
      <video src="/opt/cursor/artifacts/demo_login_flow.mp4" controls></video>
      <img src="/opt/cursor/artifacts/dashboard_after_login.webp" alt="登录后仪表盘" />
```

---

## 6. Bug 修复完整流程

```
用户报告 Bug: "点击提交按钮时页面崩溃"
    │
    ▼
[B1] Agent 分析 Bug 描述
    │   ├── 理解: 什么触发, 预期行为, 实际行为
    │   ├── 搜索相关代码 (Grep/Glob)
    │   └── 评估根因复杂度
    │
    ▼
[B2] 根因明显? → 判断分支
    │
    ├── [明显] 直接修复
    │     ├── 修改代码
    │     ├── 跳到 [B5]
    │     └── (简单 Bug 路径)
    │
    └── [不明显] 启动 Debug 子代理
          │
          ▼
        [B3] Debug 子代理: 迭代 1
          │   ├── 分析代码, 提出假设
          │   │     假设 1: onClick 处理器中的空引用 (70%)
          │   │     假设 2: 表单验证状态不一致 (20%)
          │   │     假设 3: API 响应处理错误 (10%)
          │   │
          │   ├── 插桩代码
          │   │     ├── SubmitButton.tsx:42: console.log("[DBG] form state:", formState)
          │   │     ├── SubmitButton.tsx:55: console.log("[DBG] onClick, data:", data)
          │   │     └── api.ts:120: console.log("[DBG] submit response:", response)
          │   │
          │   └── 返回复现步骤
          │         "1. 打开 /form 页面
          │          2. 不填写必填字段
          │          3. 点击提交按钮
          │          4. 查看控制台日志"
          │
          ▼
        [B3.5] Agent 复现 Bug (通过 computerUse 或 Shell)
          │   ├── 启动开发服务器
          │   ├── 按步骤操作
          │   ├── 收集控制台日志
          │   └── 反馈给 Debug 子代理: "Issue reproduced. 控制台显示:
          │         [DBG] form state: {valid: false, fields: null}
          │         [DBG] onClick, data: null
          │         Uncaught TypeError: Cannot read property 'email' of null"
          │
          ▼
        [B4] Debug 子代理: 分析日志
          │   ├── 假设 1 确认: data 为 null 时直接访问属性
          │   ├── 根因: handleSubmit 没有检查 formState.valid
          │   │
          │   ├── 修复方案:
          │   │     在 handleSubmit 开头添加:
          │   │     if (!formState.valid || !data) return;
          │   │
          │   └── 返回修复 + 清理指令
          │
          ▼
[B5] Agent 实施修复
    │   ├── StrReplace 修改代码
    │   ├── 运行 lint (eslint)
    │   └── 修复 lint 错误 (如果有)
    │
    ▼
[B6] 验证修复: 修复前 vs 修复后
    │
    │   [修复前复现 — 已在 B3.5 完成]
    │
    │   [修复后验证]
    │   ├── 开始录屏
    │   ├── 重复相同步骤
    │   ├── 确认: 页面不再崩溃
    │   ├── 确认: 正确显示验证错误
    │   ├── 保存录屏
    │   └── 截屏修复后状态
    │
    ▼
[B7] Debug 子代理清理
    │   "The issue has been fixed. Please clean up the instrumentation."
    │   → 子代理移除所有 console.log 插桩
    │
    ▼
[B8] 提交
    │   git add -A && git commit -m "Fix null reference in form submit handler" && git push
    │
    ▼
[B9] 最终回复 (含 before/after 证据)
```

---

## 7. 环境设置流程 (已有模块20深化, 此处为简要时序)

```
[详细流程见 模块18: 环境设置工作流]

简要:
  并行发现 (4个子代理/搜索) → 创建TODO
    → 安装依赖 → SetupVmEnvironment → 验证 (lint/test/build/run)
      → Hello World 任务 → 创建/更新 AGENTS.md → git push
```

---

## 8. 超时与生命周期边界

### 8.1 超时层级

```
┌─────────────────────────────────────────────────────────┐
│  层级          │ 默认超时    │ 最大超时    │ 处理方式      │
├─────────────────────────────────────────────────────────┤
│  Shell 命令    │ 30秒       │ 600秒      │ 杀死进程      │
│  子代理执行    │ 无显式限制  │ 由会话限制  │ 由父代理控制   │
│  单轮 LLM     │ 120秒      │ 300秒      │ 重试          │
│  会话空闲      │ 1800秒     │ -          │ 快照+暂停     │
│  会话总时长    │ 7200秒     │ -          │ 快照+终止     │
│  update_script │ 300秒      │ -          │ 警告但继续    │
│  VM 启动       │ 60秒       │ -          │ 失败重试      │
│  快照恢复      │ 120秒      │ -          │ 失败重试      │
│  产物上传      │ 300秒      │ -          │ 跳过失败项    │
└─────────────────────────────────────────────────────────┘
```
