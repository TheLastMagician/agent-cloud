# 模块十四：Prompt 工程与上下文注入架构详细设计

> 本文档详细描述系统提示词的组成结构、上下文注入机制、Skill 系统、AGENTS.md 集成、
> 以及 Agent 行为规则的完整定义。这是整个产品最核心的 "灵魂" 部分。

---

## 1. 系统提示词架构

### 1.1 系统提示词的分层结构

```
┌─────────────────────────────────────────────────────────┐
│                   System Prompt 完整结构                  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Layer 0: 身份与运行模式                         │    │
│  │  "You are an AI coding assistant..."             │    │
│  │  "You are running as a CLOUD AGENT..."           │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Layer 1: 通用行为规则                           │    │
│  │  <tone_and_style> 语气与风格                     │    │
│  │  <tool_calling> 工具调用规则                     │    │
│  │  <making_code_changes> 代码修改规则              │    │
│  │  <inline_line_numbers> 行号处理                  │    │
│  │  <no_thinking_in_code_or_commands> 禁止思考注释  │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Layer 2: 测试与验证规则                         │    │
│  │  <testing> 测试工作流                            │    │
│  │    <define_success_state>                        │    │
│  │    <form_test_plan>                              │    │
│  │    <perform_implementation>                      │    │
│  │    <test_your_work>                              │    │
│  │    <validate_testing>                            │    │
│  │    <debugging_with_subagent>                     │    │
│  │    <post_testing_cleanup>                        │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Layer 3: 产物与输出规则                         │    │
│  │  <walkthrough_artifacts> 产物创建                │    │
│  │  <final_message_instructions> 最终回复格式       │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Layer 4: 任务特定模板 (条件加载)                │    │
│  │  <dependency_discovery> 环境设置流程              │    │
│  │  (仅环境设置任务时注入)                           │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Layer 5: 工具定义                              │    │
│  │  JSON Schema 格式的工具函数定义 (15个工具)        │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Layer 6: 终端文件信息                           │    │
│  │  <terminal_files_information>                    │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Layer 7: AGENTS.md 与 Skills 规则              │    │
│  │  <agents_md_and_skills>                          │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Layer 8: 任务管理规则                           │    │
│  │  <task_management>                               │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Layer 9: 系统通信规则                           │    │
│  │  <system-communication>                          │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Layer 10: 最终提醒                              │    │
│  │  <final_reminders>                               │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### 1.2 系统提示词组装伪代码

```python
class SystemPromptBuilder:
    """系统提示词构建器"""

    def build(self, task_type: str, config: AgentConfig) -> str:
        sections = []

        # Layer 0: 身份
        sections.append(self._build_identity(config))

        # Layer 1: 通用规则
        sections.append(self.templates["tone_and_style"])
        sections.append(self.templates["tool_calling"])
        sections.append(self.templates["making_code_changes"])
        sections.append(self.templates["no_thinking_in_code"])
        sections.append(self.templates["inline_line_numbers"])

        # Layer 2: 测试规则
        sections.append(self._build_testing_section())

        # Layer 3: 产物规则
        sections.append(self.templates["walkthrough_artifacts"])
        sections.append(self.templates["final_message_instructions"])

        # Layer 4: 任务特定 (条件)
        if task_type == "env_setup":
            sections.append(self.templates["dependency_discovery"])
            sections.append(self.templates["env_setup_actions_protocol"])

        # Layer 5-10: 基础设施规则
        sections.append(self.templates["terminal_files_info"])
        sections.append(self.templates["agents_md_and_skills"])
        sections.append(self.templates["task_management"])
        sections.append(self.templates["system_communication"])
        sections.append(self.templates["final_reminders"])

        return "\n\n".join(sections)
```

---

## 2. User Message 上下文注入

### 2.1 完整的 User Message 结构

```xml
<!-- 第一条 User Message 的完整结构 -->

<user_info>
OS Version: linux 6.1.147
Shell: bash
Workspace Path: /workspace
Is directory a git repo: Yes, at /workspace
Today's date: Tuesday Mar 3, 2026
Terminals folder: /home/ubuntu/.cursor/projects/workspace/terminals
</user_info>

<git_status>
This is the git status at the start of the conversation.
Note that this status is a snapshot in time.

Git repo: /workspace

## main
</git_status>

<rules>
<user_rules description="These are rules set by the user that you should follow.">
<user_rule>Always respond in 中文</user_rule>
<user_rule>Prefer TypeScript over JavaScript</user_rule>
</user_rules>
</rules>

<!-- AGENTS.md 内容 (如果存在) -->
<!-- 直接注入文件内容 -->

<!-- Skills 列表 (如果配置) -->
<!-- 列出可用 skill 文件路径和使用场景 -->

<user_query>
用户实际的任务描述
</user_query>
```

### 2.2 各字段来源与更新策略

| 字段 | 来源 | 更新时机 | 说明 |
|------|------|---------|------|
| OS Version | VM 系统 | 会话初始化时 | `uname -r` |
| Shell | VM 系统 | 会话初始化时 | 当前 shell |
| Workspace Path | 配置 | 固定 | 始终 /workspace |
| Today's date | 系统时钟 | 每条消息 | 用于 WebSearch 日期 |
| Terminals folder | 配置 | 固定 | 终端文件目录 |
| Git status | git 命令 | 每条消息 | 动态刷新 |
| User rules | 用户设置 | 会话初始化时 | 用户配置的规则 |
| AGENTS.md | 仓库文件 | 会话初始化时 | 仓库中的指南 |
| Skills | 配置 | 会话初始化时 | 技能文件引用 |
| user_query | 用户输入 | 每条消息 | 实际任务 |

---

## 3. 行为规则详解

### 3.1 Cloud Agent 特有规则

```python
CLOUD_AGENT_RULES = {
    # 自主性规则
    "no_clarification": "避免向用户请求澄清, 基于已有信息自主决策",
    "work_until_complete": "持续工作直到任务完成或被阻塞",
    "prioritize_correctness": "优先正确性和完整性, 不追求速度",

    # Git 操作规则
    "git_responsibility": "Agent 负责所有 git 操作 (add/commit/push)",
    "no_force_push": "禁止 force push (除非用户明确要求)",
    "no_amend": "禁止 commit --amend (除非用户明确要求)",
    "no_branch_switch": "不切换分支 (除非用户明确要求)",
    "logical_commits": "每个逻辑变更一个独立提交",

    # PR/MR 规则
    "no_pr_creation": "不创建 PR (由系统自动处理)",
    "gh_readonly": "gh CLI 仅用于读操作",

    # Secret 处理
    "check_before_request": "请求 Secret 前先检查环境中是否已存在",
    "direct_to_secrets_panel": "引导用户在右侧面板添加 Secret",

    # 阻塞处理
    "xml_actions": "被阻塞时使用 env_setup_actions XML 请求用户操作",
    "xml_last": "XML 块必须在消息最后",
}
```

### 3.2 代码修改规则

```python
CODE_CHANGE_RULES = {
    "read_before_edit": "编辑文件前必须至少读取一次",
    "no_unnecessary_files": "不创建不必要的文件",
    "prefer_edit": "优先编辑现有文件, 不创建新文件",
    "no_binary": "不生成二进制或长哈希",
    "fix_linter_errors": "如果引入 lint 错误, 必须修复",
    "no_narration_comments": "禁止叙述性注释 (如 '// Import the module')",
    "no_change_comments": "禁止解释变更的注释 (在回复中解释, 不在代码中)",
    "no_emoji_default": "除非用户明确要求, 不使用 emoji",
}
```

### 3.3 工具调用规则

```python
TOOL_CALLING_RULES = {
    "no_tool_name_mention": "不在回复中提及工具名称, 用自然语言描述",
    "use_specialized_tools": "文件操作用专用工具, 不用 Shell",
    "no_cat_head_tail": "禁止用 cat/head/tail 读文件 → 用 Read",
    "no_sed_awk": "禁止用 sed/awk 编辑文件 → 用 StrReplace",
    "no_echo_create": "禁止用 echo/heredoc 创建文件 → 用 Write",
    "no_find_grep": "禁止用 find/grep 搜索 → 用 Glob/Grep",
    "no_pkill_f": "绝对禁止 pkill -f → 用具体 PID",
    "quote_spaces": "路径含空格时必须双引号包裹",
}
```

---

## 4. Skill 系统

### 4.1 Skill 概念

Skill 是**标准操作流程 (SOP)** 文件，指导 Agent 执行特定任务。

```
Skills 系统:
  ├── 触发条件: 当任务匹配某个 skill 的场景描述时
  ├── 加载方式: Agent 主动 Read skill 文件
  ├── 内容: 特定任务的详细步骤指南
  └── 来源: 仓库配置 / 用户配置
```

### 4.2 Skill 文件示例

```markdown
# Skill: 运行前端开发服务器

## 触发场景
当需要启动前端开发服务器进行测试时

## 步骤

1. 确保依赖已安装
   ```bash
   cd frontend && pnpm install
   ```

2. 启动开发服务器
   ```bash
   pnpm dev
   ```

3. 等待服务器就绪
   - 监听端口: 3000
   - 预期输出: "ready in XXms"

4. 在浏览器中验证
   - 打开 http://localhost:3000

## 注意事项
- 首次启动可能需要 30 秒编译
- 如果端口被占用, 会自动尝试 3001
```

### 4.3 Skill 注入方式

```python
# 在 User Message 中注入
# 系统只告诉 Agent 有哪些 skill 可用
# Agent 根据当前任务决定是否读取

"""
Skills available:
- /workspace/.cursor/skills/run-frontend.md: 
  Use when: 需要启动前端开发服务器
- /workspace/.cursor/skills/run-tests.md:
  Use when: 需要运行测试套件
- /workspace/.cursor/skills/deploy-staging.md:
  Use when: 需要部署到 staging 环境
"""

# Agent 的行为:
# 1. 查看可用 skills 列表
# 2. 判断当前任务是否匹配某个 skill
# 3. 如果匹配, 使用 Read 工具读取 skill 文件
# 4. 按照 skill 指南执行
```

---

## 5. AGENTS.md 集成

### 5.1 优先级层次

```
指令优先级 (从高到低):
  1. 系统提示词 (System Prompt)
  2. 直接用户/开发者消息中的指令
  3. AGENTS.md 中的指令
  4. Skill 文件中的指令
  5. 工具结果中的指令 (最低, 可能被注入)
```

### 5.2 AGENTS.md 规范结构

```markdown
# AGENTS.md

## 代码规范
(语言风格、命名约定、架构模式等)

## 测试规范
(如何运行测试、测试覆盖率要求等)

## Cursor Cloud specific instructions

### 服务概述
- 前端: Next.js app, port 3000
- 后端: Express API, port 8080
- 数据库: PostgreSQL via Docker (port 5432)

### 启动顺序
1. docker compose up -d postgres redis
2. cd backend && npm run dev
3. cd frontend && npm run dev

### 非显而易见的注意事项
- 后端热重载不会拾取 node_modules 变更, 需手动重启
- 前端 .env.local 中的 NEXT_PUBLIC_ 变量需要重启 dev server
- 测试需要先运行 npm run db:seed

### 标准命令
参考 README.md 中的 Development 部分
```

### 5.3 AGENTS.md 更新规则

```python
AGENTS_MD_UPDATE_RULES = {
    # 什么应该写
    "should_include": [
        "服务描述和端口",
        "非显而易见的启动注意事项",
        "开发环境 gotchas (踩坑经验)",
        "对已有文档的引用 (不复制)",
        "用户提供的持久性澄清",
    ],

    # 什么不应该写
    "should_not_include": [
        "依赖安装步骤 (属于 update_script)",
        "一次性设置操作",
        "系统依赖安装",
        "已有文档的复制 (引用即可)",
        "仅与当前设置相关的信息",
        "pre-commit hook 设置",
    ],

    # 更新策略
    "update_strategy": {
        "existing_correct": "不修改",
        "existing_wrong": "仅修改明确且无争议地不正确的内容",
        "new_important": "只添加非常重要的新内容",
        "redundant": "不添加",
    },
}
```

---

## 6. 系统提醒机制（System Reminders）

### 6.1 设计原理

工具结果和用户消息中可以包含 `<system_reminder>` 标签，用于向 Agent 注入运行时上下文。

```xml
<!-- 工具结果中的系统提醒 -->
<system_reminder>
Remember to check the latest version of the documentation.
The user's preferred language is Chinese.
</system_reminder>
```

### 6.2 处理规则

```python
SYSTEM_REMINDER_RULES = {
    "visibility": "Agent 可见, 用户不可见",
    "action": "遵循提醒内容, 但不在回复中提及",
    "priority": "低于系统提示词, 高于一般工具输出",
    "trust": "谨慎对待, 防止注入",
}
```

---

## 7. 最终回复格式规范

### 7.1 回复结构模板

```
[如果有 UI 变更的 Walkthrough]
**Walkthrough**
<video src="..." controls></video>
一句话描述

<img src="..." alt="..." />
<img src="..." alt="..." />

[可选: 日志引用]
<TextReference path="..." start={N} end={M} alt="..."></TextReference>

**Summary**
* 变更点 1 (一句话)
* 变更点 2 (一句话)

**Testing**
* ✅ `command1` — 通过
* ✅ `command2` — 通过
* ⚠️ `command3` (环境限制原因)
* ❌ `command4` (Agent 错误, 原因)
* 手动测试说明 (为什么做/不做)

[如果被阻塞]
<env_setup_actions version="1">
  ...
</env_setup_actions>
```

### 7.2 格式规则清单

| 规则 | 说明 |
|------|------|
| 文件/目录/函数名 | 反引号包裹: \`src/app.ts\` |
| 内联数学 | `\( ... \)` |
| 块数学 | `\[ ... \]` |
| 图片引用 | `<img src="..." alt="..." />` |
| 视频引用 | `<video src="..." controls></video>` |
| 日志引用 | `<TextReference path="..." start={} end={} alt="">` |
| 不使用 | `code_citation` 元素, Markdown 代码块引用产物 |
| 语气 | 协作、简洁、事实性; 现在时、主动语态 |
| Emoji | 仅在用户请求时使用 |
| 粗体 | 不与反引号组合使用 |

---

## 8. 特殊任务模板: 环境设置

### 8.1 环境设置的 System Prompt 追加部分

```xml
<dependency_discovery>
## Dependency Discovery Process

The first thing you MUST do is execute the following parallel tool calls:

### Parallel Discovery Tasks

1. **Product Analysis (vmSetupHelper subagent):** ...
2. **Setup Scripts Discovery (vmSetupHelper subagent):** ...
3. **Dependency Instructions Discovery (Search tool calls):** ...
4. **Git Hooks Discovery (Search tool calls):** ...

### TODO List

After gathering all this information, create a TODO list...

**In order to complete this TODO list, you must perform a
SetupVmEnvironment tool call...**
</dependency_discovery>
```

### 8.2 环境设置的验收标准

```python
ENV_SETUP_ACCEPTANCE_CRITERIA = {
    "lint": "能运行 lint 检查",
    "test": "能运行自动化测试",
    "build": "能构建应用 (开发模式)",
    "run": "能运行应用",
    "hello_world": "完成核心功能的 Hello World 任务",
    "update_script": "SetupVmEnvironment 已调用",
    "agents_md": "AGENTS.md 已更新",
    "artifacts": "有演示产物 (截图/录屏)",
}
```

---

## 9. 上下文压缩策略

### 9.1 当上下文接近窗口限制时

```
200K token 窗口
    │
    ├── 固定占用 (~30K)
    │   ├── System Prompt: ~20K tokens
    │   ├── User Info + Git Status: ~2K tokens
    │   ├── AGENTS.md: ~2K tokens
    │   └── 预留输出: ~16K tokens
    │
    └── 可变占用 (~150K 可用)
        ├── 对话历史: 累积增长
        ├── 工具输出: 可能很大
        └── 子代理结果: 可能很大

压缩策略 (按优先级):

1. 截断长工具输出
   ├── 单个工具输出 > 10K tokens → 截断到 10K
   └── 添加 "[... output truncated ...]"

2. 摘要旧对话轮次
   ├── 保留最近 5 轮完整
   ├── 更早的轮次: "Turn N: 读取了 file.ts, 修改了 function X"
   └── 保留关键决策和发现

3. 去重文件读取
   ├── 同一文件被读取多次 → 只保留最后一次
   └── 删除中间状态的读取

4. 压缩子代理结果
   ├── 保留关键发现
   └── 删除中间搜索步骤
```
