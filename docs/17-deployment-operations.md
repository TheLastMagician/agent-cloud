# 模块十七：部署架构与运维设计

> 本文档详细描述系统的生产部署架构、容器编排、CI/CD 管线、监控告警、灾备策略。

---

## 1. 部署架构总览

```
┌──────────────────────────────────────────────────────────────────────┐
│                            生产环境架构                               │
│                                                                      │
│  ┌────────────┐                                                      │
│  │ CloudFlare │  CDN + DDoS 防护 + WAF                               │
│  │ / AWS CF   │                                                      │
│  └─────┬──────┘                                                      │
│        │                                                             │
│  ┌─────▼──────────────────────────────────────────────────────────┐  │
│  │                    Kubernetes Cluster                           │  │
│  │                                                                │  │
│  │  ┌─────────────────────────────────────────────────────────┐   │  │
│  │  │  Ingress (nginx / Traefik)                              │   │  │
│  │  │  - TLS 终结                                             │   │  │
│  │  │  - 路由: /api → API, /ws → WebSocket, /static → CDN     │   │  │
│  │  └──────┬──────────┬──────────┬────────────────────────────┘   │  │
│  │         │          │          │                                │  │
│  │  ┌──────▼────┐ ┌───▼────┐ ┌──▼─────────┐                     │  │
│  │  │ API       │ │ WS     │ │ Frontend   │                     │  │
│  │  │ Service   │ │ Service│ │ (Next.js)  │                     │  │
│  │  │ (3 副本)  │ │(3 副本)│ │ (3 副本)   │                     │  │
│  │  └──────┬────┘ └───┬────┘ └────────────┘                     │  │
│  │         │          │                                          │  │
│  │  ┌──────▼──────────▼────────────────────┐                     │  │
│  │  │        Message Bus (Redis/NATS)       │                     │  │
│  │  └──────────────┬───────────────────────┘                     │  │
│  │                 │                                              │  │
│  │  ┌──────────────▼───────────────────────┐                     │  │
│  │  │       Orchestration Service           │                     │  │
│  │  │  - Task Router                        │                     │  │
│  │  │  - Session Manager                    │                     │  │
│  │  │  - Snapshot Manager                   │                     │  │
│  │  │  (3 副本, Leader Election)            │                     │  │
│  │  └──────────────┬───────────────────────┘                     │  │
│  │                 │                                              │  │
│  │  ┌──────────────▼───────────────────────┐                     │  │
│  │  │        VM Manager Service             │                     │  │
│  │  │  - VM Pool 管理                       │                     │  │
│  │  │  - 健康检查                           │                     │  │
│  │  │  (每个 VM Host 节点一个实例)           │                     │  │
│  │  └──────────────────────────────────────┘                     │  │
│  │                                                                │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                   VM Host Cluster (裸金属/EC2)                  │  │
│  │                                                                │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │  │
│  │  │ VM Host 1    │  │ VM Host 2    │  │ VM Host N    │   ...   │  │
│  │  │              │  │              │  │              │         │  │
│  │  │ ┌──┐┌──┐┌──┐│  │ ┌──┐┌──┐┌──┐│  │ ┌──┐┌──┐┌──┐│         │  │
│  │  │ │VM││VM││VM││  │ │VM││VM││VM││  │ │VM││VM││VM││         │  │
│  │  │ └──┘└──┘└──┘│  │ └──┘└──┘└──┘│  │ └──┘└──┘└──┘│         │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘         │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐     │
│  │ PostgreSQL       │ │ Redis Cluster    │ │ S3 / MinIO       │     │
│  │ (Primary +       │ │ (3 节点)         │ │ (对象存储)        │     │
│  │  Replica)        │ │                  │ │                  │     │
│  └──────────────────┘ └──────────────────┘ └──────────────────┘     │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. 服务拆分

| 服务 | 职责 | 副本数 | 语言 |
|------|------|--------|------|
| **API Service** | REST API, 认证, CRUD | 3+ | Go/Node.js |
| **WebSocket Service** | 实时通信, 事件分发 | 3+ | Go/Node.js |
| **Frontend** | Web UI (SSR) | 3+ | Next.js |
| **Orchestration Service** | 任务路由, 会话管理 | 3 (Leader) | Go/Rust |
| **VM Manager** | VM 生命周期, 池管理 | 每节点 1 | Go/Rust |
| **Snapshot Service** | 快照保存/恢复 | 2+ | Go |
| **Secret Service** | 密钥加解密 | 2+ | Go/Rust |
| **Artifact Service** | 产物上传/分发 | 2+ | Go |
| **Billing Service** | 计量, 计费 | 2+ | Go/Node.js |
| **Audit Service** | 审计日志 | 2+ | Go |

---

## 3. Kubernetes 资源定义

### 3.1 API Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      containers:
      - name: api
        image: registry.example.com/api-service:v1.2.3
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "2Gi"
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: url
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api-service
  ports:
  - port: 8080
    targetPort: 8080
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## 4. 监控与可观测性

### 4.1 监控栈

```
                     Grafana Dashboard
                          │
            ┌─────────────┼──────────────┐
            │             │              │
      ┌─────▼────┐  ┌────▼─────┐  ┌─────▼────┐
      │Prometheus │  │  Loki    │  │ Jaeger   │
      │ (指标)    │  │ (日志)   │  │ (追踪)   │
      └──────────┘  └──────────┘  └──────────┘
           ▲              ▲              ▲
           │              │              │
    ┌──────┴──┐    ┌──────┴──┐    ┌──────┴──┐
    │ Metrics │    │  Logs   │    │  Traces │
    │ Exporter│    │ Shipper │    │   SDK   │
    └─────────┘    └─────────┘    └─────────┘
```

### 4.2 关键 Dashboard

| Dashboard | 核心指标 |
|-----------|---------|
| **系统概览** | 活跃会话数、QPS、错误率、P99 延迟 |
| **VM 集群** | VM 利用率、Warm Pool 大小、创建延迟 |
| **Agent 性能** | Token 消耗、工具调用频率、会话时长分布 |
| **用户体验** | 首字节时间、任务完成率、用户满意度 |
| **基础设施** | CPU/内存/磁盘/网络、Pod 状态 |
| **安全** | 认证失败、异常检测、Secret 操作 |

### 4.3 告警规则

```yaml
# Prometheus AlertManager 规则示例

groups:
- name: agent-cloud-alerts
  rules:
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "API 错误率超过 5%"

  - alert: VMPoolDepleted
    expr: vm_warm_pool_size < 2
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "VM 预热池即将耗尽"

  - alert: SessionTimeout
    expr: rate(session_timeout_total[1h]) > 10
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "大量会话超时"

  - alert: HighLatency
    expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "P99 延迟超过 5 秒"

  - alert: DiskSpaceLow
    expr: node_filesystem_avail_bytes{mountpoint="/var/lib/firecracker"} / node_filesystem_size_bytes < 0.15
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "VM 存储空间不足 15%"
```

---

## 5. CI/CD 管线

```
代码推送到 main
    │
    ▼
┌──────────────┐
│ 单元测试      │  go test / npm test
└──────┬───────┘
       ▼
┌──────────────┐
│ 集成测试      │  Docker Compose 环境
└──────┬───────┘
       ▼
┌──────────────┐
│ 构建镜像      │  Docker buildx (多架构)
└──────┬───────┘
       ▼
┌──────────────┐
│ 安全扫描      │  Trivy 镜像扫描
└──────┬───────┘
       ▼
┌──────────────┐
│ 推送镜像      │  Container Registry
└──────┬───────┘
       ▼
┌──────────────┐
│ Staging 部署  │  Kubernetes Rolling Update
└──────┬───────┘
       ▼
┌──────────────┐
│ E2E 测试      │  Staging 环境验证
└──────┬───────┘
       ▼
┌──────────────┐
│ Production    │  Canary → Rolling Update
│ 部署          │  (10% → 50% → 100%)
└──────────────┘
```

---

## 6. 灾备策略

### 6.1 备份策略

| 组件 | 备份频率 | 保留期 | 方式 |
|------|---------|--------|------|
| PostgreSQL | 连续 WAL + 每日全量 | 30 天 | pg_basebackup + WAL archiving |
| Redis | RDB 每小时 + AOF | 7 天 | 主从复制 + 持久化 |
| S3 对象 | 跨区域复制 | 按策略 | S3 CRR |
| 配置文件 | Git 管理 | 永久 | GitOps |

### 6.2 多区域部署 (可选)

```
Region A (主)              Region B (灾备)
├── K8s Cluster           ├── K8s Cluster (热备)
├── PostgreSQL Primary    ├── PostgreSQL Replica
├── Redis Primary         ├── Redis Replica
├── S3 Bucket             ├── S3 Bucket (CRR)
└── VM Host Cluster       └── VM Host Cluster (冷备)
```

---

## 7. 扩容策略

| 组件 | 扩容方式 | 触发条件 |
|------|---------|---------|
| API / WS 服务 | K8s HPA (自动水平扩展) | CPU > 70% |
| VM Host 节点 | 集群自动扩展器 | Warm Pool < 阈值 |
| PostgreSQL | 只读副本 + 连接池 | 查询延迟 > 阈值 |
| Redis | Cluster 分片 | 内存 > 80% |
| S3 | 自动 (无限) | - |
