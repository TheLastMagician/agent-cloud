# 模块十八：环境设置工作流深度设计

> 本文档极致深入地描述环境设置工作流的每一个步骤，包括并行发现机制、
> Hello World 任务概念、update script vs AGENTS.md 双层设计哲学、
> 以及各类项目的具体处理策略。

---

## 1. 环境设置的触发条件

```
触发场景:
├── 用户明确请求: "设置开发环境"
├── 系统检测到新仓库 (无快照)
├── 管理员配置的自动设置任务
└── 快照过期或损坏后的重建
```

---

## 2. 并行发现阶段 (Phase 1)

### 2.1 发现任务编排

```
[同时启动, 不等待, 并行执行]

任务 1: vmSetupHelper 子代理 — 产品分析
    │
    ├── 扫描所有顶层目录
    ├── 读取核心配置文件:
    │   package.json, docker-compose.yml, Makefile,
    │   Dockerfile, requirements.txt, go.mod, Cargo.toml, ...
    ├── 分析 monorepo 结构 (如 lerna, turborepo, nx)
    ├── 识别所有服务/产品
    └── 返回:
        ├── 产品描述
        ├── 服务列表 (必须运行 vs 可选)
        └── 技术栈识别

任务 2: vmSetupHelper 子代理 — 设置脚本发现
    │
    ├── 搜索文件:
    │   init.sh, setup.sh, bootstrap.sh, install.sh,
    │   dev-setup.sh, Makefile (setup targets),
    │   scripts/setup.*, scripts/bootstrap.*
    ├── 搜索 devcontainer:
    │   .devcontainer/devcontainer.json,
    │   .devcontainer/Dockerfile,
    │   .devcontainer/*.sh
    ├── 搜索环境文件:
    │   docker-compose.yml, docker-compose.yaml,
    │   .env.example, .tool-versions, .node-version,
    │   .nvmrc, .python-version, .ruby-version
    └── 返回: 找到的文件路径列表

任务 3: Glob/Grep 搜索 — 文档指令
    │
    ├── README.md (仓库根)
    ├── CLAUDE.md / AGENTS.md
    ├── .claude/settings.json (session start hooks)
    ├── CONTRIBUTING.md
    ├── docs/**/*.md (开发设置指南)
    └── 返回: 文档路径列表

任务 4: Glob/Grep 搜索 — Git Hooks
    │
    ├── .husky/ 目录
    ├── .git/hooks/ (如果可读)
    ├── .pre-commit-config.yaml
    ├── package.json 中的 lint-staged 配置
    └── 返回: hook 配置信息
```

### 2.2 发现结果汇总

```python
@dataclass
class DiscoveryResult:
    """并行发现的汇总结果"""

    # 产品分析
    products: List[Product]
    services: List[Service]
    tech_stack: TechStack

    # 设置脚本
    setup_scripts: List[str]
    devcontainer: Optional[DevcontainerConfig]
    docker_compose: Optional[str]
    env_example: Optional[str]

    # 文档
    readme: Optional[str]
    contributing: Optional[str]
    agents_md: Optional[str]
    claude_md: Optional[str]
    docs_files: List[str]

    # Git Hooks
    husky_hooks: List[str]
    pre_commit_config: Optional[str]
    lint_staged: Optional[dict]

    # 环境检测
    node_version_file: Optional[str]   # .nvmrc / .node-version
    python_version_file: Optional[str]  # .python-version
    tool_versions: Optional[str]        # .tool-versions (asdf/mise)
```

---

## 3. TODO 列表创建 (Phase 2)

### 3.1 TODO 生成逻辑

```python
def generate_setup_todos(discovery: DiscoveryResult) -> List[TodoItem]:
    """基于发现结果生成 TODO 列表"""
    todos = []

    # 依赖安装 TODO 项
    if discovery.tech_stack.has_node:
        todos.append(TodoItem(
            id="install_node_deps",
            content="安装 Node.js 依赖",
            status="pending",
        ))

    if discovery.tech_stack.has_python:
        todos.append(TodoItem(
            id="install_python_deps",
            content="安装 Python 依赖",
            status="pending",
        ))

    if discovery.docker_compose:
        todos.append(TodoItem(
            id="start_docker_services",
            content="启动 Docker 服务 (数据库等)",
            status="pending",
        ))

    # 验证 TODO 项
    todos.append(TodoItem(id="run_lint", content="运行 lint 检查", status="pending"))
    todos.append(TodoItem(id="run_tests", content="运行自动化测试", status="pending"))
    todos.append(TodoItem(id="build_app", content="构建应用 (开发模式)", status="pending"))
    todos.append(TodoItem(id="run_app", content="运行应用", status="pending"))
    todos.append(TodoItem(id="hello_world", content="完成 Hello World 任务", status="pending"))
    todos.append(TodoItem(id="setup_vm", content="设置 VM 更新脚本", status="pending"))
    todos.append(TodoItem(id="update_agents_md", content="更新 AGENTS.md", status="pending"))

    return todos
```

---

## 4. 依赖安装阶段 (Phase 3)

### 4.1 运行时版本管理

```python
class RuntimeVersionManager:
    """管理语言运行时版本"""

    def setup_node(self, discovery: DiscoveryResult):
        """设置正确版本的 Node.js"""

        # 检测所需版本
        version = None

        if discovery.node_version_file:
            # .nvmrc 或 .node-version
            version = read_file(discovery.node_version_file).strip()
        elif discovery.tech_stack.node_engines:
            # package.json engines.node
            version = parse_semver_range(discovery.tech_stack.node_engines)

        if version:
            # 使用 nvm 安装指定版本
            shell(f"nvm install {version}")
            shell(f"nvm use {version}")
        else:
            # 使用默认版本
            shell("nvm use default")

        # 检查是否使用 mise/fnm（可能需要禁用 nvm）
        if os.path.exists(".tool-versions") or os.path.exists(".mise.toml"):
            # 可能需要 mise
            shell("mise install")

    def setup_python(self, discovery: DiscoveryResult):
        """设置正确版本的 Python"""
        if discovery.python_version_file:
            version = read_file(discovery.python_version_file).strip()
            # 使用 pyenv 或 deadsnakes PPA
```

### 4.2 包管理器选择

```python
class PackageManagerSelector:
    """根据 lockfile 选择正确的包管理器"""

    LOCKFILE_MAP = {
        "package-lock.json": ("npm", "npm install"),
        "yarn.lock": ("yarn", "yarn install"),
        "pnpm-lock.yaml": ("pnpm", "pnpm install"),
        "bun.lockb": ("bun", "bun install"),
    }

    PYTHON_MAP = {
        "requirements.txt": ("pip", "pip install -r requirements.txt"),
        "Pipfile.lock": ("pipenv", "pipenv install"),
        "poetry.lock": ("poetry", "poetry install"),
        "uv.lock": ("uv", "uv sync"),
        "pyproject.toml": ("pip", "pip install -e '.[dev]'"),
    }

    def detect(self, workspace: str) -> PackageManager:
        """检测包管理器"""

        # 先检查 lockfile
        for lockfile, (name, cmd) in self.LOCKFILE_MAP.items():
            if os.path.exists(os.path.join(workspace, lockfile)):
                return PackageManager(name=name, install_cmd=cmd)

        # 再检查 README 中的说明
        # ...

        # 默认 pnpm
        return PackageManager(name="pnpm", install_cmd="pnpm install")
```

### 4.3 处理交互式安装

```python
# 关键规则: 避免交互式命令
# 
# 问题示例:
#   pnpm approve-builds → 打开交互式 picker (会永久阻塞)
#
# 解决方案:
#   1. 使用非交互式替代:
#      - pnpm: 在 package.json 中配置 pnpm.onlyBuiltDependencies
#      - npm: npm install --yes
#      - pip: pip install --no-input
#
#   2. 如果无法避免交互:
#      - 请求用户操作 (env_setup_actions)
#      - 或使用 expect 脚本自动应答

NON_INTERACTIVE_FLAGS = {
    "npm": "--yes",
    "pnpm": "--reporter=append-only",
    "yarn": "--non-interactive",
    "pip": "--no-input",
    "apt-get": "-y",
    "brew": "--quiet",
}
```

---

## 5. Update Script 设计哲学

### 5.1 双层设计

```
┌────────────────────────────────────────────┐
│  Layer 1: Update Script                     │
│  (自动化层 — 每次 VM 启动时运行)              │
│                                            │
│  职责: 只做依赖刷新                          │
│  特性: 幂等、最小化、低风险、快速             │
│                                            │
│  ✅ 允许:                                   │
│    npm install                              │
│    pip install -r requirements.txt          │
│    pnpm install                             │
│    uv sync                                  │
│    bundle install                           │
│    go mod download                          │
│                                            │
│  ❌ 禁止:                                   │
│    docker compose up     (服务启动)          │
│    npm run dev           (开发服务器)         │
│    python manage.py runserver (启动)         │
│    npm run build         (构建)              │
│    npm test              (测试)              │
│    echo ... >> ~/.bashrc (shell 修改)        │
│    export FOO=bar        (环境变量)          │
│    python manage.py migrate (迁移)           │
│                                            │
│  鲁棒性要求:                                 │
│    - 不依赖未合并 PR 的文件                   │
│    - 对 /workspace 的当前状态有效             │
│    - 重复运行无副作用 (幂等)                  │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│  Layer 2: AGENTS.md                         │
│  (知识层 — 供未来 Agent 参考)                 │
│                                            │
│  职责: 非显而易见的操作指南                    │
│  特性: 持久、版本控制、人类可读                │
│                                            │
│  ✅ 应包含:                                  │
│    服务描述和端口                             │
│    启动顺序和依赖关系                         │
│    非显而易见的 gotchas                       │
│    对 README 的引用 (不复制)                  │
│                                            │
│  ❌ 不应包含:                                │
│    依赖安装步骤 (→ update_script)             │
│    一次性设置操作                             │
│    系统依赖安装                              │
│    已有文档的复制                             │
└────────────────────────────────────────────┘
```

### 5.2 Update Script 生成策略

```python
def generate_update_script(discovery: DiscoveryResult) -> str:
    """生成 update script"""
    lines = []

    # Node.js 依赖
    if discovery.tech_stack.has_node:
        pm = discovery.package_manager
        if pm == "pnpm":
            lines.append("pnpm install")
        elif pm == "yarn":
            lines.append("yarn install")
        elif pm == "npm":
            lines.append("npm install")
        elif pm == "bun":
            lines.append("bun install")

    # Python 依赖
    if discovery.tech_stack.has_python:
        if os.path.exists("uv.lock"):
            lines.append("uv sync")
        elif os.path.exists("requirements.txt"):
            lines.append("pip install -r requirements.txt")
        elif os.path.exists("pyproject.toml"):
            lines.append("pip install -e '.[dev]'")

    # Go 依赖
    if discovery.tech_stack.has_go:
        lines.append("go mod download")

    # Rust 依赖
    if discovery.tech_stack.has_rust:
        lines.append("cargo fetch")

    # Ruby 依赖
    if discovery.tech_stack.has_ruby:
        lines.append("bundle install")

    # 多行脚本用换行分隔 (不用 &&)
    return "\n".join(lines)
```

---

## 6. Hello World 任务设计

### 6.1 概念定义

```
Hello World 任务 = 一个涉及应用核心功能的端到端操作

目的:
├── 证明开发环境完全就绪
├── 验证应用可正常运行
├── 生成演示产物
└── 发现未预料的环境问题

规则:
├── 必须执行操作 (不只是加载页面)
├── 必须涉及核心功能
├── 必须产生可验证的结果
└── 通常需要启动应用 + GUI 交互
```

### 6.2 Hello World 任务示例库

```python
HELLO_WORLD_EXAMPLES = {
    # Web 应用
    "nextjs_app": {
        "steps": [
            "启动 dev server (npm run dev)",
            "打开浏览器导航到 localhost:3000",
            "注册一个测试账号",
            "完成一个核心操作 (如创建条目)",
        ],
        "evidence": "截图 + 录屏",
    },

    "express_api": {
        "steps": [
            "启动 API server (npm run dev)",
            "使用 curl 调用核心 API",
            "验证响应数据正确",
        ],
        "evidence": "终端输出",
    },

    "django_app": {
        "steps": [
            "启动 dev server (python manage.py runserver)",
            "打开浏览器导航到 admin 页面",
            "登录管理后台",
            "创建一条数据",
        ],
        "evidence": "截图 + 终端日志",
    },

    "cli_tool": {
        "steps": [
            "构建 CLI 工具",
            "运行 help 命令",
            "执行一个实际操作",
        ],
        "evidence": "终端输出",
    },

    "library": {
        "steps": [
            "构建库",
            "运行测试套件",
            "验证示例代码可运行",
        ],
        "evidence": "测试输出",
    },

    "mobile_app": {
        "steps": [
            "启动 Metro/Expo dev server",
            "在模拟器中启动应用",
            "完成一个核心操作",
        ],
        "evidence": "截图",
    },
}
```

### 6.3 Hello World 执行流程

```
识别应用类型
    │
    ├── Web 前端?
    │   ├── 启动 dev server
    │   ├── computerUse: 打开浏览器
    │   ├── 开始录屏
    │   ├── computerUse: 执行核心操作
    │   ├── 保存录屏
    │   └── 截屏最终状态
    │
    ├── API 服务?
    │   ├── 启动 server
    │   ├── Shell: curl 调用核心 API
    │   ├── 验证响应
    │   └── 保存日志产物
    │
    ├── CLI 工具?
    │   ├── 构建工具
    │   ├── Shell: 执行核心命令
    │   ├── 验证输出
    │   └── 保存终端输出
    │
    └── 库?
        ├── 构建
        ├── 运行测试
        └── 保存测试输出
```

---

## 7. 验证阶段 (Phase 5)

### 7.1 验证清单

```
┌─────────────────────────────────────────────────┐
│                验证矩阵                          │
│                                                 │
│  检查项          │ 命令             │ 必须通过?  │
│ ─────────────────┼──────────────────┼──────────│
│  Lint            │ npm run lint     │ ✅ 是    │
│  类型检查        │ npm run typecheck│ ✅ 是    │
│  单元测试        │ npm run test     │ ✅ 是    │
│  构建 (dev)      │ npm run dev      │ ✅ 是    │
│  应用运行        │ 页面可访问       │ ✅ 是    │
│  Hello World     │ 核心操作完成     │ ✅ 是    │
│  E2E 测试        │ npm run e2e      │ ⚠️ 尽力 │
│  性能测试        │ npm run perf     │ ❌ 跳过  │
└─────────────────────────────────────────────────┘
```

### 7.2 失败处理

```
验证失败
    │
    ├── lint 失败
    │   └── 可能是环境问题 (缺少 eslint 插件)
    │       → 安装缺失依赖 → 重试
    │
    ├── 测试失败
    │   ├── 全部失败 → 可能是环境配置问题
    │   │   → 检查 .env, 数据库连接等
    │   └── 部分失败 → 可能是已有的失败测试
    │       → 记录但不阻塞
    │
    ├── 构建失败
    │   └── 可能是 Node 版本、依赖缺失
    │       → 调整版本 → 重试
    │
    ├── 应用无法启动
    │   ├── 端口占用 → 杀掉旧进程
    │   ├── 缺少环境变量 → 请求 Secrets
    │   └── 缺少数据库 → 启动 Docker 服务
    │
    └── Hello World 无法完成
        ├── 需要登录账号 → 请求测试账号 (add_test_login)
        ├── 需要外部服务 → 请求用户操作 (external_action)
        └── 功能 Bug → 记录并报告 (不修复)
```

---

## 8. 各类项目的处理策略

### 8.1 策略矩阵

| 项目类型 | 依赖安装 | 服务启动 | Hello World |
|---------|---------|---------|-------------|
| React/Next.js 前端 | pnpm install | pnpm dev | 浏览器操作 |
| Express/Fastify API | npm install | npm run dev | curl 调用 |
| Django/Flask | pip install | python manage.py runserver | 浏览器+Admin |
| Go 服务 | go mod download | go run . | curl 调用 |
| Rust 服务 | cargo fetch | cargo run | curl/CLI |
| monorepo (turbo) | pnpm install | turbo dev | 多服务测试 |
| Docker Compose 项目 | docker compose build | docker compose up | 依项目而定 |
| CLI 工具 | 按语言 | N/A | 运行命令 |
| 库 | 按语言 | N/A | 运行测试 |

### 8.2 特殊场景处理

```python
# Docker 需要特殊处理 (Firecracker 限制)
if needs_docker:
    # 1. 安装 Docker (fuse-overlayfs + iptables-legacy)
    # 2. 启动 dockerd
    # 3. docker compose up -d

# monorepo 需要特殊处理
if is_monorepo:
    # 1. 识别所有 packages/services
    # 2. 安装根依赖
    # 3. 安装各包依赖
    # 4. 按依赖顺序启动服务

# 需要数据库种子数据
if needs_seed:
    # 1. 启动数据库服务
    # 2. 运行迁移 (migrate)
    # 3. 运行种子 (seed)

# 需要 Secrets
if needs_secrets:
    # 1. 检查环境变量
    # 2. 缺失 → 生成 env_setup_actions XML
    # 3. 有 .env.example → 复制为 .env
```

---

## 9. 最终输出格式

### 9.1 成功完成的回复模板

```markdown
**Walkthrough**
<video src="/opt/cursor/artifacts/demo_hello_world.mp4" controls></video>
完成了用户注册和首次操作。

**Summary**
* 安装了 Node.js 20 + pnpm
* 安装了项目依赖 (pnpm install)
* 配置了本地开发环境
* 添加了 `AGENTS.md` Cursor Cloud 指南

| 服务 | lint | test | build | run |
|------|------|------|-------|-----|
| 前端 (Next.js) | ✅ `pnpm lint` | ✅ `pnpm test` | ✅ `pnpm dev` | ✅ localhost:3000 |
| 后端 (Express) | ✅ `pnpm lint` | ✅ `pnpm test` | ✅ `pnpm dev` | ✅ localhost:8080 |

**Testing**
* ✅ `pnpm lint`
* ✅ `pnpm test`
* ✅ `pnpm dev` — 前端启动成功
* ✅ Hello World: 创建了测试用户并发送消息 (见视频)
```

### 9.2 被阻塞的回复模板

```markdown
**Summary**
* 安装了依赖, lint 和 test 通过
* 应用启动需要以下 Secrets

**Testing**
* ✅ `pnpm lint`
* ✅ `pnpm test`
* ⚠️ `pnpm dev` — 需要 DATABASE_URL

<env_setup_actions version="1">
  <add_secrets>
    <secret name="DATABASE_URL" />
    <secret name="API_KEY" />
    <reason><![CDATA[
需要数据库连接和 API 密钥才能启动开发服务器和完成 Hello World 任务。
    ]]></reason>
  </add_secrets>
</env_setup_actions>
```
