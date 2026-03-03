# 模块三：工具系统详细设计

> 本文档详细描述 Cloud Agent 的工具系统，包含每个工具的内部实现逻辑、参数校验、错误处理、边界约束等。

---

## 1. 工具系统架构

### 1.1 分层架构

```
┌──────────────────────────────────────────────────┐
│                 Tool Interface Layer               │
│  JSON Schema 定义 → 参数解析 → 类型验证            │
├──────────────────────────────────────────────────┤
│               Tool Execution Layer                 │
│  安全检查 → 执行逻辑 → 输出格式化 → 脱敏处理       │
├──────────────────────────────────────────────────┤
│               Tool Runtime Layer                   │
│  文件系统 API / 进程管理 / 网络请求 / 子进程创建    │
└──────────────────────────────────────────────────┘
```

### 1.2 工具基类

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Any, Dict

@dataclass
class ToolSchema:
    name: str
    description: str
    parameters: Dict[str, Any]  # JSON Schema
    required: list[str]

class BaseTool(ABC):
    """所有工具的基类"""

    @property
    @abstractmethod
    def schema(self) -> ToolSchema:
        """返回工具的 JSON Schema 定义"""
        pass

    @abstractmethod
    def execute(self, **kwargs) -> str:
        """执行工具逻辑，返回字符串结果"""
        pass

    def validate_params(self, params: dict):
        """参数校验"""
        for required_param in self.schema.required:
            if required_param not in params:
                raise ToolError(f"Missing required parameter: {required_param}")

    def format_output(self, output: Any) -> str:
        """格式化输出"""
        if isinstance(output, str):
            return output
        return str(output)
```

---

## 2. Shell 工具详细设计

### 2.1 功能概述

Shell 工具在持久化的 Shell 会话中执行命令，支持状态保持（工作目录、环境变量跨调用持久）。

### 2.2 内部实现

```python
class ShellTool(BaseTool):
    """Shell 命令执行工具"""

    DEFAULT_TIMEOUT = 30000      # 30 秒
    MAX_TIMEOUT = 600000         # 10 分钟
    MAX_OUTPUT_SIZE = 100000     # 字符

    def __init__(self):
        self.sessions = {}       # session_id → ShellSession
        self.default_session = ShellSession(cwd="/workspace")

    def execute(
        self,
        command: str,
        working_directory: str = None,
        timeout: int = None,
        is_background: bool = False,
        description: str = None,
    ) -> str:

        # 1. 确定超时
        timeout = min(timeout or self.DEFAULT_TIMEOUT, self.MAX_TIMEOUT)

        # 2. 确定工作目录
        session = self.default_session
        if working_directory:
            effective_cwd = working_directory
        else:
            effective_cwd = session.cwd

        # 3. 执行命令
        if is_background:
            result = self._execute_background(command, effective_cwd)
        else:
            result = self._execute_foreground(command, effective_cwd, timeout)

        # 4. 更新会话状态
        session.update_from_result(result)

        # 5. 格式化输出
        return self._format_result(result)

    def _execute_foreground(self, command: str, cwd: str, timeout: int) -> ExecResult:
        """前台执行（等待完成或超时）"""
        proc = subprocess.Popen(
            ["bash", "-c", command],
            cwd=cwd,
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            env=self._get_env(),
        )

        try:
            stdout, _ = proc.communicate(timeout=timeout / 1000)
            return ExecResult(
                exit_code=proc.returncode,
                output=stdout.decode("utf-8", errors="replace"),
                duration_ms=elapsed,
            )
        except subprocess.TimeoutExpired:
            proc.kill()
            return ExecResult(
                exit_code=-1,
                output="Command timed out",
                timed_out=True,
            )

    def _execute_background(self, command: str, cwd: str) -> ExecResult:
        """后台执行（立即返回）"""
        proc = subprocess.Popen(
            ["bash", "-c", command],
            cwd=cwd,
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            env=self._get_env(),
            start_new_session=True,  # 分离进程组
        )
        return ExecResult(
            exit_code=0,
            output=f"Background process started with PID {proc.pid}",
            pid=proc.pid,
        )

    def _format_result(self, result: ExecResult) -> str:
        """格式化执行结果"""
        parts = [f"Exit code: {result.exit_code}"]
        if result.output:
            output = result.output[:self.MAX_OUTPUT_SIZE]
            parts.append(f"\nCommand output:\n\n```\n{output}\n```")
        parts.append(f"\nCommand completed in {result.duration_ms} ms.")
        parts.append("\nShell state (cwd, env vars) persists for subsequent calls.")
        return "\n".join(parts)
```

### 2.3 Shell 会话状态模型

```python
@dataclass
class ShellSession:
    """Shell 会话状态"""
    cwd: str                          # 当前工作目录
    env: dict = field(default_factory=dict)  # 环境变量
    pid: int = None                   # 会话 PID

    def update_from_result(self, result: ExecResult):
        """从执行结果更新状态"""
        if result.new_cwd:
            self.cwd = result.new_cwd
        if result.env_changes:
            self.env.update(result.env_changes)
```

### 2.4 终端状态文件同步

```python
class TerminalFileSync:
    """将终端状态同步到文本文件"""

    TERMINALS_DIR = "/home/ubuntu/.cursor/projects/workspace/terminals"

    def sync(self, session_id: str, session: ShellSession, result: ExecResult):
        """同步终端状态到文件"""
        filepath = os.path.join(self.TERMINALS_DIR, f"{session_id}.txt")

        content = f"""---
pid: {session.pid}
cwd: {session.cwd}
last_command: {result.command}
last_exit_code: {result.exit_code}
---
{result.full_output}"""

        with open(filepath, "w") as f:
            f.write(content)
```

---

## 3. Read 工具详细设计

### 3.1 内部实现

```python
class ReadTool(BaseTool):
    """文件读取工具"""

    MAX_FILE_SIZE = 2_000_000     # 2MB 文本文件
    MAX_IMAGE_SIZE = 10_000_000   # 10MB 图片文件

    SUPPORTED_IMAGES = {".jpeg", ".jpg", ".png", ".gif", ".webp"}
    SUPPORTED_DOCS = {".pdf"}

    def execute(self, path: str, offset: int = None, limit: int = None) -> str:
        path = os.path.abspath(path)

        # 1. 检查文件是否存在
        if not os.path.exists(path):
            return f"Error: File not found: {path}"

        # 2. 检查文件类型
        ext = os.path.splitext(path)[1].lower()

        if ext in self.SUPPORTED_IMAGES:
            return self._read_image(path)
        elif ext in self.SUPPORTED_DOCS:
            return self._read_pdf(path)
        else:
            return self._read_text(path, offset, limit)

    def _read_text(self, path: str, offset: int = None, limit: int = None) -> str:
        """读取文本文件"""
        with open(path, "r", encoding="utf-8", errors="replace") as f:
            lines = f.readlines()

        if not lines:
            return "File is empty."

        # 处理 offset
        if offset is not None:
            if offset < 0:
                # 负数：从末尾计算
                start_idx = max(0, len(lines) + offset)
            else:
                # 正数：1-indexed
                start_idx = max(0, offset - 1)
        else:
            start_idx = 0

        # 处理 limit
        if limit is not None:
            end_idx = min(start_idx + limit, len(lines))
        else:
            end_idx = len(lines)

        # 格式化输出（带行号）
        result_lines = []
        for i in range(start_idx, end_idx):
            line_num = i + 1  # 1-indexed
            line_content = lines[i].rstrip("\n")
            result_lines.append(f"{line_num:>6}|{line_content}")

        return "\n".join(result_lines)

    def _read_image(self, path: str) -> ImageContent:
        """读取图片文件（返回 base64 编码）"""
        with open(path, "rb") as f:
            data = f.read()

        if len(data) > self.MAX_IMAGE_SIZE:
            return "Error: Image file too large"

        return ImageContent(
            type="image",
            media_type=self._get_media_type(path),
            data=base64.b64encode(data).decode()
        )

    def _read_pdf(self, path: str) -> str:
        """读取 PDF 文件（转为文本）"""
        text = extract_text_from_pdf(path)
        return text[:self.MAX_FILE_SIZE]
```

---

## 4. Write 工具详细设计

```python
class WriteTool(BaseTool):
    """文件写入工具"""

    def execute(self, path: str, contents: str) -> str:
        path = os.path.abspath(path)

        # 1. 确保父目录存在
        parent = os.path.dirname(path)
        os.makedirs(parent, exist_ok=True)

        # 2. 写入文件
        with open(path, "w", encoding="utf-8") as f:
            f.write(contents)

        return f"Wrote contents to {path}"
```

---

## 5. StrReplace 工具详细设计

### 5.1 核心逻辑

```python
class StrReplaceTool(BaseTool):
    """精确字符串替换工具"""

    def execute(
        self,
        path: str,
        old_string: str,
        new_string: str,
        replace_all: bool = False,
    ) -> str:
        path = os.path.abspath(path)

        # 1. 读取文件
        with open(path, "r", encoding="utf-8") as f:
            content = f.read()

        # 2. 验证 old_string 与 new_string 不同
        if old_string == new_string:
            return "Error: old_string and new_string are the same"

        # 3. 检查 old_string 是否存在
        count = content.count(old_string)
        if count == 0:
            return f"Error: old_string not found in {path}"

        # 4. 唯一性检查（非 replace_all 模式）
        if not replace_all and count > 1:
            return (
                f"Error: old_string appears {count} times in {path}. "
                "Provide more context to make it unique, or use replace_all=true."
            )

        # 5. 执行替换
        if replace_all:
            new_content = content.replace(old_string, new_string)
        else:
            new_content = content.replace(old_string, new_string, 1)

        # 6. 写入文件
        with open(path, "w", encoding="utf-8") as f:
            f.write(new_content)

        return f"Successfully replaced text in {path}"
```

### 5.2 设计约束

| 约束 | 原因 |
|------|------|
| old_string 必须唯一 | 避免错误替换不想修改的位置 |
| old_string ≠ new_string | 避免无意义操作 |
| 保留缩进 | Agent 必须匹配原文缩进 |
| 不添加无意义注释 | 代码质量要求 |

---

## 6. Glob 工具详细设计

```python
class GlobTool(BaseTool):
    """文件名模式匹配搜索"""

    def execute(
        self,
        glob_pattern: str,
        target_directory: str = None,
    ) -> str:
        # 1. 自动添加递归前缀
        if not glob_pattern.startswith("**/"):
            glob_pattern = f"**/{glob_pattern}"

        # 2. 确定搜索目录
        search_dir = target_directory or self.workspace_root

        # 3. 执行 glob 搜索
        matches = sorted(
            Path(search_dir).glob(glob_pattern),
            key=lambda p: p.stat().st_mtime,
            reverse=True,  # 按修改时间降序
        )

        # 4. 格式化输出
        if not matches:
            return f"Result of search in '{search_dir}': 0 files found"

        result = f"Result of search in '{search_dir}': {len(matches)} files found\n"
        for match in matches:
            result += f"{match}\n"

        return result
```

---

## 7. Grep 工具详细设计

```python
class GrepTool(BaseTool):
    """基于 ripgrep 的内容搜索工具"""

    MAX_OUTPUT_LINES = 5000  # 输出行数上限

    def execute(
        self,
        pattern: str,
        path: str = None,
        glob: str = None,
        type: str = None,
        output_mode: str = "content",
        context_before: int = None,   # -B
        context_after: int = None,    # -A
        context: int = None,          # -C
        case_insensitive: bool = False,  # -i
        multiline: bool = False,
        head_limit: int = None,
        offset: int = None,
    ) -> str:
        # 1. 构建 ripgrep 命令
        cmd = ["rg"]

        # 输出模式
        if output_mode == "files_with_matches":
            cmd.append("-l")
        elif output_mode == "count":
            cmd.append("-c")

        # 搜索选项
        if case_insensitive:
            cmd.append("-i")
        if multiline:
            cmd.extend(["-U", "--multiline-dotall"])
        if glob:
            cmd.extend(["--glob", glob])
        if type:
            cmd.extend(["--type", type])

        # 上下文行数
        if context:
            cmd.extend(["-C", str(context)])
        else:
            if context_before:
                cmd.extend(["-B", str(context_before)])
            if context_after:
                cmd.extend(["-A", str(context_after)])

        # 行号
        cmd.append("-n")

        # 模式和路径
        cmd.append(pattern)
        if path:
            cmd.extend(["--", path])

        # 2. 执行搜索
        result = subprocess.run(
            cmd,
            cwd=self.workspace_root,
            capture_output=True,
            text=True,
            timeout=30,
        )

        # 3. 处理输出
        output = result.stdout
        lines = output.split("\n")

        # 分页
        if offset:
            lines = lines[offset:]
        if head_limit:
            lines = lines[:head_limit]

        # 截断
        if len(lines) > self.MAX_OUTPUT_LINES:
            lines = lines[:self.MAX_OUTPUT_LINES]
            truncated = True
        else:
            truncated = False

        return "\n".join(lines) + (
            "\n(results truncated)" if truncated else ""
        )
```

---

## 8. EditNotebook 工具详细设计

```python
class EditNotebookTool(BaseTool):
    """Jupyter Notebook 编辑工具"""

    def execute(
        self,
        target_notebook: str,
        cell_idx: int,
        is_new_cell: bool,
        cell_language: str,  # python, markdown, javascript, etc.
        old_string: str,
        new_string: str,
    ) -> str:
        # 1. 读取 notebook
        with open(target_notebook, "r") as f:
            notebook = json.load(f)

        if is_new_cell:
            # 2a. 插入新 cell
            new_cell = self._create_cell(cell_language, new_string)
            notebook["cells"].insert(cell_idx, new_cell)
        else:
            # 2b. 编辑现有 cell
            cell = notebook["cells"][cell_idx]
            cell_content = "".join(cell["source"])

            if old_string not in cell_content:
                return f"Error: old_string not found in cell {cell_idx}"
            if cell_content.count(old_string) > 1:
                return f"Error: old_string not unique in cell {cell_idx}"

            new_content = cell_content.replace(old_string, new_string, 1)
            cell["source"] = new_content.splitlines(keepends=True)

        # 3. 写回 notebook
        with open(target_notebook, "w") as f:
            json.dump(notebook, f, indent=1)

        return f"Successfully edited notebook {target_notebook}"

    def _create_cell(self, language: str, content: str) -> dict:
        cell_type = "code" if language not in ("markdown", "raw") else language
        return {
            "cell_type": cell_type,
            "source": content.splitlines(keepends=True),
            "metadata": {},
            "outputs": [] if cell_type == "code" else None,
        }
```

---

## 9. WebSearch 工具详细设计

```python
class WebSearchTool(BaseTool):
    """网络搜索工具"""

    def execute(
        self,
        search_term: str,
        explanation: str = None,
    ) -> str:
        # 1. 调用搜索 API
        results = self.search_api.search(
            query=search_term,
            num_results=10,
        )

        # 2. 摘要化结果
        summaries = []
        for result in results:
            summaries.append({
                "title": result.title,
                "url": result.url,
                "snippet": result.snippet,
                "content_summary": self._summarize(result.content),
            })

        # 3. 格式化输出
        return self._format_results(summaries)
```

---

## 10. RecordScreen 工具详细设计

```python
class RecordScreenTool(BaseTool):
    """屏幕录制控制工具"""

    ARTIFACTS_DIR = "/opt/cursor/artifacts"

    def __init__(self):
        self.recorder = None       # 录制进程
        self.recording = False     # 录制状态

    def execute(
        self,
        mode: str,
        save_as_filename: str = None,
    ) -> str:
        match mode:
            case "START_RECORDING":
                return self._start()
            case "SAVE_RECORDING":
                return self._save(save_as_filename)
            case "DISCARD_RECORDING":
                return self._discard()

    def _start(self) -> str:
        if self.recording:
            return "Error: Already recording"

        # 使用 ffmpeg 录制桌面
        self.temp_file = tempfile.mktemp(suffix=".mp4")
        self.recorder = subprocess.Popen([
            "ffmpeg",
            "-f", "x11grab",
            "-video_size", "1920x1080",
            "-i", ":0.0",
            "-codec:v", "libx264",
            "-preset", "ultrafast",
            "-y", self.temp_file
        ])
        self.recording = True
        return "Screen recording started"

    def _save(self, filename: str = None) -> str:
        if not self.recording:
            return "Error: Not recording"

        self.recorder.terminate()
        self.recorder.wait()
        self.recording = False

        # 确定输出文件名
        if filename:
            output_name = f"{filename}.mp4"
        else:
            output_name = f"recording_{int(time.time())}.mp4"

        output_path = os.path.join(self.ARTIFACTS_DIR, output_name)
        shutil.move(self.temp_file, output_path)

        # 同时生成 demo 版本
        demo_path = output_path.replace(".mp4", "_demo.mp4")
        self._create_demo_version(output_path, demo_path)

        return f"Recording saved to {output_path}"

    def _discard(self) -> str:
        if not self.recording:
            return "Error: Not recording"

        self.recorder.terminate()
        self.recorder.wait()
        self.recording = False

        os.remove(self.temp_file)
        return "Recording discarded"
```

---

## 11. SetupVmEnvironment 工具详细设计

```python
class SetupVmEnvironmentTool(BaseTool):
    """VM 环境更新脚本设置工具"""

    def execute(self, update_script: str) -> str:
        # 1. 验证脚本
        self._validate_script(update_script)

        # 2. 保存到元数据
        self.session_metadata["update_script"] = update_script

        # 3. 通知编排层
        self.orchestrator.set_update_script(update_script)

        return "Environment setup commands suggested successfully. The user will review them."

    def _validate_script(self, script: str):
        """验证更新脚本"""
        dangerous_patterns = [
            "docker compose up",
            "docker-compose up",
            "npm run dev",
            "pnpm dev",
            "python manage.py runserver",
            "rm -rf /",
        ]
        for pattern in dangerous_patterns:
            if pattern in script:
                raise ToolError(
                    f"Update script should not contain service startup "
                    f"commands: '{pattern}'"
                )
```

---

## 12. TodoWrite 工具详细设计

```python
class TodoWriteTool(BaseTool):
    """任务列表管理工具"""

    def __init__(self):
        self.todos: Dict[str, TodoItem] = {}

    def execute(
        self,
        todos: List[dict],
        merge: bool,
    ) -> str:
        if merge:
            # 增量更新
            for todo_data in todos:
                todo_id = todo_data["id"]
                if todo_id in self.todos:
                    # 更新现有项
                    existing = self.todos[todo_id]
                    if "content" in todo_data:
                        existing.content = todo_data["content"]
                    if "status" in todo_data:
                        existing.status = todo_data["status"]
                else:
                    # 添加新项
                    self.todos[todo_id] = TodoItem(**todo_data)
        else:
            # 全量替换
            self.todos = {
                t["id"]: TodoItem(**t) for t in todos
            }

        # 通知 UI 更新
        self.stream_emitter.emit_todo_update(self.todos)

        return self._format_response()

    def _format_response(self) -> str:
        lines = ["Successfully updated TODOs.\n"]
        lines.append("Here are the latest contents of your todo list:")
        for todo in self.todos.values():
            status_label = todo.status.upper().replace("_", " ")
            lines.append(f"- **{status_label}**: {todo.content} (id: {todo.id})")
        return "\n".join(lines)
```

---

## 13. Task 工具详细设计（子代理启动器）

```python
class TaskTool(BaseTool):
    """子代理启动/恢复工具"""

    MAX_CONCURRENT_SUBAGENTS = 4

    def execute(
        self,
        subagent_type: str,
        description: str,
        prompt: str,
        readonly: bool = False,
        resume: str = None,
        attachments: List[str] = None,
        model: str = None,
    ) -> str:
        # 1. 检查并发数
        if len(self.active_subagents) >= self.MAX_CONCURRENT_SUBAGENTS:
            return "Error: Maximum concurrent subagents reached (4)"

        # 2. 自动恢复有状态子代理
        if subagent_type in ("debug", "computerUse"):
            existing = self.find_latest_subagent(subagent_type)
            if existing:
                resume = existing.id

        # 3. 启动/恢复子代理
        if resume:
            subagent = self.resume_subagent(resume, prompt)
        else:
            subagent = self.create_subagent(
                subagent_type=subagent_type,
                prompt=prompt,
                model=model or self.parent_model,
                readonly=readonly,
                attachments=attachments,
            )

        # 4. 等待完成
        result = subagent.wait_for_completion()

        # 5. 返回结果
        return f"Agent ID: {subagent.id}\n\n{result}"

    def create_subagent(self, **kwargs) -> Subagent:
        """创建新的子代理进程"""
        subagent = Subagent(
            id=generate_uuid(),
            type=kwargs["subagent_type"],
            tools=self._get_tools_for_type(kwargs["subagent_type"]),
            system_prompt=self._get_prompt_for_type(kwargs["subagent_type"]),
            model=kwargs["model"],
        )
        subagent.start(kwargs["prompt"])
        return subagent

    def _get_tools_for_type(self, subagent_type: str) -> List[BaseTool]:
        """根据子代理类型返回可用工具集"""
        match subagent_type:
            case "explore":
                return [ReadTool(), GlobTool(), GrepTool(), ShellTool(readonly=True)]
            case "vmSetupHelper":
                return [ReadTool(), GlobTool(), GrepTool(), ShellTool(readonly=True)]
            case "generalPurpose":
                return self.all_tools  # 完整工具集
            case "debug":
                return self.all_tools + [InstrumentationTool()]
            case "computerUse":
                return [ComputerTool(), ScreenshotTool()]
            case "videoReview":
                return [VideoAnalysisTool()]
```

---

## 14. MCP 工具详细设计

```python
class ListMcpResourcesTool(BaseTool):
    """列出 MCP 服务器资源"""

    def execute(self, server: str = None) -> str:
        resources = []
        for mcp_server in self.mcp_servers:
            if server and mcp_server.name != server:
                continue
            server_resources = mcp_server.list_resources()
            for r in server_resources:
                r["server"] = mcp_server.name
                resources.append(r)

        return json.dumps(resources, indent=2)


class FetchMcpResourceTool(BaseTool):
    """获取 MCP 资源"""

    def execute(
        self,
        server: str,
        uri: str,
        downloadPath: str = None,
    ) -> str:
        mcp_server = self.get_server(server)
        content = mcp_server.read_resource(uri)

        if downloadPath:
            abs_path = os.path.join(self.workspace_root, downloadPath)
            with open(abs_path, "wb") as f:
                f.write(content)
            return f"Resource saved to {abs_path}"
        else:
            return content.decode("utf-8", errors="replace")
```

---

## 15. 工具安全约束汇总

| 工具 | 安全约束 |
|------|----------|
| Shell | 禁止 pkill -f；输出脱敏；超时限制 |
| Read | 文件大小限制；路径限制 |
| Write | 优先编辑不创建；不生成二进制 |
| StrReplace | 唯一性检查；不允许相同的新旧字符串 |
| Delete | 安全路径检查 |
| Grep | 输出行数上限 |
| WebSearch | 结果安全过滤 |
| Task | 最大并发 4；子代理上下文隔离 |
| RecordScreen | 仅用于 GUI 测试 |
| SetupVmEnvironment | 禁止危险命令 |
