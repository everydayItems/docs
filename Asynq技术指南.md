# Asynq 技术指南

> 本文档整合了 Asynq 队列的基础概念、监控运维、Prometheus 集成等内容，适用于开发和运维人员参考。

## 目录

1. [基础概念](#一基础概念)
2. [监控与运维](#二监控与运维)
3. [日常运维操作](#三日常运维操作)
4. [高级特性](#四高级特性)
5. [故障排查](#五故障排查)
6. [配置参考](#六配置参考)

---

## 一、基础概念

### 1.1 队列类型与优先级

**Asynq** 是基于 Redis 实现的分布式任务队列。系统定义了 6 个队列类型：

| 队列 | 权重 | 用途 | 响应要求 | 告警阈值 |
|------|------|------|---------|---------|
| **critical** | 10 | 高优先级任务、数据同步、回调处理 | 毫秒级 | > 100 |
| **high** | 6 | 计算密集型任务 | 秒级 | > 500 |
| **default** | 4 | I/O 密集型任务 | 秒级 | > 1000 |
| **scheduled** | 3 | 异步回调任务 | 分钟级 | > 5000 |
| **low** | 1 | 报表生成、日志清理、数据归档 | 无时限 | > 10000 |
| **dead_letter** | 1 | 失败任务存档（人工介入） | - | **> 0** |

> 权重决定 Worker 消费频率。critical:high = 10:6，即 critical 消费频率约为 high 的 1.67 倍。

### 1.2 任务状态与生命周期

| 状态 | 说明 | 产生原因 | 可执行操作 |
|------|------|---------|------------|
| **Pending** | 等待消费 | 刚入队 / Worker 繁忙 | Run、Delete、Archive |
| **Active** | 处理中 | Worker 正在执行 | Cancel |
| **Scheduled** | 延迟等待 | ProcessIn 延迟 / 轮询退避 | Run Now、Delete |
| **Retry** | 等待重试 | Handler 错误 / 超时 | Run Now、Archive、Delete |
| **Archived** | 死信（已归档） | 重试耗尽 / 手动归档 | Run、Delete |
| **Completed** | 已完成 | 成功（需设 Retention） | Delete |

### 1.3 状态流转图

```
                                    ┌─────────────┐
                                    │  Scheduled  │ ←── ProcessIn(delay)
                                    └──────┬──────┘
                                           │ 到达执行时间
                                           ▼
┌─────────┐    入队     ┌─────────┐  Worker取出  ┌─────────┐
│  New    │ ──────────→ │ Pending │ ───────────→ │  Active │
└─────────┘             └─────────┘              └────┬────┘
                                                      │
                        ┌─────────────────────────────┼─────────────────────────────┐
                        │                             │                             │
                        ▼                             ▼                             ▼
                 ┌───────────┐              ┌─────────────────┐            ┌─────────────┐
                 │   Retry   │              │    Completed    │            │  Archived   │
                 │ (失败重试) │              │  (成功，有Ret.) │            │  (死信队列)  │
                 └─────┬─────┘              └─────────────────┘            └─────────────┘
                       │
                       │ 重试次数耗尽
                       ▼
                ┌─────────────┐
                │  Archived   │
                └─────────────┘
```

### 1.4 重试策略（指数退避）

异步任务采用**指数退避**策略：初始延迟 → 逐步增长 → 上限，最多重试 120 次后进入死信队列。

| 平台类型 | 初始延迟 | 最大延迟 |
|---------|---------|---------|
| 长时任务 | 10s | 30s |
| 中时任务 | 5s | 20s |
| 短时任务 | 3s | 15s |

### 1.5 任务 ID 格式

| 类型 | 格式 | 示例 |
|------|------|------|
| 初始任务 | `job:{taskID}:0` | job:task_abc:0 |
| 重试任务 | `job:{taskID}:{n}` | job:task_abc:3 |
| 死信重试 | `job:{taskID}:retry:{ts}` | job:task_abc:retry:1642000000 |

### 1.6 Redis Key 结构

| 状态 | Key | 结构 |
|------|-----|------|
| Pending | `asynq:{queue}:pending` | List |
| Active | `asynq:{queue}:active` | Sorted Set |
| Scheduled | `asynq:{queue}:scheduled` | Sorted Set |
| Retry | `asynq:{queue}:retry` | Sorted Set |
| Archived | `asynq:{queue}:archived` | Sorted Set |
| Completed | `asynq:{queue}:completed` | Sorted Set |

---

## 二、监控与运维

### 2.1 监控工具对比

| 对比维度 | Grafana 集成 | Asynqmon Web UI | 管理 API |
|---------|-------------|-----------------|----------|
| **部署复杂度** | 高（需要 Prometheus + Grafana） | 低（内嵌到后端） | 无需部署 |
| **配置难度** | 中等（需要编写 PromQL 查询） | 极低（开箱即用） | 极低 |
| **实时性** | 准实时（抓取间隔 15s） | 实时（直连 Redis） | 实时 |
| **历史数据** | ✅ 支持（可查询历史趋势） | ❌ 不支持（仅当前状态） | ❌ 不支持 |
| **告警功能** | ✅ 支持（Grafana Alerting） | ❌ 不支持 | ❌ 不支持 |
| **任务管理** | ❌ 只能查看（只读） | ✅ 可操作（重试/删除/归档） | ✅ 可操作 |
| **资源消耗** | 高（~700MB 内存 + 50GB 存储） | 低（~10MB 内存） | 无 |
| **适用场景** | 生产环境长期监控 | 开发/调试/快速查看 | 脚本自动化 |

### 2.2 选择建议

| 场景 | 推荐工具 | 原因 |
|------|---------|------|
| **日常运维** | Asynqmon Web UI | 操作方便，实时性好，可直接重试/删除任务 |
| **趋势分析** | Grafana | 支持历史数据查询和趋势图表 |
| **告警配置** | Grafana | 支持 Alerting，可配置多种通知渠道 |
| **脚本自动化** | 管理 API | 可集成到运维脚本和 CI/CD 流程 |
| **生产环境** | 三者配合 | Web UI 日常操作 + Grafana 监控告警 + API 自动化 |

**推荐流程**：Grafana 告警 → Asynqmon 定位 → API/Web UI 操作

### 2.3 Asynqmon Web UI

#### 界面概览

| 顶部菜单 | 功能 | 说明 |
|---------|------|------|
| **Queues** | 查看队列状态 | 主要操作入口 |
| **Servers** | 查看 Worker 节点 | 检查 Worker 数量和状态 |
| **Schedulers** | 查看定时任务 | 检查定时任务配置 |
| **Redis Info** | 查看 Redis 信息 | 检查 Redis 连接和内存 |

#### 队列正常范围

| 队列名 | 权重 | 用途 | 正常范围 |
|--------|------|------|---------|
| `critical` | 10 | 高优先级任务、数据同步 | pending < 10 |
| `high` | 6 | 计算密集型任务 | pending < 50 |
| `default` | 4 | 一般任务 | pending < 100 |
| `scheduled` | 3 | 异步回调任务 | pending < 500 |
| `low` | 1 | 清理任务 | pending < 50 |
| `dead_letter` | 1 | 失败任务 | archived < 20 |

#### 任务详情字段

| 字段 | 说明 | 示例 |
|------|------|------|
| **ID** | 任务唯一标识 | `job:task_abc123:0` |
| **Type** | 任务类型 | `task:process` |
| **Queue** | 所属队列 | `scheduled` |
| **State** | 当前状态 | `pending` |
| **Retried** | 已重试次数 | `3` |
| **Max Retry** | 最大重试次数 | `0`（由代码控制） |
| **Next Process At** | 下次执行时间 | `2026-01-22T10:30:00Z` |
| **Payload** | 任务数据（JSON） | 包含业务参数 |
| **Last Error** | 最后一次错误 | `timeout` |

### 2.4 Grafana 监控配置

#### 关键指标

| 指标名 | 说明 | PromQL 示例 |
|--------|------|-------------|
| `asynq_queue_size` | 队列任务数 | `asynq_queue_size{queue="scheduled"}` |
| `asynq_active_workers` | 活跃 Worker 数 | `asynq_active_workers` |
| `asynq_processed_total` | 已处理任务数 | `rate(asynq_processed_total[5m])` |
| `asynq_failed_total` | 失败任务数 | `rate(asynq_failed_total[5m])` |

#### 推荐 Dashboard 面板

| 面板名称 | 用途 | 数据源 |
|---------|------|--------|
| 队列大小趋势 | 监控任务堆积情况 | `asynq_queue_size` |
| 处理速率 | 监控任务处理效率 | `rate(asynq_processed_total[5m])` |
| 失败率 | 监控任务失败比例 | `asynq_failed_total / asynq_processed_total` |
| Worker 状态 | 监控 Worker 数量变化 | `asynq_active_workers` |

#### 告警配置建议

| 告警名称 | 条件 | 严重级别 | 通知渠道 |
|---------|------|---------|---------|
| 队列堆积 | pending > 500 持续 5 分钟 | Warning | Slack/企业微信 |
| 队列严重堆积 | pending > 2000 持续 5 分钟 | Critical | 电话/短信 |
| 重试过多 | retry > 50 持续 10 分钟 | Warning | Slack/企业微信 |
| 死信任务增加 | archived 增量 > 10/小时 | Warning | Slack/企业微信 |
| Worker 离线 | worker_count = 0 持续 2 分钟 | Critical | 电话/短信 |
| 背压触发 | backpressure_state != NORMAL | Warning | Slack/企业微信 |

### 2.5 Prometheus 集成

#### 配置步骤

**第一步：编辑 Prometheus 配置文件**

找到 `prometheus.yml` 配置文件（常见位置）：

```bash
# 直接安装
/etc/prometheus/prometheus.yml

# Docker 部署 - 查看挂载路径
docker inspect prometheus | grep -A5 Mounts

# K8s 部署 - 查看 ConfigMap
kubectl get configmap prometheus-config -n monitoring -o yaml
```

**第二步：添加抓取配置**

在 `prometheus.yml` 的 `scrape_configs` 部分添加：

```yaml
scrape_configs:
  # 新增：Asynq 监控
  - job_name: 'app-asynq'
    scheme: https
    metrics_path: '/metrics'
    scrape_interval: 15s
    scrape_timeout: 10s
    static_configs:
      - targets: ['api.example.com']
        labels:
          env: 'production'
          app: 'myapp'
    # HTTPS 配置（如遇证书问题可取消注释）
    # tls_config:
    #   insecure_skip_verify: true
```

**第三步：验证配置文件语法**

```bash
promtool check config /etc/prometheus/prometheus.yml
# 输出应为：SUCCESS
```

**第四步：重载 Prometheus 配置**

```bash
# 方式一：发送 SIGHUP 信号（推荐，无需重启服务）
kill -HUP $(pgrep -f prometheus)

# 方式二：调用 reload API（需启用 --web.enable-lifecycle）
curl -X POST http://localhost:9090/-/reload

# 方式三：重启服务
sudo systemctl restart prometheus
# 或 Docker
docker restart prometheus
# 或 K8s
kubectl rollout restart deployment/prometheus -n monitoring
```

**第五步：验证抓取成功**

1. 访问 Prometheus UI：`http://<prometheus-ip>:9090/targets`
2. 找到 `app-asynq` job，确认 State 为 **UP**（绿色）
3. 在 Graph 页面执行查询：`asynq_system_health`，应返回值为 `1`

#### 预期抓取的指标

| 指标名 | 说明 |
|--------|------|
| `asynq_system_health` | 系统健康状态（1=健康） |
| `asynq_active_workers` | 活跃 Worker 数量 |
| `asynq_queue_size` | 队列任务总数 |
| `asynq_pending_tasks` | 待处理任务数 |
| `asynq_active_tasks` | 正在处理的任务数 |
| `asynq_scheduled_tasks` | 计划执行的任务数 |
| `asynq_retry_tasks` | 等待重试的任务数 |
| `asynq_dead_tasks` | 死信队列任务数 |

#### 常见问题

| 错误信息 | 解决方案 |
|---------|----------|
| `connection refused` | 检查目标地址是否正确，防火墙是否放行 |
| `context deadline exceeded` | 增加 `scrape_timeout` 或检查网络延迟 |
| `x509: certificate signed by unknown authority` | 添加 `tls_config: insecure_skip_verify: true` |
| `server returned HTTP status 404` | 检查 `metrics_path` 是否正确 |

### 2.6 管理 API 接口

> API 主要用于脚本自动化和 Grafana 集成，日常操作建议优先使用 Web UI。

#### 接口汇总

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/v1/queue/stats` | GET | 获取队列统计信息 |
| `/api/v1/queue/health` | GET | 获取健康状态 |
| `/api/v1/queue/overview` | GET | 获取概览信息 |
| `/api/v1/queue/tasks` | GET | 查询队列任务列表 |
| `/api/v1/queue/tasks/retry` | POST | 重试队列任务 |
| `/api/v1/queue/tasks/:id` | DELETE | 删除队列任务 |
| `/api/v1/queue/tasks/archive` | POST | 归档队列任务 |
| `/api/v1/queue/tasks/cancel` | POST | 手动取消任务+资源回退 |
| `/api/v1/queue/tasks/cancel/batch` | POST | 批量取消任务+资源回退 |

#### 获取队列统计信息

```bash
curl -H "Authorization: Bearer <admin-token>" \
  http://localhost:8000/api/v1/queue/stats
```

**响应**：
```json
{
  "success": true,
  "data": {
    "scheduled": {
      "queue": "scheduled",
      "size": 150,
      "pending": 100,
      "active": 10,
      "scheduled": 35,
      "retry": 5,
      "archived": 3
    }
  }
}
```

#### 获取健康状态

```bash
curl -H "Authorization: Bearer <admin-token>" \
  http://localhost:8000/api/v1/queue/health
```

**响应**：
```json
{
  "success": true,
  "data": {
    "healthy": true,
    "redis_connected": true,
    "queue_utilization": 0.15,
    "worker_active_count": 8,
    "processor_count": 20,
    "backpressure_state": "NORMAL"
  }
}
```

#### 查询任务列表

```bash
curl -H "Authorization: Bearer <admin-token>" \
  "http://localhost:8000/api/v1/queue/tasks?queue=scheduled&state=retry&page=1&size=20"
```

**参数**：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `queue` | `scheduled` | 队列名 |
| `state` | `pending` | 任务状态（pending/active/scheduled/retry/archived） |
| `page` | `1` | 页码 |
| `size` | `20` | 每页数量（1-100） |

#### 手动重试任务

```bash
curl -X POST -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -d '{"queue":"scheduled","task_id":"job:task_xxx:5"}' \
  http://localhost:8000/api/v1/queue/tasks/retry
```

#### 删除任务

```bash
curl -X DELETE -H "Authorization: Bearer <admin-token>" \
  "http://localhost:8000/api/v1/queue/tasks/job:task_xxx:0?queue=scheduled&state=archived"
```

#### 归档任务

```bash
curl -X POST -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -d '{"queue":"scheduled","task_id":"job:task_xxx:0"}' \
  http://localhost:8000/api/v1/queue/tasks/archive
```

#### 手动取消任务+资源回退

将异步任务取消并回退资源。适用于外部服务故障、用户申请取消、任务卡住等场景。

```bash
curl -X POST -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -d '{"task_id":"task_xxx","reason":"用户申请取消"}' \
  http://localhost:8000/api/v1/queue/tasks/cancel
```

**请求参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `task_id` | string | 是 | 任务 ID |
| `reason` | string | 否 | 取消原因（默认：管理员手动取消） |

**响应**：
```json
{
  "success": true,
  "message": "任务已取消并退还资源",
  "data": {
    "task_id": "task_xxx",
    "user_id": 100,
    "credits": 100000,
    "refunded": true,
    "reason": "用户申请取消"
  }
}
```

#### 批量取消任务+资源回退

```bash
curl -X POST -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -d '{"task_ids":["task_xxx","task_yyy"],"reason":"外部服务故障批量取消"}' \
  http://localhost:8000/api/v1/queue/tasks/cancel/batch
```

**请求参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `task_ids` | string[] | 是 | 任务 ID 列表（1-100 个） |
| `reason` | string | 否 | 取消原因（默认：管理员批量取消） |

---

## 三、日常运维操作

### 3.1 取消任务

**场景**：用户请求取消尚未完成的任务

**Web UI 操作**：
1. `Queues` -> `{队列名}` -> `Scheduled` 或 `Pending`
2. 找到目标任务，点击任务 ID
3. 点击 `Delete` 删除

**API 备选**：
```bash
curl -X DELETE -H "Authorization: Bearer <admin-token>" \
  "http://localhost:8000/api/v1/queue/tasks/job:task_xxx:0?queue=scheduled&state=scheduled"
```

**注意**：`active` 状态的任务正在执行，无法直接取消

### 3.2 重试失败任务

**场景**：外部 API 恢复后，重试之前失败的任务

**Web UI 操作（单个）**：
1. `Queues` -> `scheduled` -> `Retry` 或 `Archived`
2. 点击任务 ID 查看 Last Error，确认问题已修复
3. 点击 `Run Now`

**Web UI 操作（批量）**：
1. `Queues` -> `scheduled` -> `Retry`
2. 点击 `Run All`

**API 备选**：
```bash
curl -X POST -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -d '{"queue":"scheduled","task_id":"job:task_xxx:5"}' \
  http://localhost:8000/api/v1/queue/tasks/retry
```

### 3.3 清理死信队列

**场景**：定期清理已归档的失败任务

**Web UI 操作**：
1. `Queues` -> `dead_letter` -> `Archived`
2. 查看任务列表，确认无需保留
3. 点击 `Delete All` 批量删除

**Grafana 监控**：设置告警 `archived > 20` 时通知运维人员

### 3.4 查看任务堆积情况

**场景**：监控队列是否有任务堆积

**Web UI 操作**：
1. `Queues` 查看各队列的 pending 数量
2. 重点关注 `scheduled` 队列（异步回调任务最多）

**正常范围与告警阈值**：

| 队列 | pending 正常范围 | 告警阈值 |
|------|-----------------|---------|
| `critical` | < 10 | > 50 |
| `high` | < 50 | > 200 |
| `scheduled` | < 500 | > 2000 |
| `dead_letter` (archived) | < 20 | > 50 |

### 3.5 查看 Worker 健康状态

**场景**：确认 Worker 是否正常运行

**Web UI 操作**：
- `Servers` 查看 Worker 列表和活跃任务数

**Grafana 监控**：
- 指标：`asynq_active_workers`
- 告警：`worker_count = 0` 持续 2 分钟

### 3.6 紧急停止 Asynq

**场景**：系统异常，需要紧急停止任务处理

**方式一：K8s 热更新（推荐）**
```bash
kubectl set env deployment/app-server -c app-server \
  ASYNQ_POLLING_ENABLED=false UPDATE_TASK=true
```

**方式二：优雅关闭**
```bash
# 发送 SIGTERM，等待当前任务完成
kill -SIGTERM <pid>
```

### 3.7 迁移存量任务

**场景**：首次启用 Asynq，需要将数据库中的未完成任务迁移到队列

```bash
# 1. 临时启用迁移
kubectl set env deployment/app-server -c app-server \
  ASYNQ_MIGRATE_EXISTING=true

# 2. 等待 Pod 重启完成
kubectl rollout status deployment/app-server

# 3. 查看迁移日志
kubectl logs -l app=app-server --tail=100 | grep "migrate"

# 4. 恢复配置
kubectl set env deployment/app-server -c app-server \
  ASYNQ_MIGRATE_EXISTING=false
```

**注意**：
- 迁移使用分布式锁，多 Pod 只会执行一次
- 建议在低峰期执行

---

## 四、高级特性

### 4.1 熔断器机制

系统实现了两级熔断保护：

| 熔断器类型 | 作用域 | 触发条件 | 恢复条件 |
|-----------|--------|---------|---------|
| **服务熔断器** | 单个服务实例 | 连续失败 5 次 | 30s 后半开，连续成功 2 次恢复 |
| **平台熔断器** | 平台级别 | 连续失败 5 次 | 30s 后半开，连续成功 2 次恢复 |

**熔断状态**：

| 状态 | 说明 | 行为 |
|------|------|------|
| `CLOSED` | 正常 | 允许所有请求 |
| `OPEN` | 熔断中 | 拒绝所有请求 |
| `HALF_OPEN` | 探测恢复 | 允许有限请求（3个） |

**查看熔断器状态**：
```bash
kubectl logs -l app=app-server --tail=100 | grep "CircuitBreaker"

# 示例日志
# [CircuitBreaker] service_1: CLOSED -> OPEN (failures=5)
# [CircuitBreaker] service_1: OPEN -> HALF_OPEN
# [CircuitBreaker] service_1: HALF_OPEN -> CLOSED
```

### 4.2 背压控制

系统通过背压控制防止队列过载：

| 状态 | 队列利用率 | 行为 |
|------|-----------|------|
| `NORMAL` | < 70% | 正常处理 |
| `WARNING` | 70% ~ 90% | 记录警告日志 |
| `CRITICAL` | >= 90% | 拒绝新入队请求 |

### 4.3 系统过载时的降级操作

**方案 1：降低并发数**
```bash
kubectl set env deployment/app-server -c app-server \
  ASYNQ_CONCURRENCY=5
```

**方案 2：禁用 Asynq，使用传统轮询**
```bash
kubectl set env deployment/app-server -c app-server \
  ASYNQ_POLLING_ENABLED=false UPDATE_TASK=true
```

**方案 3：扩容 Pod**
```bash
kubectl scale deployment/app-server --replicas=5
```

---

## 五、故障排查

### 5.1 任务堆积（Pending 过多）

**症状**：`pending` 任务数持续增长

**Web UI 排查**：
1. `Queues` 查看各队列 pending 数量
2. `Servers` 检查 Worker 数量是否正常

**Grafana 排查**：
- 查看 `asynq_queue_size` 趋势
- 查看 `asynq_active_workers` 是否 > 0

**解决方案**：
- 增加并发数：`ASYNQ_CONCURRENCY=30`
- 扩容 Pod：`kubectl scale deployment/app-server --replicas=5`

### 5.2 任务重试过多（Retry 过多）

**症状**：`retry` 队列任务数持续增长

**Web UI 排查**：
1. `Queues` -> `scheduled` -> `Retry`
2. 点击任务查看 `Last Error`

**常见原因**：
- 外部 API 超时或返回错误
- 网络抖动
- 外部服务限流

### 5.3 Worker 不消费任务

**症状**：任务入队成功，但 `active` 始终为 0

**Web UI 排查**：
- `Servers` 检查 Worker 列表是否为空

**日志排查**：
```bash
kubectl logs -l app=app-server --tail=100 | grep "Asynq"
# 应该看到: [Asynq] Polling system initialized successfully
```

**解决方案**：
- 确认 `ASYNQ_POLLING_ENABLED=true`
- 检查 Redis 连接字符串是否正确
- 重启服务

### 5.4 任务超时

**症状**：任务长时间停留在 `active` 状态

**Web UI 排查**：
1. `Queues` -> `{队列名}` -> `Active`
2. 查看 `Started At` 时间

**解决方案**：
- 检查外部 API 是否变慢
- 检查网络延迟

---

## 六、配置参考

### 6.1 环境变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `ASYNQ_POLLING_ENABLED` | `true` | 是否启用 Asynq 事件驱动轮询 |
| `ASYNQ_REDIS_DB` | `0` | Asynq 使用的 Redis 数据库编号 |
| `ASYNQ_CONCURRENCY` | `10` | Worker 并发数 |
| `ASYNQ_MIGRATE_EXISTING` | `false` | 启动时是否迁移存量任务 |
| `ASYNQ_READONLY` | `false` | Asynqmon 只读模式 |
| `PROMETHEUS_ADDRESS` | - | Prometheus 地址（用于 Metrics 图表） |
| `UPDATE_TASK` | `true` | 是否启用传统轮询（兜底） |
| `TASK_POLL_INTERVAL` | `15` | 传统轮询基础间隔（秒） |

### 6.2 任务重试配置

| 平台类型 | 初始延迟 | 最大延迟 | 最大重试 |
|---------|---------|---------|---------|
| 长时任务 | 10s | 30s | 120 |
| 中时任务 | 5s | 20s | 120 |
| 短时任务 | 3s | 15s | 120 |
| 默认 | 10s | 30s | 120 |

### 6.3 告警阈值

| 告警名称 | 条件 | 严重级别 |
|---------|------|---------|
| 队列堆积 | pending > 500 持续 5 分钟 | Warning |
| 队列严重堆积 | pending > 2000 持续 5 分钟 | Critical |
| 重试过多 | retry > 50 持续 10 分钟 | Warning |
| 死信任务增加 | archived 增量 > 10/小时 | Warning |
| Worker 离线 | worker_count = 0 持续 2 分钟 | Critical |
| 背压触发 | backpressure_state != NORMAL | Warning |

---

## 附录

### A. 日志关键字

| 关键字 | 说明 |
|--------|------|
| `[Asynq] Polling system initialized` | Asynq 启动成功 |
| `[Asynq] Task enqueued` | 任务入队 |
| `[Asynq] Task completed` | 任务完成 |
| `[Asynq] Task failed` | 任务失败 |
| `[CircuitBreaker]` | 熔断器状态变化 |
| `[Backpressure]` | 背压状态变化 |
| `[兜底轮询]` | 传统轮询执行 |
| `ErrDuplicateTask` | 重复入队（正常，静默跳过） |
| `[CancelTaskAndRefund]` | 手动取消任务+资源回退 |
| `[BatchCancelTaskAndRefund]` | 批量取消任务+资源回退 |

### B. 任务状态与重试策略

以下是数据库中任务的状态与 Asynq 重试策略对照：

| 状态 | 说明 | 重试策略 | 延迟序列 | 退出条件 | 退出处理 |
|------|------|---------|---------|---------|---------|
| `SUCCESS` | 成功 | 出队 | - | 立即 | - |
| `FAILURE` | 失败 | 出队 | - | 立即 | - |
| `IN_PROGRESS` | 处理中 | 固定间隔 | 3s → 3s → 3s → ... | 超时 | 标记失败+资源回退 |
| `SUBMITTED` | 已提交外部 | 指数退避 | 3→6→12→24→30s | 超时 | 标记失败+资源回退 |
| `QUEUED` | 外部排队中 | 指数退避 | 3→6→12→24→30s | 超时 | 标记失败+资源回退 |
| `UNKNOWN` | 未知 | 指数退避 | 5→10→20s | 3 次重试 | 标记失败+资源回退 |
| `""` | 空状态 | 立即出队 | - | 立即 | 标记失败+资源回退 |
| `NOT_START` | 未开始 | 指数退避 | 默认策略 | 不处理 | 不处理 |

### C. 相关文档

- [Asynq 官方文档](https://github.com/hibiken/asynq)
- [Asynqmon 官方文档](https://github.com/hibiken/asynqmon)

---

**文档版本**: v1.0.0
**最后更新**: 2026-01-28
