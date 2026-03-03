# 模块十：Git 集成与代码管理详细设计

> 本文档详细描述 Agent 的 Git 操作策略、提交管理、分支策略、GitHub CLI 集成、PR 自动化等。

---

## 1. Git 操作架构

### 1.1 组件架构

```
┌──────────────────────────────────────────────────────┐
│                  Git Integration Layer                 │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Git Client   │  │ Commit       │  │ Branch     │ │
│  │ (libgit2/    │  │ Manager      │  │ Manager    │ │
│  │  shell git)  │  │              │  │            │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬─────┘ │
│         │                 │                  │       │
│  ┌──────▼─────────────────▼──────────────────▼─────┐ │
│  │              Workspace Manager                    │ │
│  │  - Clone / Fetch / Pull                           │ │
│  │  - Diff tracking                                  │ │
│  │  - Conflict detection                             │ │
│  └──────────────────────────────────────────────────┘ │
│                                                      │
│  ┌──────────────────────────────────────────────────┐ │
│  │              GitHub CLI (Read-Only)                │ │
│  │  - PR 查看                                        │ │
│  │  - CI 日志查看                                    │ │
│  │  - Issue 查看                                     │ │
│  └──────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

---

## 2. Git 操作策略

### 2.1 操作权限矩阵

| Git 操作 | Agent 可执行 | 条件 |
|----------|-------------|------|
| `git status` | ✅ 总是 | - |
| `git diff` | ✅ 总是 | - |
| `git log` | ✅ 总是 | - |
| `git add` | ✅ 总是 | - |
| `git commit` | ✅ 总是 | 有描述性消息 |
| `git push` | ✅ 总是 | 当前分支 |
| `git pull` | ✅ 总是 | 当前分支 |
| `git fetch` | ✅ 总是 | - |
| `git checkout` | ⚠️ 受限 | 仅当用户明确要求 |
| `git branch` (创建) | ⚠️ 受限 | 仅当用户明确要求 |
| `git push --force` | ❌ 默认禁止 | 仅当用户明确要求 |
| `git commit --amend` | ❌ 默认禁止 | 仅当用户明确要求 |
| `git rebase` | ❌ 默认禁止 | 仅当用户明确要求 |
| `git reset --hard` | ❌ 默认禁止 | 仅当用户明确要求 |

### 2.2 提交策略

```python
class CommitManager:
    """Git 提交管理"""

    def create_commit(self, changes: List[FileChange], message: str):
        """创建一个提交"""

        # 规则 1: 每个逻辑变更一个提交
        # 规则 2: 不批量提交（除非用户要求）
        # 规则 3: 描述性提交消息

        # 暂存文件
        for change in changes:
            self.git.add(change.path)

        # 验证提交消息
        self._validate_message(message)

        # 创建提交
        self.git.commit(message)

    def _validate_message(self, message: str):
        """验证提交消息质量"""
        if len(message) < 10:
            raise ValueError("Commit message too short")
        if message.startswith("WIP") or message == "update":
            raise ValueError("Commit message not descriptive enough")

    # 好的提交消息示例:
    # "Add user authentication middleware with JWT validation"
    # "Fix race condition in WebSocket connection handler"
    # "Refactor database queries to use connection pooling"
    # "Update README with development setup instructions"

    # 坏的提交消息示例:
    # "fix"
    # "update"
    # "WIP"
    # "changes"
```

### 2.3 提交粒度指南

```
场景                              推荐提交数

修复单个 Bug:                     1 个提交
  "Fix null pointer in UserService.getProfile()"

新增 Feature (小):                 1-2 个提交
  "Add password reset API endpoint"
  "Add password reset email template"

新增 Feature (大):                 3-5 个提交
  "Add database schema for notifications"
  "Implement notification service"
  "Add notification API endpoints"
  "Add notification UI components"
  "Add notification integration tests"

环境设置:                          1-2 个提交
  "Add AGENTS.md with development setup instructions"
  "Configure Docker Compose for local development"
```

---

## 3. 工作区管理

### 3.1 初始化流程

```python
class WorkspaceManager:
    """工作区管理"""

    def initialize(self, repo_url: str, branch: str):
        """初始化工作区"""

        if os.path.exists("/workspace/.git"):
            # 已存在: 更新
            self.git.fetch("origin")
            self.git.checkout(branch)
            self.git.pull("origin", branch)
        else:
            # 不存在: 克隆
            self.git.clone(repo_url, "/workspace")
            self.git.checkout(branch)

    def get_status_snapshot(self) -> str:
        """获取 Git 状态快照（注入到 Agent 上下文）"""
        branch = self.git.current_branch()
        status = self.git.status()
        return f"""Git repo: /workspace

## {branch}
{status}"""
```

### 3.2 变更追踪

```python
class ChangeTracker:
    """追踪 Agent 的代码变更"""

    def __init__(self):
        self.initial_commit = self.git.head()
        self.committed_changes: List[CommitInfo] = []

    def track_commit(self, commit_hash: str):
        """记录 Agent 创建的提交"""
        self.committed_changes.append(CommitInfo(
            hash=commit_hash,
            message=self.git.log(commit_hash, n=1),
            timestamp=datetime.utcnow(),
            diff_stat=self.git.diff_stat(f"{commit_hash}~1", commit_hash),
        ))

    def get_total_diff(self) -> str:
        """获取 Agent 所有变更的总 diff"""
        return self.git.diff(self.initial_commit, "HEAD")

    def get_changed_files(self) -> List[str]:
        """获取 Agent 变更的所有文件"""
        return self.git.diff_name_only(self.initial_commit, "HEAD")
```

---

## 4. GitHub CLI 集成

### 4.1 只读操作

```python
class GitHubCLI:
    """GitHub CLI 只读集成"""

    # gh CLI 已预认证

    def view_pr(self, pr_number: int = None) -> str:
        """查看 PR 详情"""
        cmd = ["gh", "pr", "view"]
        if pr_number:
            cmd.append(str(pr_number))
        return self.exec(cmd)

    def list_runs(self, limit: int = 10) -> str:
        """列出 CI 运行"""
        return self.exec([
            "gh", "run", "list",
            "--limit", str(limit),
        ])

    def view_run_log(self, run_id: str) -> str:
        """查看 CI 运行日志"""
        return self.exec([
            "gh", "run", "view", run_id,
            "--log",
        ])

    def view_issue(self, issue_number: int) -> str:
        """查看 Issue"""
        return self.exec([
            "gh", "issue", "view", str(issue_number),
        ])

    def list_prs(self, state: str = "open") -> str:
        """列出 PR"""
        return self.exec([
            "gh", "pr", "list",
            "--state", state,
        ])

    # 以下操作被禁止:
    # gh pr create  ← 由系统自动处理
    # gh pr merge   ← 由系统自动处理
    # gh issue create ← 不允许
    # gh repo create  ← 不允许
```

### 4.2 典型用例

```
Agent 使用 gh CLI 的场景:

1. 查看 CI 失败日志:
   gh run view 12345 --log-failed

2. 查看相关 PR 的讨论:
   gh pr view 42 --comments

3. 查看 Issue 详情（用户引用的 bug）:
   gh issue view 99

4. 检查 CI 状态:
   gh run list --branch current-branch --limit 5
```

---

## 5. PR 自动化

### 5.1 PR 创建流程

```
Agent 推送代码到分支
    │
    ▼
┌──────────────────┐
│ 系统检测到推送    │  Webhook / 轮询
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 自动创建/更新 PR  │  系统处理, 非 Agent
└────────┬─────────┘
         ▼
┌──────────────────┐
│ PR 包含:          │
│ - Agent 对话摘要   │
│ - 变更文件列表     │
│ - 测试结果        │
│ - 产物链接        │
└──────────────────┘
```

### 5.2 Agent 的分支策略

```
仓库分支策略:

main ─────────────────────────────────────────▶
    │
    └── cursor/task-name-xxxxx ────────────────▶ (Agent 工作分支)
        │
        ├── commit: "Add feature A"
        ├── commit: "Add tests for feature A"
        └── commit: "Fix lint issues"

Agent 始终在当前分支上工作 (不主动切换)
分支名通常由系统预创建: cursor/{task-description}-{hash}
```

---

## 6. 冲突处理

### 6.1 冲突检测

```python
class ConflictDetector:
    """冲突检测"""

    def check_before_push(self) -> Optional[ConflictInfo]:
        """推送前检查冲突"""
        # 获取远程最新
        self.git.fetch("origin")

        # 检查是否有新提交
        behind = self.git.rev_list_count(f"HEAD..origin/{self.branch}")
        if behind == 0:
            return None  # 无冲突

        # 尝试 merge（dry run）
        try:
            self.git.merge("origin/" + self.branch, no_commit=True, no_ff=True)
            self.git.merge_abort()
            return ConflictInfo(type="auto_resolvable", behind=behind)
        except MergeConflictError as e:
            self.git.merge_abort()
            return ConflictInfo(type="conflict", files=e.conflicting_files)
```

### 6.2 冲突解决策略

| 冲突类型 | 策略 |
|---------|------|
| 无冲突 | 直接 push |
| 可自动合并 | pull --rebase（如果允许） |
| 有冲突 | 报告给用户, 不自动解决 |

---

## 7. 安全约束

### 7.1 敏感文件保护

```python
class GitSecurity:
    """Git 安全约束"""

    PROTECTED_FILES = [
        ".env",
        ".env.local",
        "*.pem",
        "*.key",
        "id_rsa",
        "credentials.json",
    ]

    def pre_commit_check(self, staged_files: List[str]) -> List[Warning]:
        """提交前安全检查"""
        warnings = []

        for file in staged_files:
            # 检查敏感文件
            for pattern in self.PROTECTED_FILES:
                if fnmatch.fnmatch(file, pattern):
                    warnings.append(Warning(
                        f"Potentially sensitive file staged: {file}"
                    ))

            # 检查大文件
            size = os.path.getsize(file)
            if size > 10_000_000:  # 10MB
                warnings.append(Warning(
                    f"Large file staged: {file} ({size / 1_000_000:.1f}MB)"
                ))

        return warnings
```

### 7.2 Git Hooks 集成

```python
class GitHooksIntegration:
    """Git Hooks 集成"""

    def discover_hooks(self) -> List[HookInfo]:
        """发现项目中的 Git Hooks"""
        hooks = []

        # .husky/
        for hook_file in glob.glob(".husky/*"):
            hooks.append(HookInfo(
                source="husky",
                hook_type=os.path.basename(hook_file),
                path=hook_file,
            ))

        # .pre-commit-config.yaml
        if os.path.exists(".pre-commit-config.yaml"):
            hooks.append(HookInfo(
                source="pre-commit",
                hook_type="pre-commit",
                path=".pre-commit-config.yaml",
            ))

        # lint-staged in package.json
        pkg = json.load(open("package.json"))
        if "lint-staged" in pkg:
            hooks.append(HookInfo(
                source="lint-staged",
                hook_type="pre-commit",
                config=pkg["lint-staged"],
            ))

        return hooks
```
