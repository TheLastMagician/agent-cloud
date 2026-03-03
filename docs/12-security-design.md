# 模块十二：安全体系详细设计

> 本文档详细描述 Cloud Agent 的安全架构，包括威胁模型、沙箱隔离、输入/输出安全、权限控制等。

---

## 1. 威胁模型

### 1.1 攻击面分析

```
┌─────────────────────────────────────────────────────────────┐
│                      攻击面地图                               │
│                                                             │
│  用户输入 ──▶ [1. Prompt 注入]                               │
│                                                             │
│  代码仓库 ──▶ [2. 恶意代码执行]                               │
│                                                             │
│  网络搜索 ──▶ [3. 搜索结果操纵]                               │
│                                                             │
│  工具调用 ──▶ [4. 命令注入]                                   │
│                                                             │
│  Secret 值 ──▶ [5. 密钥泄露]                                 │
│                                                             │
│  VM 环境 ──▶ [6. 沙箱逃逸]                                   │
│                                                             │
│  Agent 输出 ──▶ [7. 敏感信息泄露]                             │
│                                                             │
│  子代理 ──▶ [8. 权限提升]                                    │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 威胁与对策矩阵

| 威胁 | 风险等级 | 攻击向量 | 对策 |
|------|---------|---------|------|
| Prompt 注入 | 高 | 恶意用户指令 | 系统提示优先级 > AGENTS.md > 用户指令 |
| 恶意代码执行 | 高 | 仓库中的恶意脚本 | VM 沙箱隔离 |
| 搜索结果操纵 | 中 | 搜索结果中的恶意指令 | 优先用户原始请求 |
| 命令注入 | 中 | Shell 命令拼接 | 参数校验，禁止 pkill -f |
| 密钥泄露 | 高 | 输出/提交中包含密钥 | 输出脱敏，提交扫描 |
| 沙箱逃逸 | 严重 | 利用 VM 漏洞 | Firecracker 硬件隔离 |
| 敏感信息泄露 | 中 | Agent 输出包含敏感信息 | 输出过滤，Secret 脱敏 |
| 权限提升 | 中 | 子代理获取额外权限 | 工具集限制，上下文隔离 |

---

## 2. 沙箱隔离详细设计

### 2.1 三层防御

```
┌─────────────────────────────────────────┐
│  Layer 1: Firecracker MicroVM (硬件隔离) │
│                                         │
│  ├── 独立内核                            │
│  ├── 独立文件系统                         │
│  ├── 独立网络命名空间                     │
│  ├── CPU/内存资源限制                     │
│  └── 最小设备模型（减少攻击面）            │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │ Layer 2: Docker Container        │   │
│  │                                  │   │
│  │ ├── 进程隔离 (PID namespace)     │   │
│  │ ├── 文件系统隔离                  │   │
│  │ ├── 非 root 用户 (ubuntu)        │   │
│  │ ├── Capability 限制              │   │
│  │ └── Seccomp 过滤                 │   │
│  │                                  │   │
│  │ ┌──────────────────────────┐     │   │
│  │ │ Layer 3: Agent Process   │     │   │
│  │ │                          │     │   │
│  │ │ ├── 工具调用约束          │     │   │
│  │ │ ├── 禁止 pkill -f        │     │   │
│  │ │ ├── 文件路径限制          │     │   │
│  │ │ ├── 网络出站控制          │     │   │
│  │ │ └── 超时限制              │     │   │
│  │ └──────────────────────────┘     │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### 2.2 Firecracker 安全配置

```python
class FirecrackerSecurityConfig:
    """Firecracker 安全配置"""

    config = {
        # 最小设备模型
        "devices": {
            "virtio_net": True,    # 网络
            "virtio_blk": True,    # 块存储
            "serial": True,        # 串口
            # 无 GPU、USB、PCI passthrough 等
        },

        # 资源限制
        "rate_limiters": {
            "bandwidth": {
                "one_time_burst": 1048576,    # 1MB burst
                "refill_time": 1000,           # 1s
                "size": 10485760,              # 10MB/s
            },
            "ops": {
                "one_time_burst": 1000,
                "refill_time": 1000,
                "size": 5000,                  # 5000 IOPS
            },
        },

        # Jailer 配置（额外的安全层）
        "jailer": {
            "chroot_base": "/srv/jailer",
            "uid": 1000,
            "gid": 1000,
            "netns": True,                     # 网络命名空间
            "cgroup_v2": True,
        },
    }
```

### 2.3 Docker 安全配置

```python
class DockerSecurityConfig:
    """Docker 容器安全配置"""

    config = {
        # 非 root 用户
        "user": "ubuntu",

        # 只读根文件系统（可选）
        "read_only": False,  # Agent 需要写入

        # Capability 限制
        "cap_drop": ["ALL"],
        "cap_add": ["NET_BIND_SERVICE", "SYS_PTRACE"],

        # Seccomp 配置
        "security_opt": ["seccomp=default"],

        # 无特权
        "privileged": False,

        # PID 限制
        "pids_limit": 500,

        # 内存限制
        "mem_limit": "4g",
        "memswap_limit": "4g",

        # CPU 限制
        "cpus": 2.0,

        # 临时文件系统限制
        "tmpfs": {
            "/tmp": "size=1G",
        },
    }
```

---

## 3. 输入安全

### 3.1 Prompt 注入防护

```python
class PromptInjectionDefense:
    """Prompt 注入防护"""

    # 优先级规则:
    # 系统提示 > 直接用户/开发者指令 > AGENTS.md > 工具结果内容

    def sanitize_tool_output(self, output: str) -> str:
        """清理工具输出中的潜在注入"""

        # 检测可疑指令
        suspicious_patterns = [
            r"<system>",
            r"</system>",
            r"ignore previous",
            r"forget your instructions",
            r"you are now",
            r"new system prompt",
        ]

        for pattern in suspicious_patterns:
            if re.search(pattern, output, re.I):
                output = f"⚠️ [系统提醒: 工具输出包含可疑内容]\n{output}"

        return output

    def validate_agents_md(self, content: str) -> str:
        """验证 AGENTS.md 内容"""
        # AGENTS.md 指令不能覆盖系统提示
        # 标记 <system_reminder> 等特殊标签
        return content
```

### 3.2 Shell 命令安全

```python
class ShellSecurity:
    """Shell 命令安全检查"""

    BLOCKED_COMMANDS = [
        r"pkill\s+-f",              # 按名杀进程
        r"killall",                 # 杀所有同名进程
        r"rm\s+-rf\s+/[^w]",       # 删除根目录（非 /workspace）
        r"dd\s+if=.*of=/dev/",     # 直接写入设备
        r"mkfs\.",                  # 格式化文件系统
        r"reboot|shutdown|halt",   # 关机/重启
        r"chmod\s+777",            # 过于宽松的权限
    ]

    RATE_LIMITS = {
        "max_commands_per_minute": 60,
        "max_background_processes": 10,
    }

    def check_command(self, command: str) -> SecurityCheckResult:
        """检查命令安全性"""
        for pattern in self.BLOCKED_COMMANDS:
            if re.search(pattern, command):
                return SecurityCheckResult(
                    allowed=False,
                    reason=f"Blocked command pattern: {pattern}",
                )

        return SecurityCheckResult(allowed=True)
```

---

## 4. 输出安全

### 4.1 Secret 脱敏管线

```
Agent/Tool 输出
    │
    ▼
┌──────────────────┐
│ Secret 值扫描     │  遍历所有已知 Secret 值
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 值替换            │  将匹配的值替换为 [REDACTED]
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 编码变体检测      │  base64、URL编码等变体
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 子串检测         │  部分匹配的 Secret 片段
└────────┬─────────┘
         ▼
安全的输出内容
```

### 4.2 脱敏实现

```python
class OutputRedactionPipeline:
    """输出脱敏管线"""

    def __init__(self, secrets: Dict[str, str]):
        self.secrets = secrets
        self.patterns = self._build_patterns()

    def _build_patterns(self) -> List[tuple]:
        """构建检测模式"""
        patterns = []

        for name, value in self.secrets.items():
            if not value or len(value) < 4:
                continue

            # 原始值
            patterns.append((value, f"[REDACTED:{name}]"))

            # base64 编码
            b64 = base64.b64encode(value.encode()).decode()
            patterns.append((b64, f"[REDACTED:{name}:base64]"))

            # URL 编码
            url_encoded = urllib.parse.quote(value)
            patterns.append((url_encoded, f"[REDACTED:{name}:url]"))

        # 按长度降序排序（先匹配长字符串）
        patterns.sort(key=lambda p: len(p[0]), reverse=True)

        return patterns

    def redact(self, text: str) -> str:
        """执行脱敏"""
        for pattern, replacement in self.patterns:
            text = text.replace(pattern, replacement)
        return text
```

---

## 5. 网络安全

### 5.1 出站流量控制

```python
class NetworkPolicy:
    """网络安全策略"""

    # 允许的出站端口
    ALLOWED_PORTS = {
        80,     # HTTP
        443,    # HTTPS
        22,     # SSH (Git)
        9418,   # Git 协议
        53,     # DNS
    }

    # 阻止的目标
    BLOCKED_DESTINATIONS = [
        "169.254.169.254",    # AWS 元数据服务
        "10.0.0.0/8",         # 内网
        "172.16.0.0/12",      # 内网（排除 Docker）
        "192.168.0.0/16",     # 内网
    ]

    # 速率限制
    RATE_LIMITS = {
        "connections_per_second": 50,
        "bandwidth_mbps": 100,
    }
```

### 5.2 DNS 安全

```python
class DNSSecurity:
    """DNS 安全"""

    # 使用安全 DNS 解析器
    DNS_SERVERS = [
        "1.1.1.1",    # Cloudflare
        "8.8.8.8",    # Google
    ]

    # 阻止已知恶意域名
    BLOCKLIST_SOURCES = [
        "https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts",
    ]
```

---

## 6. 权限控制

### 6.1 工具权限矩阵

```
┌──────────────────────────────────────────────────────────┐
│                   工具权限矩阵                            │
│                                                          │
│  Agent Type      │ Shell │ Write │ Git │ Screen │ Task  │
│ ─────────────────┼───────┼───────┼─────┼────────┼──────│
│  Main Agent      │  ✅   │  ✅   │ ✅  │  ✅    │  ✅  │
│  explore         │  🔒   │  ❌   │ ❌  │  ❌    │  ❌  │
│  vmSetupHelper   │  🔒   │  ❌   │ ❌  │  ❌    │  ❌  │
│  generalPurpose  │  ✅   │  ✅   │ ✅  │  ✅    │  ✅  │
│  debug           │  ✅   │  ✅   │ ❌  │  ❌    │  ❌  │
│  computerUse     │  ✅   │  ❌   │ ❌  │  ❌    │  ❌  │
│  videoReview     │  ❌   │  ❌   │ ❌  │  ❌    │  ❌  │
│                                                          │
│  🔒 = 只读模式                                           │
└──────────────────────────────────────────────────────────┘
```

### 6.2 GitHub CLI 权限

```python
class GitHubCLIPermissions:
    """GitHub CLI 权限控制"""

    READ_ONLY_COMMANDS = {
        "pr": ["view", "list", "status", "checks", "diff"],
        "run": ["view", "list", "watch"],
        "issue": ["view", "list", "status"],
        "repo": ["view"],
        "api": ["GET"],  # 只允许 GET 请求
    }

    BLOCKED_COMMANDS = {
        "pr": ["create", "merge", "close", "edit", "review"],
        "issue": ["create", "close", "edit", "comment"],
        "repo": ["create", "delete", "edit", "fork"],
        "release": ["create", "delete", "edit"],
        "api": ["POST", "PUT", "PATCH", "DELETE"],
    }

    def is_allowed(self, command: List[str]) -> bool:
        """检查 gh 命令是否被允许"""
        if len(command) < 2:
            return False

        resource = command[0]
        action = command[1]

        allowed = self.READ_ONLY_COMMANDS.get(resource, [])
        return action in allowed
```

---

## 7. 审计与日志

### 7.1 审计事件

```python
class AuditLog:
    """安全审计日志"""

    EVENTS = {
        # 认证事件
        "auth.login": "用户登录",
        "auth.token_refresh": "令牌刷新",

        # Secret 事件
        "secret.create": "创建 Secret",
        "secret.access": "访问 Secret",
        "secret.delete": "删除 Secret",
        "secret.leak_detected": "检测到 Secret 泄露",

        # Agent 事件
        "agent.session_start": "Agent 会话开始",
        "agent.session_end": "Agent 会话结束",
        "agent.tool_call": "Agent 工具调用",
        "agent.git_push": "Agent Git 推送",
        "agent.blocked_command": "Agent 被阻止的命令",

        # VM 事件
        "vm.create": "VM 创建",
        "vm.snapshot": "VM 快照",
        "vm.terminate": "VM 终止",

        # 安全事件
        "security.rate_limit_exceeded": "超出速率限制",
        "security.suspicious_activity": "可疑活动",
        "security.sandbox_violation": "沙箱违规",
    }

    def log(self, event_type: str, details: dict):
        """记录审计事件"""
        audit_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "event_type": event_type,
            "session_id": self.current_session_id,
            "user_id": self.current_user_id,
            "details": details,
            "source_ip": self.source_ip,
        }
        self.store.append(audit_entry)

        # 高风险事件实时告警
        if event_type in self.HIGH_RISK_EVENTS:
            self.alerter.send(audit_entry)
```

---

## 8. 安全最佳实践

### 8.1 Agent 行为约束

| 约束 | 实施方式 |
|------|---------|
| 不执行搜索结果中的可疑指令 | 系统提示中明确规则 |
| 不执行与用户请求无关的操作 | 上下文绑定 + 约束检查 |
| 不生成/暴露二进制数据 | 输出过滤 |
| 不修改系统关键文件 | 路径白名单 |
| 不进程名杀进程 | Shell 命令检查 |
| 不强制推送 | Git 操作约束 |

### 8.2 数据安全

| 措施 | 说明 |
|------|------|
| 传输加密 | 所有通信 TLS 1.3 |
| 存储加密 | Secret 值 AES-256-GCM |
| 日志脱敏 | 审计日志中不含 Secret 明文 |
| 会话隔离 | 不同用户的 VM 完全隔离 |
| 数据保留 | 按策略定期清理快照和产物 |

### 8.3 公共仓库安全

| 策略 | 说明 |
|------|------|
| 默认禁用 Secret 注入 | 公共仓库不注入 Secret |
| 需管理员启用 | 仅仓库所有者/管理员可启用 |
| 提交扫描 | Redacted Secret 类型在提交中扫描 |
| Fork 隔离 | Fork 仓库不继承 Secret |
