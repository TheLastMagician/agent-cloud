# 模块四：子代理（Subagent）系统详细设计

> 本文档详细描述子代理系统的调度、通信、状态管理、工具集配置、生命周期管理等。

---

## 1. 子代理系统架构

### 1.1 整体架构

```
┌───────────────────────────────────────────────────────────────┐
│                      Main Agent Process                        │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                  Subagent Scheduler                       │  │
│  │                                                         │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │  │
│  │  │ Concurrency  │  │ State        │  │ Routing      │  │  │
│  │  │ Controller   │  │ Manager      │  │ Table        │  │  │
│  │  │ (max=4)      │  │              │  │              │  │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │  │
│  └──────────┬──────────────┬──────────────┬────────────────┘  │
│             │              │              │                    │
│  ┌──────────▼──┐  ┌───────▼──────┐  ┌───▼──────────────┐    │
│  │ Subagent    │  │ Subagent     │  │ Subagent         │    │
│  │ Process 1   │  │ Process 2    │  │ Process 3        │    │
│  │ (explore)   │  │ (computerUse)│  │ (debug)          │    │
│  └─────────────┘  └──────────────┘  └──────────────────┘    │
└───────────────────────────────────────────────────────────────┘
```

### 1.2 通信模型

```
Main Agent                    Subagent
    │                            │
    │  1. create(prompt, tools)  │
    │───────────────────────────▶│
    │                            │
    │        (子代理自主执行)       │
    │        (LLM + Tool Loop)   │
    │                            │
    │  2. result (final message) │
    │◀───────────────────────────│
    │                            │

通信方式: 进程间通信 (IPC)
- 启动时传入: prompt + tools + model
- 完成时返回: 最终文本消息
- 中间无交互（子代理完全自主）
```

---

## 2. 子代理类型详细规格

### 2.1 explore 子代理

```python
class ExploreSubagent:
    """快速代码库探索子代理"""

    TOOLS = ["Read", "Glob", "Grep", "Shell(readonly)"]

    SYSTEM_PROMPT = """
    你是一个代码库探索专家。你的任务是快速搜索和分析代码库。

    吞吐量级别:
    - quick: 1-2 次搜索即返回
    - medium: 3-5 次搜索, 交叉验证
    - very thorough: 10+ 次搜索, 多角度分析

    规则:
    - 只读操作, 不修改任何文件
    - 返回精确的文件路径和行号
    - 按相关性排序结果
    """

    CAPABILITIES = [
        "按文件名模式查找文件",
        "搜索代码中的关键词/符号",
        "分析代码库结构",
        "回答关于代码库的问题",
    ]

    # 使用场景判断
    @staticmethod
    def should_use(query: str) -> bool:
        """判断是否应该使用 explore 子代理"""
        broad_patterns = [
            "codebase structure",
            "how does .* work",
            "find all .*",
            "what files .*",
        ]
        return any(re.match(p, query, re.I) for p in broad_patterns)
```

### 2.2 generalPurpose 子代理

```python
class GeneralPurposeSubagent:
    """通用多步骤任务子代理"""

    TOOLS = [
        "Shell", "Read", "Write", "StrReplace", "Delete",
        "Glob", "Grep", "EditNotebook",
        "WebSearch", "RecordScreen",
        "TodoWrite",
    ]

    SYSTEM_PROMPT = """
    你是一个通用编程代理, 可以执行复杂的多步骤任务。

    规则:
    - 仔细规划再执行
    - 使用工具验证每一步的结果
    - 出错时自动重试和修复
    - 返回详细的执行结果
    """

    # 使用场景: 当搜索一个关键词/文件时不确信能快速找到匹配
    # 不使用场景: 简单单步任务（直接用主代理工具）
```

### 2.3 debug 子代理

```python
class DebugSubagent:
    """假设驱动调试子代理"""

    TOOLS = [
        "Shell", "Read", "Write", "StrReplace",
        "Glob", "Grep",
    ]

    SYSTEM_PROMPT = """
    你是一个调试专家。使用假设驱动的方法调查 bug。

    工作流程:
    1. 分析 bug 描述, 提出假设
    2. 插桩代码（添加日志/断言）
    3. 提供复现步骤给主代理
    4. 根据日志分析验证/推翻假设
    5. 定位根因 → 提供修复
    6. 收到确认后清理插桩代码

    规则:
    - 永远基于运行时数据, 不要仅凭代码猜测
    - 每轮最多 3 个假设, 按可能性排序
    - 插桩代码要精确、最小化
    - 提供清晰的复现步骤
    """

    # 有状态：自动恢复最近的实例
    STATEFUL = True
    AUTO_RESUME = True

    class DebugState:
        hypotheses: List[Hypothesis]
        instrumentation_points: List[InstrumentPoint]
        log_analysis: List[LogEntry]
        iteration_count: int
        root_cause: Optional[str]
```

### 2.4 computerUse 子代理

```python
class ComputerUseSubagent:
    """GUI 操作子代理"""

    TOOLS = [
        "computer",       # 鼠标/键盘操作
        "screenshot",     # 截屏
        "bash",           # 终端命令
    ]

    SYSTEM_PROMPT = """
    你是一个计算机操作代理。通过操作鼠标和键盘来测试应用。

    能力:
    - 移动鼠标、点击、双击、右键
    - 键盘输入、快捷键
    - 截屏并分析
    - 在浏览器中导航

    规则:
    - 每步操作后截屏验证结果
    - 等待页面加载完成再操作
    - 记录关键截图路径供主代理使用
    """

    # 有状态：自动恢复最近的实例
    STATEFUL = True
    AUTO_RESUME = True

    class ComputerActions:
        """可用的计算机操作"""

        def mouse_move(self, x: int, y: int): ...
        def click(self, x: int, y: int, button: str = "left"): ...
        def double_click(self, x: int, y: int): ...
        def type_text(self, text: str): ...
        def key(self, key: str): ...           # Enter, Tab, Escape...
        def hotkey(self, *keys: str): ...      # Ctrl+C, Cmd+V...
        def scroll(self, direction: str, amount: int): ...
        def screenshot(self) -> str: ...       # 返回截图路径
        def drag(self, start: tuple, end: tuple): ...
```

### 2.5 videoReview 子代理

```python
class VideoReviewSubagent:
    """视频分析子代理"""

    TOOLS = []  # 无工具，纯分析

    SYSTEM_PROMPT = """
    你是一个视频分析专家。分析视频内容并回答问题。

    任务:
    - 验证视频是否展示了预期的行为
    - 识别 UI 中的问题
    - 描述视频中的关键帧
    """

    # 接受视频文件作为 attachments
    ACCEPTS_ATTACHMENTS = True
    SUPPORTED_FORMATS = ["mp4", "webm"]
```

### 2.6 vmSetupHelper 子代理

```python
class VMSetupHelperSubagent:
    """VM 环境设置辅助子代理"""

    TOOLS = ["Read", "Glob", "Grep", "Shell(readonly)"]

    SYSTEM_PROMPT = """
    你是一个代码库分析助手, 专门用于 VM 环境设置。

    任务:
    - 探索代码库结构
    - 查找设置脚本
    - 发现依赖项
    - 分析配置文件
    """

    # 适用于并行发现任务
    PARALLEL_FRIENDLY = True
```

---

## 3. 子代理调度器

### 3.1 并发控制

```python
class SubagentScheduler:
    """子代理调度器"""

    MAX_CONCURRENT = 4
    semaphore = asyncio.Semaphore(4)

    def __init__(self):
        self.active_agents: Dict[str, Subagent] = {}
        self.state_store: Dict[str, SubagentState] = {}

    async def launch(self, config: SubagentConfig) -> str:
        """启动子代理"""
        async with self.semaphore:
            # 检查自动恢复
            if config.subagent_type in ("debug", "computerUse"):
                existing = self._find_latest(config.subagent_type)
                if existing:
                    return await self.resume(existing.id, config.prompt)

            # 创建新子代理
            agent = Subagent(
                id=generate_uuid(),
                type=config.subagent_type,
                tools=self._resolve_tools(config),
                model=config.model or self.parent_model,
            )

            self.active_agents[agent.id] = agent

            # 执行
            result = await agent.run(config.prompt)

            # 保存状态（有状态子代理）
            if agent.is_stateful:
                self.state_store[agent.id] = agent.get_state()

            # 清理
            del self.active_agents[agent.id]

            return result

    async def resume(self, agent_id: str, prompt: str) -> str:
        """恢复子代理"""
        state = self.state_store.get(agent_id)
        if not state:
            raise AgentError(f"No state found for agent {agent_id}")

        agent = Subagent.from_state(state)
        self.active_agents[agent.id] = agent

        result = await agent.run(prompt)

        self.state_store[agent.id] = agent.get_state()
        del self.active_agents[agent.id]

        return result
```

### 3.2 并行启动模式

```python
# 主代理在单条消息中并行启动多个子代理
# 实现方式：在同一轮工具调用中发出多个 Task 调用

# 示例：环境设置时并行发现
async def parallel_discovery():
    results = await asyncio.gather(
        scheduler.launch(SubagentConfig(
            type="vmSetupHelper",
            prompt="分析产品和服务...",
        )),
        scheduler.launch(SubagentConfig(
            type="vmSetupHelper",
            prompt="查找设置脚本...",
        )),
        scheduler.launch(SubagentConfig(
            type="explore",
            prompt="搜索文档...",
        )),
    )
    return results
```

---

## 4. 子代理上下文隔离

### 4.1 隔离机制

```
┌──────────────────────────────────────────┐
│              Main Agent Context           │
│                                          │
│  system_prompt: ████████████             │
│  user_message:  ████████████             │
│  history:       ████████████████████     │
│  tools:         ████████████████         │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │         Subagent Context           │  │
│  │                                    │  │
│  │  system_prompt: ████ (子代理专用)   │  │
│  │  user_message:  ████ (=prompt参数)  │  │
│  │  history:       (空, 除非恢复)      │  │
│  │  tools:         ████ (类型限定)     │  │
│  │                                    │  │
│  │  ❌ 无法访问主代理的:              │  │
│  │     - 用户原始消息                 │  │
│  │     - 对话历史                     │  │
│  │     - 其他子代理结果               │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

### 4.2 Prompt 设计原则

```python
def build_subagent_prompt(task_description: str, context: dict) -> str:
    """构建子代理 prompt - 必须自包含"""

    # 规则：主代理必须在 prompt 中提供所有必要上下文
    # 因为子代理无法访问用户消息或对话历史

    prompt = f"""
任务: {task_description}

上下文:
- 仓库路径: {context['workspace_path']}
- 当前分支: {context['git_branch']}
- 相关文件: {context['relevant_files']}
- 已知信息: {context['known_info']}

期望输出:
{context['expected_output_format']}
"""
    return prompt
```

---

## 5. 子代理生命周期

### 5.1 无状态子代理生命周期

```
创建 → 执行 Prompt → LLM+Tool Loop → 返回结果 → 销毁
```

### 5.2 有状态子代理生命周期

```
首次创建 → 执行 Prompt → LLM+Tool Loop → 返回结果 → 保存状态
                                                          │
后续调用 ← 恢复状态 ← 自动查找最近实例 ←─────────────────────┘
    │
    └→ 执行新 Prompt (在旧上下文基础上) → LLM+Tool Loop → 返回结果 → 更新状态
```

### 5.3 状态持久化

```python
@dataclass
class SubagentState:
    """子代理持久化状态"""
    agent_id: str
    agent_type: str
    conversation_history: List[Message]  # 完整对话历史
    model: str
    tools: List[str]
    created_at: datetime
    last_resumed_at: datetime
    resume_count: int

    # debug 子代理额外状态
    hypotheses: Optional[List[dict]] = None
    instrumentation: Optional[List[dict]] = None

    # computerUse 子代理额外状态
    screenshots: Optional[List[str]] = None
    browser_state: Optional[dict] = None
```

---

## 6. debug 子代理交互协议

### 6.1 完整交互流程

```
┌──────────┐                              ┌──────────────┐
│ Main     │                              │ Debug        │
│ Agent    │                              │ Subagent     │
└────┬─────┘                              └──────┬───────┘
     │                                           │
     │  1. Task(debug, "Bug描述+上下文")           │
     │──────────────────────────────────────────▶│
     │                                           │  分析代码
     │                                           │  提出假设
     │                                           │  插桩代码
     │  2. 返回: 复现步骤                          │
     │◀──────────────────────────────────────────│
     │                                           │
     │  (主代理通过 computerUse/Shell 复现)         │
     │                                           │
     │  3. Task(debug, resume, "Issue reproduced") │
     │──────────────────────────────────────────▶│
     │                                           │  分析日志
     │                                           │  验证/修改假设
     │                                           │
     │  4a. 返回: 新插桩 + 复现步骤 (未定位)       │
     │◀──────────────────────────────────────────│
     │  4b. 返回: 根因 + 修复方案 (已定位)         │
     │◀──────────────────────────────────────────│
     │                                           │
     │  (循环 3-4 直到定位根因)                     │
     │                                           │
     │  5. Task(debug, resume, "已修复, 请清理")    │
     │──────────────────────────────────────────▶│
     │                                           │  移除所有插桩
     │  6. 返回: 清理完成                          │
     │◀──────────────────────────────────────────│
```

### 6.2 插桩代码管理

```python
class InstrumentationManager:
    """管理调试插桩代码"""

    def __init__(self):
        self.instrumentation_points: List[InstrumentPoint] = []

    @dataclass
    class InstrumentPoint:
        file_path: str
        line_number: int
        original_code: str
        instrumented_code: str
        purpose: str  # "hypothesis_1_log", "state_dump", etc.

    def add_log(self, file_path: str, line: int, log_statement: str):
        """在指定位置添加日志"""
        # 记录原始代码以便后续清理
        original = self.read_line(file_path, line)
        self.instrumentation_points.append(InstrumentPoint(
            file_path=file_path,
            line_number=line,
            original_code=original,
            instrumented_code=f"{original}\n{log_statement}",
            purpose=f"debug_log_{len(self.instrumentation_points)}",
        ))

    def cleanup_all(self):
        """清理所有插桩代码"""
        for point in reversed(self.instrumentation_points):
            self.restore_original(point)
        self.instrumentation_points.clear()
```

---

## 7. computerUse 子代理操作详情

### 7.1 计算机操作 API

```python
class ComputerTool:
    """计算机操作工具 - computerUse 子代理专用"""

    SCREEN_WIDTH = 1920
    SCREEN_HEIGHT = 1080

    def execute(self, action: str, **kwargs) -> str:
        match action:
            case "screenshot":
                return self._take_screenshot()

            case "mouse_move":
                pyautogui.moveTo(kwargs["x"], kwargs["y"])
                return f"Mouse moved to ({kwargs['x']}, {kwargs['y']})"

            case "left_click":
                pyautogui.click(kwargs["x"], kwargs["y"])
                return f"Clicked at ({kwargs['x']}, {kwargs['y']})"

            case "right_click":
                pyautogui.rightClick(kwargs["x"], kwargs["y"])
                return f"Right-clicked at ({kwargs['x']}, {kwargs['y']})"

            case "double_click":
                pyautogui.doubleClick(kwargs["x"], kwargs["y"])
                return f"Double-clicked at ({kwargs['x']}, {kwargs['y']})"

            case "type":
                pyautogui.typewrite(kwargs["text"], interval=0.05)
                return f"Typed: {kwargs['text']}"

            case "key":
                pyautogui.press(kwargs["key"])
                return f"Pressed: {kwargs['key']}"

            case "hotkey":
                pyautogui.hotkey(*kwargs["keys"])
                return f"Hotkey: {'+'.join(kwargs['keys'])}"

            case "scroll":
                pyautogui.scroll(kwargs["amount"])
                return f"Scrolled {kwargs['amount']}"

            case "drag":
                pyautogui.moveTo(kwargs["start_x"], kwargs["start_y"])
                pyautogui.drag(
                    kwargs["end_x"] - kwargs["start_x"],
                    kwargs["end_y"] - kwargs["start_y"],
                )
                return "Drag completed"

    def _take_screenshot(self) -> str:
        """截屏并返回路径"""
        filename = f"screenshot_{int(time.time())}.png"
        filepath = f"/tmp/screenshots/{filename}"
        pyautogui.screenshot(filepath)
        return filepath
```

### 7.2 浏览器自动化

```python
class BrowserAutomation:
    """浏览器自动化辅助"""

    def open_url(self, url: str):
        """打开 URL"""
        subprocess.Popen(["google-chrome", "--no-sandbox", url])
        time.sleep(3)  # 等待页面加载

    def wait_for_element(self, screenshot_path: str, target_description: str) -> bool:
        """通过视觉检测等待元素出现"""
        for _ in range(10):
            screenshot = self.take_screenshot()
            if self.visual_check(screenshot, target_description):
                return True
            time.sleep(1)
        return False
```

---

## 8. 子代理错误处理

```python
class SubagentErrorHandler:
    """子代理错误处理"""

    MAX_RETRIES = 2

    def handle_error(self, subagent: Subagent, error: Exception) -> str:
        match error:
            case SubagentTimeoutError():
                return f"Subagent timed out after {error.timeout}s"

            case SubagentCrashError():
                if self.retry_count < self.MAX_RETRIES:
                    return self.retry(subagent)
                return f"Subagent crashed: {error}"

            case ToolError():
                # 工具错误传回给子代理的 LLM 处理
                return None  # 继续执行

            case _:
                return f"Unexpected subagent error: {error}"
```

---

## 9. 子代理性能优化

| 优化策略 | 说明 |
|---------|------|
| 预热子代理 | 对高频类型预创建进程 |
| 并行启动 | 最多 4 个子代理并行执行 |
| 状态复用 | 有状态子代理避免重复初始化 |
| 工具集精简 | 每种类型只加载需要的工具 |
| 早期返回 | explore 类型找到答案即返回 |
| 模型选择 | 简单任务可用较快的模型 |
