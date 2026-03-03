# 模块二十二：多租户与组织架构设计

> 本文档详细描述系统的多租户隔离模型、组织/团队/用户层级、权限控制、数据隔离策略。

---

## 1. 组织层级模型

```
┌─────────────────────────────────────────────────────┐
│                  Organization (组织)                  │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │  Team A (团队)                               │    │
│  │  ├── User 1 (admin)                          │    │
│  │  ├── User 2 (member)                         │    │
│  │  └── User 3 (member)                         │    │
│  │                                              │    │
│  │  Shared Resources:                           │    │
│  │  ├── Team Secrets                            │    │
│  │  ├── Repo Access (repos A, B, C)             │    │
│  │  └── Billing (shared quota)                  │    │
│  └─────────────────────────────────────────────┘    │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │  Team B (团队)                               │    │
│  │  ├── User 4 (admin)                          │    │
│  │  └── User 5 (member)                         │    │
│  │                                              │    │
│  │  Shared Resources:                           │    │
│  │  ├── Team Secrets (独立于 Team A)             │    │
│  │  └── Repo Access (repos D, E)                │    │
│  └─────────────────────────────────────────────┘    │
│                                                     │
│  Organization-level:                                │
│  ├── Org Billing (总配额)                           │
│  ├── Org Settings (策略)                            │
│  └── Audit Logs (全组织)                            │
└─────────────────────────────────────────────────────┘
```

---

## 2. 数据模型

```sql
-- 组织
CREATE TABLE orgs (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB DEFAULT '{}',
    billing_email   VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 团队
CREATE TABLE teams (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL REFERENCES orgs(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, slug)
);

-- 用户-组织关系
CREATE TABLE org_members (
    org_id          UUID REFERENCES orgs(id),
    user_id         UUID REFERENCES users(id),
    role            VARCHAR(20) NOT NULL DEFAULT 'member',
                    -- owner | admin | member
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (org_id, user_id)
);

-- 用户-团队关系
CREATE TABLE team_members (
    team_id         UUID REFERENCES teams(id),
    user_id         UUID REFERENCES users(id),
    role            VARCHAR(20) NOT NULL DEFAULT 'member',
                    -- admin | member
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (team_id, user_id)
);

-- 团队-仓库访问权限
CREATE TABLE team_repos (
    team_id         UUID REFERENCES teams(id),
    repo_id         UUID REFERENCES repos(id),
    permission      VARCHAR(20) NOT NULL DEFAULT 'read',
                    -- read | write | admin
    PRIMARY KEY (team_id, repo_id)
);
```

---

## 3. 权限控制模型 (RBAC)

### 3.1 角色权限矩阵

```
┌──────────────────────────────────────────────────────────────────┐
│                        权限矩阵                                   │
│                                                                  │
│  操作                    │ Owner │ Admin │ Member │ Guest         │
│ ─────────────────────────┼───────┼───────┼────────┼─────────────│
│  创建/删除组织            │  ✅   │  ❌   │  ❌    │  ❌          │
│  管理组织设置             │  ✅   │  ✅   │  ❌    │  ❌          │
│  管理计费                │  ✅   │  ✅   │  ❌    │  ❌          │
│  邀请/移除成员            │  ✅   │  ✅   │  ❌    │  ❌          │
│  创建/删除团队            │  ✅   │  ✅   │  ❌    │  ❌          │
│  管理团队成员             │  ✅   │  ✅   │  ❌    │  ❌          │
│  查看审计日志             │  ✅   │  ✅   │  ❌    │  ❌          │
│  管理 Team Secrets       │  ✅   │  ✅   │  ❌    │  ❌          │
│  管理 Personal Secrets   │  ✅   │  ✅   │  ✅    │  ❌          │
│  创建 Agent 会话          │  ✅   │  ✅   │  ✅    │  ❌          │
│  查看其他人的会话         │  ✅   │  ✅   │  ❌    │  ❌          │
│  查看自己的会话           │  ✅   │  ✅   │  ✅    │  ✅(只读)    │
│  配置仓库设置             │  ✅   │  ✅   │  ❌    │  ❌          │
│  查看使用量统计           │  ✅   │  ✅   │  ✅    │  ❌          │
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 权限检查实现

```python
class PermissionChecker:
    """权限检查器"""

    def check(self, user: User, action: str, resource: Resource) -> bool:
        """检查用户是否有权限执行操作"""

        # 1. 获取用户在组织中的角色
        org_role = self.get_org_role(user.id, resource.org_id)

        # 2. Owner 拥有所有权限
        if org_role == "owner":
            return True

        # 3. 按操作检查
        match action:
            case "session.create":
                # 需要: 对仓库有 write 权限
                return self.has_repo_access(user.id, resource.repo_id, "write")

            case "secret.manage_team":
                # 需要: org admin 或 team admin
                return org_role == "admin" or \
                       self.is_team_admin(user.id, resource.team_id)

            case "secret.manage_personal":
                # 任何成员都可以管理自己的 secrets
                return org_role in ("admin", "member")

            case "audit.view":
                return org_role == "admin"

            case _:
                return False

    def has_repo_access(
        self, user_id: str, repo_id: str, required: str
    ) -> bool:
        """检查用户对仓库的访问权限"""
        # 通过团队关系检查
        teams = self.get_user_teams(user_id)
        for team in teams:
            perm = self.get_team_repo_permission(team.id, repo_id)
            if perm and self.permission_level(perm) >= self.permission_level(required):
                return True
        return False
```

---

## 4. 数据隔离策略

### 4.1 隔离层次

```
┌─────────────────────────────────────────────────┐
│              数据隔离层次                         │
│                                                 │
│  Level 1: VM 隔离 (最强)                         │
│  ├── 每个 Agent 会话运行在独立 VM 中              │
│  ├── 不同用户的 VM 完全隔离                       │
│  └── 无法跨 VM 访问                              │
│                                                 │
│  Level 2: 数据库行级隔离                          │
│  ├── 所有查询带 org_id / user_id 过滤            │
│  ├── Row-Level Security (RLS) 策略               │
│  └── 审计日志不可跨组织查看                       │
│                                                 │
│  Level 3: Secret 隔离                            │
│  ├── Personal: 仅创建者可见                      │
│  ├── Team: 仅团队成员可见                        │
│  ├── Repo: 仅有仓库权限的用户可用                 │
│  └── 加密密钥按组织隔离                           │
│                                                 │
│  Level 4: 快照/产物隔离                           │
│  ├── S3 key 包含 org_id/user_id                  │
│  ├── 产物 URL 带签名, 有过期时间                  │
│  └── 不可通过 URL 猜测访问                       │
└─────────────────────────────────────────────────┘
```

### 4.2 RLS (Row-Level Security)

```sql
-- PostgreSQL RLS 策略示例

ALTER TABLE sessions ENABLE ROW LEVEL SECURITY;

CREATE POLICY sessions_isolation ON sessions
    USING (
        user_id = current_setting('app.current_user_id')::uuid
        OR
        EXISTS (
            SELECT 1 FROM org_members
            WHERE org_members.user_id = current_setting('app.current_user_id')::uuid
            AND org_members.org_id = (
                SELECT org_id FROM repos WHERE id = sessions.repo_id
            )
            AND org_members.role IN ('owner', 'admin')
        )
    );

ALTER TABLE secrets ENABLE ROW LEVEL SECURITY;

CREATE POLICY secrets_personal ON secrets
    FOR ALL
    USING (
        (scope = 'personal' AND owner_id = current_setting('app.current_user_id')::uuid)
        OR
        (scope = 'team' AND owner_id IN (
            SELECT team_id FROM team_members
            WHERE user_id = current_setting('app.current_user_id')::uuid
        ))
        OR
        (scope = 'repo' AND repo_id IN (
            SELECT repo_id FROM team_repos tr
            JOIN team_members tm ON tm.team_id = tr.team_id
            WHERE tm.user_id = current_setting('app.current_user_id')::uuid
        ))
    );
```

---

## 5. 配额分配

### 5.1 组织级配额

```
组织 (Enterprise Plan)
├── 总月 Token 配额: 50M
├── 总并发会话: 20
├── 总存储: 500GB
│
├── Team A (分配 60%)
│   ├── Token: 30M
│   ├── 并发: 12
│   └── 存储: 300GB
│
└── Team B (分配 40%)
    ├── Token: 20M
    ├── 并发: 8
    └── 存储: 200GB
```

### 5.2 个人 vs 团队配额

```python
class QuotaResolver:
    """配额解析"""

    def resolve(self, user_id: str) -> Quota:
        # 个人用户
        if not user.org_id:
            return self.get_personal_quota(user.plan)

        # 组织用户
        org = self.get_org(user.org_id)
        team = self.get_user_primary_team(user_id)

        return Quota(
            token_limit=team.token_allocation or org.token_limit,
            concurrent_sessions=min(
                team.concurrent_allocation or org.concurrent_limit,
                org.concurrent_limit
            ),
            storage_gb=team.storage_allocation or org.storage_limit,
        )
```
