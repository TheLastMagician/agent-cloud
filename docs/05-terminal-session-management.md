# 模块五：终端会话管理详细设计

> 本文档详细描述终端会话的创建、状态持久化、文件同步、多会话管理等。

---

## 1. 设计理念

### 1.1 终端即文件

核心设计决策：**将终端会话状态建模为文本文件**。

```
传统方案:                          本方案:
Terminal Emulator (UI)            文本文件 (Text File)
    │                                 │
    ├── PTY 管理                     ├── 元数据头部 (YAML front matter)
    ├── ANSI 解析                    ├── 纯文本输出
    ├── 滚动缓冲区                   ├── Agent 可直接 Read/Grep
    └── WebSocket 流                 └── 自动同步更新
```

**优势**：
- Agent 可用 Read 工具直接读取终端输出
- Agent 可用 Grep 工具搜索终端内容
- 无需复杂的终端模拟器协议
- 多终端会话轻量管理

### 1.2 文件系统布局

```
~/.cursor/projects/workspace/terminals/
├── 1.txt          # 终端会话 1
├── 2.txt          # 终端会话 2
├── 3.txt          # 终端会话 3
└── ...
```

---

## 2. 终端文件格式

### 2.1 文件结构

```yaml
---
pid: 68861                    # 会话进程 PID
cwd: /workspace/src           # 当前工作目录
last_command: npm run test     # 最近执行的命令
last_exit_code: 0              # 最近命令退出码
is_running: true               # 是否有活跃命令正在运行
started_at: 2026-03-03T10:30:00Z  # 会话启动时间
---
$ cd /workspace/src
$ npm run test

> project@1.0.0 test
> jest --coverage

 PASS  src/utils.test.js
 PASS  src/api.test.js

Test Suites: 2 passed, 2 total
Tests:       15 passed, 15 total

$ _
```

### 2.2 元数据字段定义

| 字段 | 类型 | 说明 |
|------|------|------|
| pid | int | Shell 进程 PID |
| cwd | string | 当前工作目录（实时更新） |
| last_command | string | 最近执行的命令 |
| last_exit_code | int | 最近命令的退出码 |
| is_running | bool | 当前是否有命令在执行 |
| started_at | datetime | 会话启动时间 |

---

## 3. 终端会话管理器

### 3.1 核心实现

```python
import os
import pty
import select
import subprocess
from dataclasses import dataclass, field
from typing import Dict, Optional
from datetime import datetime

@dataclass
class TerminalSession:
    """终端会话"""
    id: str
    pid: int
    master_fd: int          # PTY 主端文件描述符
    cwd: str = "/workspace"
    last_command: str = ""
    last_exit_code: int = 0
    is_running: bool = False
    started_at: datetime = field(default_factory=datetime.utcnow)
    output_buffer: str = ""  # 完整输出缓冲区


class TerminalManager:
    """终端会话管理器"""

    TERMINALS_DIR = "/home/ubuntu/.cursor/projects/workspace/terminals"
    MAX_SESSIONS = 10
    MAX_OUTPUT_SIZE = 500_000  # 500KB 输出缓冲上限

    def __init__(self):
        self.sessions: Dict[str, TerminalSession] = {}
        os.makedirs(self.TERMINALS_DIR, exist_ok=True)

    def create_session(self, session_id: str = None) -> TerminalSession:
        """创建新终端会话"""
        session_id = session_id or str(len(self.sessions) + 1)

        # 创建 PTY
        master_fd, slave_fd = pty.openpty()

        # 启动 bash 进程
        proc = subprocess.Popen(
            ["bash", "--norc", "--noprofile"],
            stdin=slave_fd,
            stdout=slave_fd,
            stderr=slave_fd,
            preexec_fn=os.setsid,
            cwd="/workspace",
            env=self._get_env(),
        )

        session = TerminalSession(
            id=session_id,
            pid=proc.pid,
            master_fd=master_fd,
        )

        self.sessions[session_id] = session
        self._sync_to_file(session)

        # 启动输出读取线程
        self._start_output_reader(session)

        return session

    def execute_command(
        self,
        session_id: str,
        command: str,
        timeout: int = 30000,
    ) -> str:
        """在指定会话中执行命令"""
        session = self.sessions[session_id]

        # 更新元数据
        session.last_command = command
        session.is_running = True
        self._sync_to_file(session)

        # 发送命令到 PTY
        os.write(session.master_fd, f"{command}\n".encode())

        # 等待命令完成
        output = self._wait_for_completion(session, timeout)

        # 更新状态
        session.is_running = False
        session.last_exit_code = self._get_exit_code(session)
        session.cwd = self._get_cwd(session)
        self._sync_to_file(session)

        return output

    def _start_output_reader(self, session: TerminalSession):
        """启动后台线程读取终端输出"""
        import threading

        def reader():
            while True:
                try:
                    readable, _, _ = select.select(
                        [session.master_fd], [], [], 1.0
                    )
                    if readable:
                        data = os.read(session.master_fd, 4096)
                        if not data:
                            break
                        text = data.decode("utf-8", errors="replace")

                        # ANSI 转义码清理
                        text = self._strip_ansi(text)

                        # 追加到缓冲区
                        session.output_buffer += text

                        # 缓冲区大小限制
                        if len(session.output_buffer) > self.MAX_OUTPUT_SIZE:
                            session.output_buffer = (
                                "[... output truncated ...]\n"
                                + session.output_buffer[-self.MAX_OUTPUT_SIZE:]
                            )

                        # 同步到文件
                        self._sync_to_file(session)
                except OSError:
                    break

        thread = threading.Thread(target=reader, daemon=True)
        thread.start()

    def _sync_to_file(self, session: TerminalSession):
        """同步终端状态到文件"""
        filepath = os.path.join(self.TERMINALS_DIR, f"{session.id}.txt")

        metadata = f"""---
pid: {session.pid}
cwd: {session.cwd}
last_command: {session.last_command}
last_exit_code: {session.last_exit_code}
is_running: {str(session.is_running).lower()}
started_at: {session.started_at.isoformat()}
---
"""
        content = metadata + session.output_buffer

        with open(filepath, "w") as f:
            f.write(content)

    def _strip_ansi(self, text: str) -> str:
        """去除 ANSI 转义序列"""
        import re
        ansi_escape = re.compile(r'\x1B(?:[@-Z\\-_]|\[[0-?]*[ -/]*[@-~])')
        return ansi_escape.sub('', text)

    def _get_cwd(self, session: TerminalSession) -> str:
        """获取当前工作目录"""
        try:
            return os.readlink(f"/proc/{session.pid}/cwd")
        except (OSError, FileNotFoundError):
            return session.cwd

    def _get_exit_code(self, session: TerminalSession) -> int:
        """获取最后命令的退出码"""
        os.write(session.master_fd, b"echo $?\n")
        # 读取并解析输出
        # ...
        return 0

    def get_all_metadata(self) -> Dict[str, dict]:
        """获取所有终端的元数据"""
        result = {}
        for session_id, session in self.sessions.items():
            result[session_id] = {
                "pid": session.pid,
                "cwd": session.cwd,
                "last_command": session.last_command,
                "last_exit_code": session.last_exit_code,
                "is_running": session.is_running,
            }
        return result
```

---

## 4. Agent 与终端的交互模式

### 4.1 Agent 查看终端状态

```python
# Agent 通过 Shell 工具查看所有终端元数据
# 命令: head -n 10 ~/.cursor/projects/workspace/terminals/*.txt

# Agent 通过 Read 工具读取特定终端的完整输出
# Read(path="/home/ubuntu/.cursor/projects/workspace/terminals/1.txt")

# Agent 通过 Grep 工具搜索终端输出
# Grep(pattern="ERROR", path="/home/ubuntu/.cursor/projects/workspace/terminals/")
```

### 4.2 防止重复启动

```python
# Agent 在启动长期进程前的检查逻辑:
#
# 1. 读取所有终端文件的元数据
# 2. 检查是否已有相同进程在运行
# 3. 如果已存在, 复用而非重启

def check_process_running(process_name: str) -> bool:
    """检查进程是否已在某个终端中运行"""
    for terminal_file in glob.glob(f"{TERMINALS_DIR}/*.txt"):
        with open(terminal_file) as f:
            content = f.read()
            if f"last_command: {process_name}" in content:
                if "is_running: true" in content:
                    return True
    return False
```

---

## 5. 终端输出处理

### 5.1 ANSI 转义码处理

| 转义码类别 | 处理方式 |
|-----------|---------|
| 颜色代码 (如 `\033[31m`) | 移除 |
| 光标移动 (如 `\033[H`) | 移除 |
| 清屏 (如 `\033[2J`) | 移除 |
| 进度条 (`\r` 覆写) | 保留最终行 |
| Unicode | 保留 |

### 5.2 输出缓冲区管理

```
输出增长策略:
  新输出 → 追加到 buffer → 检查大小
                              │
                     ┌────────┤
                     │        │
              < 500KB     > 500KB
                │              │
           正常追加       截断头部
                         保留最近 500KB
                         添加 "[... truncated ...]"
```
