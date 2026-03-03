# 模块二：Agent 核心引擎详细设计

> 本文档详细描述 Cloud Agent 的核心引擎，包括 LLM 调用、上下文管理、工具执行循环、流式输出等。

---

## 1. 引擎架构

### 1.1 组件关系图

```
┌──────────────────────────────────────────────────────────────┐
│                    Agent Core Engine                          │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ Context      │  │ LLM          │  │ Response         │   │
│  │ Builder      │──▶ Orchestrator │──▶ Processor        │   │
│  └──────────────┘  └──────┬───────┘  └────────┬─────────┘   │
│                           │                    │             │
│                    ┌──────▼───────┐    ┌───────▼──────────┐  │
│                    │ Tool         │    │ Stream           │  │
│                    │ Executor     │    │ Emitter          │  │
│                    └──────┬───────┘    └──────────────────┘  │
│                           │                                  │
│                    ┌──────▼───────┐                          │
│                    │ Tool         │                          │
│                    │ Registry     │                          │
│                    └──────────────┘                          │
└──────────────────────────────────────────────────────────────┘
```

### 1.2 核心循环（Agent Loop）

```
┌─────────────────────────────────────────────────────┐
│                  Agent Main Loop                     │
│                                                     │
│   while not done:                                   │
│     1. context = context_builder.build()             │
│     2. response = llm.generate(context)              │
│     3. stream_emitter.emit(response.text)            │
│     4. if response.has_tool_calls():                 │
│        4a. results = tool_executor.execute_batch(    │
│              response.tool_calls                     │
│            )                                         │
│        4b. context_builder.append_tool_results(      │
│              results                                 │
│            )                                         │
│     5. else:                                         │
│        done = True                                   │
│     6. if response.is_end_turn:                      │
│        done = True                                   │
│                                                     │
│   return final_response                              │
└─────────────────────────────────────────────────────┘
```

---

## 2. 上下文构建器（Context Builder）

### 2.1 上下文层次结构

```python
class ContextBuilder:
    """构建 LLM 调用的完整上下文"""

    def build(self) -> List[Message]:
        context = []

        # Layer 1: System Prompt (不可变)
        context.append(Message(
            role="system",
            content=self.build_system_prompt()
        ))

        # Layer 2: User Message (含注入上下文)
        context.append(Message(
            role="user",
            content=self.build_user_message()
        ))

        # Layer 3: Conversation History (工具调用+结果)
        context.extend(self.conversation_history)

        return context
```

### 2.2 System Prompt 构建

```python
def build_system_prompt(self) -> str:
    """构建系统提示词"""
    sections = []

    # 核心身份与行为规则
    sections.append(self.load_template("identity_and_rules"))

    # 工具使用指南
    sections.append(self.load_template("tool_usage_guidelines"))

    # 代码修改规则
    sections.append(self.load_template("code_change_rules"))

    # 测试工作流规则
    sections.append(self.load_template("testing_workflow"))

    # 产物创建规则
    sections.append(self.load_template("artifact_guidelines"))

    # 最终回复格式
    sections.append(self.load_template("final_message_format"))

    # 特殊任务模板 (如环境设置)
    if self.task_type == "env_setup":
        sections.append(self.load_template("env_setup_workflow"))

    return "\n\n".join(sections)
```

### 2.3 User Message 注入上下文

```python
def build_user_message(self) -> str:
    """构建用户消息（含注入上下文）"""
    parts = []

    # 用户系统信息
    parts.append(f"""<user_info>
OS Version: {self.os_info}
Shell: {self.shell}
Workspace Path: {self.workspace_path}
Is directory a git repo: {self.git_info}
Today's date: {self.today}
Terminals folder: {self.terminals_path}
</user_info>""")

    # Git 状态快照
    parts.append(f"""<git_status>
{self.git_status_snapshot}
</git_status>""")

    # 用户规则
    if self.user_rules:
        parts.append(f"""<rules>
<user_rules>
{self.format_user_rules()}
</user_rules>
</rules>""")

    # AGENTS.md 内容
    if self.agents_md:
        parts.append(self.agents_md_content)

    # Skills 引用
    if self.skills:
        parts.append(self.format_skills())

    # 实际用户查询
    parts.append(f"""<user_query>
{self.user_query}
</user_query>""")

    return "\n\n".join(parts)
```

### 2.4 上下文窗口管理

```python
class ContextWindowManager:
    """管理上下文窗口大小，防止溢出"""

    MAX_CONTEXT_TOKENS = 200000   # Claude 的上下文窗口
    RESERVED_OUTPUT_TOKENS = 16000  # 预留给输出
    SYSTEM_PROMPT_BUDGET = 30000   # 系统提示预算

    def fit_context(self, messages: List[Message]) -> List[Message]:
        """确保上下文在窗口内"""
        total_tokens = sum(self.count_tokens(m) for m in messages)
        budget = self.MAX_CONTEXT_TOKENS - self.RESERVED_OUTPUT_TOKENS

        if total_tokens <= budget:
            return messages

        # 压缩策略（按优先级）
        messages = self.truncate_tool_outputs(messages)    # 1. 截断长输出
        messages = self.summarize_old_turns(messages)       # 2. 摘要旧轮次
        messages = self.drop_redundant_reads(messages)      # 3. 去重复读取

        return messages

    def truncate_tool_outputs(self, messages: List[Message]) -> List[Message]:
        """截断过长的工具输出"""
        MAX_TOOL_OUTPUT = 10000  # tokens
        for msg in messages:
            if msg.role == "tool" and self.count_tokens(msg) > MAX_TOOL_OUTPUT:
                msg.content = self.truncate_with_indicator(
                    msg.content, MAX_TOOL_OUTPUT
                )
        return messages
```

---

## 3. LLM 调用编排器（LLM Orchestrator）

### 3.1 调用流程

```python
class LLMOrchestrator:
    """编排 LLM 调用"""

    def generate(self, context: List[Message]) -> LLMResponse:
        """发起 LLM 推理"""

        # 1. 准备请求
        request = self.build_request(context)

        # 2. 选择模型
        model = self.select_model()

        # 3. 调用 API（流式）
        stream = self.api_client.stream(
            model=model,
            messages=request.messages,
            tools=request.tools,
            max_tokens=request.max_tokens,
            temperature=0,             # 编程任务用 0
            thinking={
                "type": "enabled",
                "budget_tokens": 10000  # 思考 token 预算
            }
        )

        # 4. 处理流式响应
        response = self.process_stream(stream)

        return response

    def build_request(self, context: List[Message]) -> LLMRequest:
        """构建 API 请求"""
        return LLMRequest(
            messages=context,
            tools=self.tool_registry.get_tool_definitions(),
            max_tokens=16000,
        )

    def select_model(self) -> str:
        """选择模型"""
        # 主代理使用最强模型
        if self.agent_type == "main":
            return "claude-4.6-opus-high-thinking"
        # 子代理可能使用不同模型
        elif self.agent_type == "subagent":
            return self.parent_model  # 继承父模型
```

### 3.2 流式响应处理

```python
class StreamProcessor:
    """处理流式 LLM 响应"""

    def process_stream(self, stream) -> LLMResponse:
        response = LLMResponse()

        for event in stream:
            match event.type:
                case "thinking":
                    # 思考内容（不展示给用户）
                    response.thinking += event.text

                case "text":
                    # 文本输出（实时展示给用户）
                    response.text += event.text
                    self.stream_emitter.emit_text(event.text)

                case "tool_use_start":
                    # 工具调用开始
                    tool_call = ToolCall(
                        id=event.id,
                        name=event.name,
                    )
                    response.tool_calls.append(tool_call)
                    self.stream_emitter.emit_tool_start(event.name)

                case "tool_use_input":
                    # 工具参数（增量）
                    response.tool_calls[-1].input += event.text

                case "tool_use_end":
                    # 工具调用结束
                    tool_call = response.tool_calls[-1]
                    tool_call.input = json.loads(tool_call.input)

                case "end_turn":
                    response.is_end_turn = True

        return response
```

### 3.3 思考模式（Extended Thinking）

```python
class ThinkingConfig:
    """扩展思考配置"""

    # 思考模式允许 LLM 在生成回复前进行内部推理
    # 思考内容不展示给用户，但影响后续输出质量

    THINKING_ENABLED = True
    THINKING_BUDGET_TOKENS = 10000  # 思考预算

    # 思考内容不计入用户可见输出
    # 但计入总 token 消耗
    # 适用于复杂推理、规划、调试等场景
```

---

## 4. 工具执行器（Tool Executor）

### 4.1 执行流程

```python
class ToolExecutor:
    """执行工具调用"""

    def execute_batch(self, tool_calls: List[ToolCall]) -> List[ToolResult]:
        """批量执行工具调用（支持并行）"""

        # 1. 分析依赖关系
        independent, dependent = self.analyze_dependencies(tool_calls)

        results = []

        # 2. 并行执行独立调用
        if independent:
            parallel_results = asyncio.gather(
                *[self.execute_single(tc) for tc in independent]
            )
            results.extend(parallel_results)

        # 3. 顺序执行依赖调用
        for tc in dependent:
            result = self.execute_single(tc)
            results.append(result)

        return results

    def execute_single(self, tool_call: ToolCall) -> ToolResult:
        """执行单个工具调用"""

        # 1. 验证参数
        self.validate_params(tool_call)

        # 2. 获取工具实例
        tool = self.tool_registry.get(tool_call.name)

        # 3. 安全检查
        self.security_check(tool_call)

        # 4. 执行
        try:
            output = tool.execute(**tool_call.input)
        except ToolError as e:
            output = f"Error: {str(e)}"
        except TimeoutError:
            output = "Error: Tool execution timed out"

        # 5. 脱敏处理
        output = self.redact_secrets(output)

        # 6. 截断处理
        output = self.truncate_if_needed(output)

        return ToolResult(
            tool_call_id=tool_call.id,
            output=output
        )

    def redact_secrets(self, output: str) -> str:
        """脱敏处理：替换 Secret 值"""
        for secret_name, secret_value in self.secrets.items():
            output = output.replace(secret_value, "[REDACTED]")
        return output
```

### 4.2 工具注册表

```python
class ToolRegistry:
    """工具注册表"""

    def __init__(self):
        self.tools = {}
        self.register_builtin_tools()

    def register_builtin_tools(self):
        """注册内置工具"""
        self.register("Shell", ShellTool())
        self.register("Read", ReadTool())
        self.register("Write", WriteTool())
        self.register("StrReplace", StrReplaceTool())
        self.register("Delete", DeleteTool())
        self.register("Glob", GlobTool())
        self.register("Grep", GrepTool())
        self.register("EditNotebook", EditNotebookTool())
        self.register("RecordScreen", RecordScreenTool())
        self.register("WebSearch", WebSearchTool())
        self.register("SetupVmEnvironment", SetupVmEnvironmentTool())
        self.register("TodoWrite", TodoWriteTool())
        self.register("Task", TaskTool())
        self.register("ListMcpResources", ListMcpResourcesTool())
        self.register("FetchMcpResource", FetchMcpResourceTool())

    def get_tool_definitions(self) -> List[dict]:
        """获取所有工具的 JSON Schema 定义"""
        return [tool.schema for tool in self.tools.values()]
```

---

## 5. 对话历史管理

### 5.1 消息类型

```python
from enum import Enum
from dataclasses import dataclass
from typing import List, Optional

class MessageRole(Enum):
    SYSTEM = "system"
    USER = "user"
    ASSISTANT = "assistant"
    TOOL = "tool"

@dataclass
class Message:
    role: MessageRole
    content: str
    tool_calls: Optional[List[ToolCall]] = None  # assistant 消息中
    tool_call_id: Optional[str] = None           # tool 消息中
    thinking: Optional[str] = None               # assistant 消息中

@dataclass
class ToolCall:
    id: str
    name: str
    input: dict

@dataclass
class ToolResult:
    tool_call_id: str
    output: str
    is_error: bool = False
```

### 5.2 对话历史结构

```
Turn 1:
  [system]     系统提示词
  [user]       用户查询 + 注入上下文

Turn 2:
  [assistant]  (thinking: ...) + 文本 + [ToolCall(Shell, ...), ToolCall(Read, ...)]
  [tool]       Shell 结果
  [tool]       Read 结果

Turn 3:
  [assistant]  (thinking: ...) + 文本 + [ToolCall(Write, ...)]
  [tool]       Write 结果

Turn N:
  [assistant]  最终文本回复 (无工具调用 → 结束)
```

### 5.3 跟进消息处理

```python
class ConversationManager:
    """跨轮次对话管理"""

    def handle_followup(self, user_message: str):
        """处理用户跟进消息"""

        # 追加用户消息到历史
        self.history.append(Message(
            role=MessageRole.USER,
            content=user_message
        ))

        # 重新注入上下文（Git 状态可能已变化）
        self.refresh_dynamic_context()

        # 继续 Agent Loop
        self.agent_loop.run()
```

---

## 6. 错误处理与重试

### 6.1 LLM 调用错误处理

```python
class LLMErrorHandler:
    """LLM 调用错误处理"""

    MAX_RETRIES = 3
    RETRY_DELAYS = [1, 5, 15]  # 秒

    def call_with_retry(self, request: LLMRequest) -> LLMResponse:
        for attempt in range(self.MAX_RETRIES):
            try:
                return self.llm_client.call(request)

            except RateLimitError:
                delay = self.RETRY_DELAYS[attempt]
                sleep(delay)
                continue

            except ContextWindowExceededError:
                request = self.shrink_context(request)
                continue

            except APIConnectionError:
                delay = self.RETRY_DELAYS[attempt]
                sleep(delay)
                continue

            except InvalidRequestError as e:
                raise AgentError(f"Invalid LLM request: {e}")

        raise AgentError("Max retries exceeded for LLM call")
```

### 6.2 工具执行错误处理

```python
class ToolErrorHandler:
    """工具执行错误分类"""

    def handle_error(self, tool_name: str, error: Exception) -> str:
        """将错误转为 Agent 可理解的消息"""

        if isinstance(error, FileNotFoundError):
            return f"Error: File not found - {error.filename}"

        elif isinstance(error, PermissionError):
            return f"Error: Permission denied - {error}"

        elif isinstance(error, TimeoutError):
            return f"Error: Command timed out after {error.timeout}ms"

        elif isinstance(error, CommandFailedError):
            return f"Exit code: {error.exit_code}\n\n{error.stderr}"

        else:
            return f"Error: {type(error).__name__}: {str(error)}"
```

---

## 7. 性能优化

### 7.1 并行工具调用

```
串行执行 (慢):           并行执行 (快):
  Tool A ─────▶           Tool A ─────▶
              Tool B ──▶  Tool B ─────▶    节省时间
                 Tool C ──▶ Tool C ───▶
  总时间: A+B+C           总时间: max(A,B,C)
```

### 7.2 预取与缓存

```python
class ContextCache:
    """上下文缓存"""

    def __init__(self):
        self.file_cache = LRUCache(maxsize=100)     # 文件内容缓存
        self.glob_cache = TTLCache(ttl=60)           # Glob 结果缓存
        self.git_status_cache = TTLCache(ttl=30)     # Git 状态缓存

    def get_file(self, path: str) -> Optional[str]:
        """缓存的文件读取"""
        cached = self.file_cache.get(path)
        if cached and cached.mtime == os.path.getmtime(path):
            return cached.content
        return None
```

### 7.3 Token 效率

| 优化策略 | 方法 |
|---------|------|
| 工具输出截断 | 超过阈值的输出自动截断并添加省略标记 |
| 文件读取限行 | 支持 offset + limit 参数避免读取整个大文件 |
| 搜索结果限制 | Grep 结果上限截断 |
| 旧轮次摘要 | 长对话中对早期轮次进行摘要压缩 |
| 去重读取 | 检测并合并重复的文件读取 |

---

## 8. Agent 能力边界

### 8.1 Agent 可以做的

| 能力 | 说明 |
|------|------|
| 读写文件 | 任意路径（安全限制内） |
| 执行命令 | Shell 命令（含后台进程） |
| 搜索代码 | 文件名模式匹配 + 内容搜索 |
| Git 操作 | add / commit / push |
| 浏览器操作 | 通过 computerUse 子代理 |
| 网络搜索 | WebSearch 工具 |
| 调试代码 | 通过 debug 子代理 |
| 录制屏幕 | 视频演示 |
| 管理任务 | TODO 列表 |
| 配置环境 | 更新脚本 + AGENTS.md |

### 8.2 Agent 不可以做的

| 限制 | 说明 |
|------|------|
| 创建 PR | 由系统自动处理 |
| 修改 GitHub 资源 | gh CLI 只读 |
| 强制推送 | 除非用户明确要求 |
| 按名杀进程 | 禁止 pkill -f |
| 访问宿主机 | VM 隔离 |
| 持久化跨会话状态 | 除了 AGENTS.md 和 Git |
