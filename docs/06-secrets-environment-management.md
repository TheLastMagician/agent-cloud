# 模块六：Secrets 与环境管理详细设计

> 本文档详细描述 Secrets 的生命周期管理、加密机制、注入流程、作用域解析，以及环境配置系统。

---

## 1. Secrets 管理系统

### 1.1 整体架构

```
┌────────────────────────────────────────────────────────────┐
│                    Secrets Management Service               │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ Secret CRUD  │  │ Encryption   │  │ Injection        │ │
│  │ API          │  │ Service      │  │ Service          │ │
│  └──────┬───────┘  └──────┬───────┘  └───────┬──────────┘ │
│         │                 │                   │            │
│  ┌──────▼─────────────────▼───────────────────▼──────────┐ │
│  │                  Secret Store                          │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────────┐ │ │
│  │  │  Personal   │ │    Team     │ │   Repo-scoped   │ │ │
│  │  │  Secrets    │ │  Secrets    │ │    Secrets      │ │ │
│  │  └─────────────┘ └─────────────┘ └─────────────────┘ │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                            │
│  ┌───────────────────────────────────────────────────────┐ │
│  │                 Redaction Scanner                      │ │
│  │  Git commit 扫描 → 检测泄露的 Secret 值                │ │
│  └───────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

### 1.2 Secret 数据模型

```python
from enum import Enum
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

class SecretType(Enum):
    SECRET = "secret"                # 标准加密
    REDACTED_SECRET = "redacted"     # 加密 + 提交扫描

class SecretScope(Enum):
    PERSONAL = "personal"     # 个人
    TEAM = "team"             # 团队
    REPO = "repo"             # 仓库

@dataclass
class Secret:
    name: str                          # SECRET_NAME
    encrypted_value: bytes             # 加密后的值
    type: SecretType                   # secret | redacted
    scope: SecretScope                 # personal | team | repo
    owner_id: str                      # 用户/团队 ID
    repo_id: Optional[str]            # 仓库 ID（repo scope）
    created_at: datetime
    updated_at: datetime
    created_by: str

    def decrypt(self, key: bytes) -> str:
        """解密 Secret 值"""
        return decrypt_aes_gcm(self.encrypted_value, key)
```

### 1.3 Secret 生命周期

```
创建 → 加密存储 → 注入 VM → 使用中 → 更新/删除
                                │
                        ┌───────┤
                        │       │
                  工具输出脱敏  提交扫描(Redacted类型)
```

---

## 2. 加密机制

### 2.1 加密架构

```
用户输入 Secret 值
    │
    ▼
┌──────────────┐
│ 传输层加密    │  TLS 1.3
│ (in transit) │
└──────┬───────┘
       ▼
┌──────────────┐
│ 应用层加密    │  AES-256-GCM
│ (at rest)    │
└──────┬───────┘
       ▼
┌──────────────┐
│ 密钥管理     │  AWS KMS / HashiCorp Vault
│ (key mgmt)  │
└──────────────┘
```

### 2.2 加密实现

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

class SecretEncryption:
    """Secret 加密服务"""

    def __init__(self, master_key_provider):
        self.key_provider = master_key_provider

    def encrypt(self, plaintext: str, key_id: str) -> bytes:
        """加密 Secret"""
        # 1. 获取数据加密密钥 (DEK)
        dek = self.key_provider.generate_data_key(key_id)

        # 2. 生成随机 nonce
        nonce = os.urandom(12)

        # 3. AES-256-GCM 加密
        aesgcm = AESGCM(dek.plaintext)
        ciphertext = aesgcm.encrypt(nonce, plaintext.encode(), None)

        # 4. 打包: encrypted_dek + nonce + ciphertext
        return dek.encrypted + nonce + ciphertext

    def decrypt(self, encrypted_data: bytes, key_id: str) -> str:
        """解密 Secret"""
        # 1. 解包
        encrypted_dek = encrypted_data[:256]
        nonce = encrypted_data[256:268]
        ciphertext = encrypted_data[268:]

        # 2. 解密 DEK
        dek = self.key_provider.decrypt_data_key(encrypted_dek, key_id)

        # 3. AES-256-GCM 解密
        aesgcm = AESGCM(dek)
        plaintext = aesgcm.decrypt(nonce, ciphertext, None)

        return plaintext.decode()
```

---

## 3. Secret 注入流程

### 3.1 注入时序

```
VM 启动
    │
    ▼
┌─────────────────┐
│ 1. 解析作用域    │  确定需要注入哪些 Secrets
└────────┬────────┘
         ▼
┌─────────────────┐
│ 2. 优先级解析    │  Personal > Team (同名覆盖)
└────────┬────────┘
         ▼
┌─────────────────┐
│ 3. 解密         │  从密钥管理服务获取 DEK 解密
└────────┬────────┘
         ▼
┌─────────────────┐
│ 4. 环境变量注入  │  设置到 VM 进程环境
└────────┬────────┘
         ▼
┌─────────────────┐
│ 5. 内存清理      │  解密后的临时副本安全擦除
└─────────────────┘
```

### 3.2 作用域优先级解析

```python
class SecretResolver:
    """Secret 作用域解析器"""

    def resolve(
        self,
        user_id: str,
        team_id: str,
        repo_id: str,
    ) -> Dict[str, str]:
        """解析最终的 Secret 集合"""

        resolved = {}

        # 1. 团队 Secrets (最低优先级)
        team_secrets = self.store.get_by_scope(
            scope=SecretScope.TEAM,
            owner_id=team_id,
        )
        for s in team_secrets:
            resolved[s.name] = s.decrypt(self.key)

        # 2. 个人 Secrets (覆盖团队同名)
        personal_secrets = self.store.get_by_scope(
            scope=SecretScope.PERSONAL,
            owner_id=user_id,
        )
        for s in personal_secrets:
            resolved[s.name] = s.decrypt(self.key)

        # 3. 仓库 Secrets (独立命名空间)
        repo_secrets = self.store.get_by_scope(
            scope=SecretScope.REPO,
            repo_id=repo_id,
        )
        for s in repo_secrets:
            resolved[s.name] = s.decrypt(self.key)

        return resolved
```

### 3.3 输出脱敏

```python
class OutputRedactor:
    """输出脱敏处理器"""

    def __init__(self, secrets: Dict[str, str]):
        self.secrets = secrets
        # 按值长度降序排序, 避免部分匹配问题
        self.sorted_values = sorted(
            secrets.values(),
            key=len,
            reverse=True,
        )

    def redact(self, output: str) -> str:
        """将输出中的 Secret 值替换为 [REDACTED]"""
        for value in self.sorted_values:
            if value and len(value) >= 4:  # 跳过过短的值
                output = output.replace(value, "[REDACTED]")
        return output
```

---

## 4. 提交扫描（Redacted Secret）

```python
class CommitScanner:
    """Git 提交 Secret 泄露扫描"""

    def scan_commit(self, commit_hash: str, redacted_secrets: List[Secret]) -> List[Finding]:
        """扫描提交中是否包含 Redacted Secret 的值"""
        findings = []

        # 获取提交差异
        diff = git.show(commit_hash, format="diff")

        for secret in redacted_secrets:
            plaintext = secret.decrypt(self.key)
            if plaintext in diff:
                findings.append(Finding(
                    secret_name=secret.name,
                    commit=commit_hash,
                    severity="CRITICAL",
                    message=f"Secret '{secret.name}' value found in commit",
                ))

        return findings

    def on_push(self, commits: List[str]):
        """Push 前扫描钩子"""
        for commit in commits:
            findings = self.scan_commit(commit, self.redacted_secrets)
            if findings:
                raise SecretLeakError(
                    f"Blocked push: {len(findings)} secret(s) detected in commits"
                )
```

---

## 5. 环境配置系统

### 5.1 配置优先级

```
最高优先级
    │
    ├── .cursor/environment.json (仓库中)
    │
    ├── Personal Environment Config
    │
    ├── Team Environment Config
    │
    └── 默认值
最低优先级
```

### 5.2 environment.json 格式

```json
{
  "vm": {
    "os": "ubuntu-24.04",
    "vcpu": 4,
    "memory_mb": 8192,
    "disk_gb": 50
  },
  "runtime": {
    "node_version": "20",
    "python_version": "3.12"
  },
  "docker": {
    "enabled": true,
    "compose_files": ["docker-compose.yml"]
  },
  "secrets": {
    "required": ["DATABASE_URL", "API_KEY"],
    "optional": ["SENTRY_DSN"]
  },
  "setup": {
    "update_script": "npm install"
  }
}
```

### 5.3 快照与 environment.json 的交互

```
场景: .cursor/environment.json 已在仓库中提交

问题: 快照管理的环境设置无法生效
     (因为 environment.json 优先级更高)

解决: Agent 必须在 PR 中删除 .cursor/environment.json
     以使快照管理的设置生效
```

---

## 6. 公共仓库安全

```python
class PublicRepoPolicy:
    """公共仓库的 Secret 注入策略"""

    def should_inject_secrets(
        self,
        repo: Repository,
        user: User,
    ) -> bool:
        """判断是否应该注入 Secrets"""

        if not repo.is_public:
            return True  # 私有仓库总是注入

        # 公共仓库默认禁用
        if not repo.settings.allow_secret_injection:
            return False

        # 只有仓库所有者/管理员可以启用
        if user.is_owner_or_admin(repo):
            return True

        return False
```

---

## 7. Agent 请求 Secrets 的交互协议

### 7.1 检查 → 请求流程

```python
def check_and_request_secrets(required_secrets: List[str]) -> Optional[str]:
    """
    Agent 在请求 Secrets 前的标准流程:
    1. 先检查环境变量中是否已存在
    2. 只请求确实缺失的
    """
    missing = []
    for name in required_secrets:
        if not os.environ.get(name):
            missing.append(name)

    if not missing:
        return None  # 所有 Secrets 已就位

    # 生成 XML 请求
    xml = '<env_setup_actions version="1">\n'
    xml += '  <add_secrets>\n'
    for name in missing:
        xml += f'    <secret name="{name}" />\n'
    xml += '    <reason><![CDATA[\n'
    xml += '需要这些 Secrets 来启动服务和运行测试\n'
    xml += '    ]]></reason>\n'
    xml += '  </add_secrets>\n'
    xml += '</env_setup_actions>'

    return xml
```

### 7.2 XML 动作类型汇总

| 动作 | XML 标签 | 属性 | 用途 |
|------|---------|------|------|
| 添加 Secrets | `<add_secrets>` | - | 请求用户添加密钥 |
| 添加测试账号 | `<add_test_login>` | username/password/otp_seed secret names | 请求测试登录凭证 |
| 外部操作 | `<external_action>` | id, title | 请求用户执行外部操作 |
