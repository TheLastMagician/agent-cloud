# 模块一：基础设施层详细设计

> 本文档详细描述 Cloud Agent 的基础设施层，包括虚拟化、容器化、网络、存储、资源管理等。

---

## 1. 整体架构

### 1.1 三级隔离模型

```
┌─────────────────────────────────────────────────────────────────┐
│                        物理宿主机集群                             │
│                                                                 │
│  ┌─────────────────────────┐  ┌─────────────────────────┐       │
│  │    Firecracker MicroVM   │  │    Firecracker MicroVM   │  ... │
│  │    (Agent Session 1)     │  │    (Agent Session 2)     │      │
│  │  ┌─────────────────────┐│  │  ┌─────────────────────┐│      │
│  │  │  Docker Container   ││  │  │  Docker Container   ││      │
│  │  │  ┌───────────────┐  ││  │  │  ┌───────────────┐  ││      │
│  │  │  │ Agent Runtime │  ││  │  │  │ Agent Runtime │  ││      │
│  │  │  └───────────────┘  ││  │  │  └───────────────┘  ││      │
│  │  └─────────────────────┘│  │  └─────────────────────┘│      │
│  └─────────────────────────┘  └─────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

**设计原理**：

| 层级 | 隔离方式 | 防护目标 |
|------|----------|---------|
| L1: Firecracker VM | 硬件虚拟化 (KVM) | 宿主机保护、租户间隔离 |
| L2: Docker Container | 进程/文件系统/网络命名空间 | 运行时环境标准化 |
| L3: Agent Process | 用户级权限 + 工具约束 | 最小权限原则 |

---

## 2. Firecracker MicroVM 详细设计

### 2.1 为什么选 Firecracker

| 特性 | Firecracker | 传统 VM (QEMU/KVM) | 容器 (Docker) |
|------|------------|-------------------|--------------|
| 启动时间 | < 125ms | 数秒~分钟 | < 1s |
| 内存开销 | ~5MB | ~128MB+ | 共享内核 |
| 安全隔离 | 硬件级 | 硬件级 | 内核级（较弱） |
| 设备模型 | 极简 | 完整 | 无 |
| API | RESTful | libvirt/QEMU | Docker API |

**结论**：Firecracker 在安全性与性能之间取得最佳平衡。

### 2.2 VM 生命周期管理

```
                    ┌──────────┐
     创建请求 ────▶ │ Creating │
                    └────┬─────┘
                         │ boot 完成
                         ▼
                    ┌──────────┐
                    │ Running  │◀─────────────────┐
                    └────┬─────┘                  │
                         │                        │
              ┌──────────┼──────────┐             │
              ▼          ▼          ▼             │
         ┌────────┐ ┌────────┐ ┌────────┐        │
         │Snapshot│ │ Pause  │ │ Error  │        │
         │ Save   │ │        │ │Recovery│        │
         └───┬────┘ └───┬────┘ └───┬────┘        │
             │          │          │              │
             ▼          ▼          │              │
        ┌─────────┐ ┌────────┐    │              │
        │Snapshot │ │ Resume │────┘              │
        │ Stored  │ └────────┘                   │
        └───┬─────┘                              │
            │ 恢复请求                             │
            └──────────────── Restore ───────────┘

         任意状态 ──▶ Terminate ──▶ Destroyed
```

### 2.3 VM 配置规格

```json
{
  "vm_config": {
    "vcpu_count": 2,
    "mem_size_mib": 4096,
    "kernel": {
      "image_path": "/opt/firecracker/vmlinux",
      "boot_args": "console=ttyS0 reboot=k panic=1 pci=off"
    },
    "rootfs": {
      "drive_id": "rootfs",
      "path": "/opt/firecracker/rootfs.ext4",
      "is_root_device": true,
      "is_read_only": false
    },
    "network": {
      "iface_id": "eth0",
      "guest_mac": "AA:FC:00:00:00:01",
      "host_dev_name": "tap0"
    }
  }
}
```

### 2.4 VM 编排服务（VMM - VM Manager）

```
┌─────────────────────────────────────────┐
│           VM Manager Service             │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │         VM Pool Manager         │    │
│  │  ┌───────┐ ┌───────┐ ┌───────┐ │    │
│  │  │ Warm  │ │ Active│ │ Cold  │ │    │
│  │  │ Pool  │ │ Pool  │ │ Store │ │    │
│  │  └───────┘ └───────┘ └───────┘ │    │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │      Lifecycle Controller        │    │
│  │  - create()                      │    │
│  │  - start()                       │    │
│  │  - snapshot_save()               │    │
│  │  - snapshot_restore()            │    │
│  │  - terminate()                   │    │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │      Resource Allocator          │    │
│  │  - CPU quota 分配                │    │
│  │  - Memory 分配与限制             │    │
│  │  - Disk I/O 限制                │    │
│  │  - Network bandwidth 限制        │    │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │       Health Monitor             │    │
│  │  - Heartbeat 检测                │    │
│  │  - OOM 检测与处理                │    │
│  │  - Timeout 检测                  │    │
│  │  - Error recovery                │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

### 2.5 VM 资源池策略

```python
# 伪代码：VM 池管理策略

class VMPoolManager:
    """
    维护三种池:
    - warm_pool:  预热的空 VM, 随时可用 (快速启动)
    - active_pool: 正在运行 Agent 会话的 VM
    - cold_store:  已快照保存的 VM 状态 (S3/对象存储)
    """

    WARM_POOL_SIZE = 10          # 预热池大小
    MAX_ACTIVE_VMS = 100         # 最大活跃 VM 数
    VM_IDLE_TIMEOUT = 1800       # 空闲超时 30 分钟
    VM_MAX_LIFETIME = 7200       # 最大生命周期 2 小时

    def acquire_vm(self, session_id: str, snapshot_id: str = None) -> VM:
        """获取一个 VM 实例"""
        if snapshot_id:
            # 从冷存储恢复快照
            vm = self.restore_snapshot(snapshot_id)
        elif self.warm_pool:
            # 从预热池获取
            vm = self.warm_pool.pop()
        else:
            # 创建新 VM
            vm = self.create_vm()

        self.active_pool[session_id] = vm
        self.refill_warm_pool()  # 异步补充预热池
        return vm

    def release_vm(self, session_id: str, save_snapshot: bool = True):
        """释放 VM"""
        vm = self.active_pool.pop(session_id)
        if save_snapshot:
            snapshot_id = self.save_snapshot(vm)
            self.cold_store[session_id] = snapshot_id
        vm.terminate()
```

---

## 3. Docker-in-Docker 详细设计

### 3.1 镜像构建

```dockerfile
FROM ubuntu:24.04

# 系统基础包
RUN apt-get update && apt-get install -y \
    curl wget git build-essential \
    python3 python3-pip python3-venv \
    && rm -rf /var/lib/apt/lists/*

# Node.js (通过 nvm)
ENV NVM_DIR=/root/.nvm
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash \
    && . "$NVM_DIR/nvm.sh" \
    && nvm install 20 \
    && nvm alias default 20 \
    && npm install -g pnpm yarn

# Docker-in-Docker
RUN install -m 0755 -d /etc/apt/keyrings \
    && curl --retry 3 --retry-delay 5 -fsSL \
       https://download.docker.com/linux/ubuntu/gpg \
       | gpg --dearmor -o /etc/apt/keyrings/docker.gpg \
    && chmod a+r /etc/apt/keyrings/docker.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) \
       signed-by=/etc/apt/keyrings/docker.gpg] \
       https://download.docker.com/linux/ubuntu \
       $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
       | tee /etc/apt/sources.list.d/docker.list > /dev/null \
    && apt-get update \
    && apt-get install -y \
       docker-ce=5:28.5.2-1~ubuntu.24.04~noble \
       docker-ce-cli=5:28.5.2-1~ubuntu.24.04~noble \
       containerd.io docker-buildx-plugin docker-compose-plugin \
    && rm -rf /var/lib/apt/lists/*

# Docker 存储驱动：fuse-overlayfs（Firecracker 内核限制）
RUN apt-get update && apt-get install -y fuse-overlayfs \
    && rm -rf /var/lib/apt/lists/*
RUN mkdir -p /etc/docker && printf '%s\n' \
    '{' '  "storage-driver": "fuse-overlayfs"' '}' \
    > /etc/docker/daemon.json

# iptables 兼容（Firecracker 内核 nftables 限制）
RUN apt-get update && apt-get install -y iptables \
    && rm -rf /var/lib/apt/lists/*
RUN update-alternatives --set iptables /usr/sbin/iptables-legacy \
    && update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy

# 搜索工具
RUN apt-get update && apt-get install -y ripgrep \
    && rm -rf /var/lib/apt/lists/*

# Chrome 浏览器（GUI 测试）
RUN wget -q https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb \
    && dpkg -i google-chrome-stable_current_amd64.deb || apt-get install -fy \
    && rm google-chrome-stable_current_amd64.deb

# GitHub CLI
RUN curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
    | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) \
       signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] \
       https://cli.github.com/packages stable main" \
       | tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
    && apt-get update && apt-get install -y gh \
    && rm -rf /var/lib/apt/lists/*

# 工作目录
WORKDIR /workspace

# 产物目录
RUN mkdir -p /opt/cursor/artifacts

# 终端状态目录
RUN mkdir -p /home/ubuntu/.cursor/projects/workspace/terminals

# 非 root 用户
RUN useradd -m -s /bin/bash ubuntu
USER ubuntu
```

### 3.2 Firecracker 内核兼容性问题及解决方案

| 问题 | 根因 | 解决方案 |
|------|------|----------|
| overlay2 不工作 | Firecracker 内核不完全支持 overlay2 | 使用 `fuse-overlayfs` |
| nftables 不工作 | Firecracker 内核不支持所有 nftables 特性 | 切换到 `iptables-legacy` |
| Docker 29+ 不兼容 | containerd-snapshotter 与 fuse-overlayfs 冲突 | 固定 Docker 28.x 或禁用 containerd-snapshotter |

---

## 4. 网络设计

### 4.1 网络拓扑

```
                 互联网
                   │
           ┌───────▼──────┐
           │   NAT Gateway │
           │   + Firewall  │
           └───────┬──────┘
                   │
           ┌───────▼──────────────┐
           │   宿主机网络          │
           │   Bridge: br0        │
           │                      │
           │   ┌──────────────┐   │
           │   │ TAP: tap0    │   │ ← VM 1
           │   ├──────────────┤   │
           │   │ TAP: tap1    │   │ ← VM 2
           │   ├──────────────┤   │
           │   │ TAP: tapN    │   │ ← VM N
           │   └──────────────┘   │
           └──────────────────────┘
```

### 4.2 网络规则

```bash
# 每个 VM 的 TAP 设备设置
ip tuntap add dev tap0 mode tap
ip addr add 172.16.0.1/24 dev tap0
ip link set tap0 up

# NAT 转发（允许 VM 访问互联网）
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i tap0 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o tap0 -m state --state RELATED,ESTABLISHED -j ACCEPT

# 出站流量限制（安全策略）
# - 允许: HTTPS(443), HTTP(80), SSH(22), Git(9418), DNS(53)
# - 限制: 速率限制防止滥用
# - 阻止: 内网扫描、特定恶意域名
```

### 4.3 端口映射与服务暴露

```
VM 内部端口        ←→       宿主机代理端口         ←→     用户浏览器
dev server :3000   ←→    Reverse Proxy :PORT_A   ←→    Desktop Pane (VNC/noVNC)
API server :8080   ←→    (仅 VM 内部访问)
Database   :5432   ←→    (仅 VM 内部访问)
VNC Server :5900   ←→    Reverse Proxy :PORT_B   ←→    Desktop Pane WebSocket
```

---

## 5. 存储设计

### 5.1 存储层次

```
┌─────────────────────────────────────────────────┐
│                  VM 存储布局                      │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │  rootfs (ext4, 读写)                     │    │
│  │  - OS 系统文件                           │    │
│  │  - 预装工具和运行时                       │    │
│  │  - Docker 数据 (/var/lib/docker)         │    │
│  │  容量: 20-50 GB                          │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │  /workspace (Git 仓库挂载)               │    │
│  │  - 用户代码                              │    │
│  │  - node_modules / venv 等                │    │
│  │  容量: 10-20 GB                          │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │  /opt/cursor/artifacts (产物输出)         │    │
│  │  - 截图 / 录屏 / 日志                    │    │
│  │  容量: 1-5 GB                            │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
         │
         │  快照保存 / 恢复
         ▼
┌─────────────────────────────────────────────────┐
│            对象存储 (S3 / MinIO)                  │
│                                                 │
│  snapshots/                                     │
│  ├── {session_id}/rootfs.snapshot               │
│  ├── {session_id}/memory.snapshot               │
│  └── {session_id}/metadata.json                 │
│                                                 │
│  artifacts/                                     │
│  ├── {session_id}/{filename}                    │
│  └── ...                                        │
└─────────────────────────────────────────────────┘
```

### 5.2 快照存储策略

```python
class SnapshotManager:
    """VM 快照管理"""

    SNAPSHOT_TTL_DAYS = 30        # 快照保留天数
    MAX_SNAPSHOTS_PER_USER = 50  # 每用户最大快照数

    def save_snapshot(self, vm: VM, session_id: str) -> SnapshotMeta:
        """保存 VM 快照"""
        # 1. 暂停 VM
        vm.pause()

        # 2. 保存内存状态
        memory_path = f"snapshots/{session_id}/memory.snapshot"
        vm.save_memory(memory_path)

        # 3. 保存磁盘增量（CoW）
        rootfs_path = f"snapshots/{session_id}/rootfs.snapshot"
        vm.save_disk_delta(rootfs_path)

        # 4. 保存元数据
        metadata = {
            "session_id": session_id,
            "timestamp": now(),
            "vm_config": vm.config,
            "git_branch": get_current_branch(),
            "git_commit": get_head_commit(),
            "update_script": get_update_script(),
        }
        save_json(f"snapshots/{session_id}/metadata.json", metadata)

        # 5. 上传到对象存储
        upload_to_s3(memory_path, rootfs_path, metadata)

        return SnapshotMeta(**metadata)

    def restore_snapshot(self, session_id: str) -> VM:
        """恢复 VM 快照"""
        # 1. 下载快照
        metadata = download_from_s3(f"snapshots/{session_id}/metadata.json")
        memory = download_from_s3(f"snapshots/{session_id}/memory.snapshot")
        rootfs = download_from_s3(f"snapshots/{session_id}/rootfs.snapshot")

        # 2. 创建 VM 并应用快照
        vm = create_vm(metadata["vm_config"])
        vm.restore_disk(rootfs)
        vm.restore_memory(memory)

        # 3. 拉取最新代码
        vm.exec("cd /workspace && git pull")

        # 4. 运行更新脚本
        vm.exec(metadata["update_script"])

        return vm
```

---

## 6. 资源管理

### 6.1 资源配额

| 资源 | 默认配额 | 最大配额 | 说明 |
|------|---------|---------|------|
| vCPU | 2 | 8 | 按计划升级 |
| 内存 | 4 GB | 16 GB | 按计划升级 |
| 磁盘 | 50 GB | 100 GB | 包含 rootfs + workspace |
| 网络出站 | 100 Mbps | 1 Gbps | 速率限制 |
| 会话时长 | 30 min | 120 min | 可续期 |
| Shell 超时 | 30s | 600s (10min) | 单命令超时 |

### 6.2 资源监控

```python
class ResourceMonitor:
    """VM 资源监控"""

    METRICS = [
        "cpu_usage_percent",
        "memory_usage_mb",
        "disk_usage_gb",
        "network_tx_bytes",
        "network_rx_bytes",
        "process_count",
        "open_file_count",
    ]

    ALERTS = {
        "memory_usage_percent > 90": "WARNING: Memory pressure",
        "disk_usage_percent > 85": "WARNING: Disk space low",
        "cpu_usage_percent > 95 for 60s": "WARNING: CPU saturated",
    }

    def collect_metrics(self, vm_id: str) -> dict:
        """采集 VM 指标"""
        return {
            "cpu": self.get_cpu_usage(vm_id),
            "memory": self.get_memory_usage(vm_id),
            "disk": self.get_disk_usage(vm_id),
            "network": self.get_network_stats(vm_id),
            "timestamp": now(),
        }

    def check_health(self, vm_id: str) -> HealthStatus:
        """健康检查"""
        metrics = self.collect_metrics(vm_id)
        alerts = []

        if metrics["memory"]["percent"] > 90:
            alerts.append(Alert.MEMORY_PRESSURE)

        if metrics["disk"]["percent"] > 85:
            alerts.append(Alert.DISK_LOW)

        if not self.heartbeat(vm_id, timeout=30):
            alerts.append(Alert.UNRESPONSIVE)

        return HealthStatus(healthy=len(alerts) == 0, alerts=alerts)
```

---

## 7. 高可用与容错

### 7.1 故障恢复策略

| 故障类型 | 检测方式 | 恢复策略 |
|----------|---------|---------|
| VM 进程崩溃 | Heartbeat 超时 | 从最近快照恢复 |
| OOM Kill | cgroup 通知 | 增加内存重启 |
| 磁盘满 | 磁盘监控告警 | 清理临时文件后重启 |
| 网络中断 | 连接探测 | 等待恢复 / 重连 |
| 宿主机故障 | 集群健康检查 | 迁移到新宿主机 |

### 7.2 优雅关闭流程

```
收到关闭信号
    │
    ├── 1. 通知 Agent 停止接受新工具调用
    ├── 2. 等待当前工具调用完成 (最大 60s)
    ├── 3. 保存 VM 快照
    ├── 4. 上传产物到对象存储
    ├── 5. 更新会话状态 (可恢复)
    └── 6. 终止 VM
```
