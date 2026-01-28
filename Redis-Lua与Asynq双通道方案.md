# Redis Lua 与 Asynq 双通道方案

> 本文档描述一种通用的资源变更同步架构：使用 Redis Lua 保证原子性，Asynq 作为快速通道，Redis List 作为兜底保险。

## 1. 方案概述

### 1.1 核心思想

> **Lua 原子操作完全不变**，Asynq 只是在 Go 层"锦上添花"，Redis List 从"主通道"变成"兜底保险"。

### 1.2 设计原则

| 原则 | 说明 |
|-----|------|
| **原子性优先** | Lua 脚本保持不变，确保变更 + 记录的原子性 |
| **双通道冗余** | Asynq 快速通道 + List 兜底通道，数据不丢失 |
| **幂等保证** | MySQL 唯一索引防止重复处理 |
| **渐进式升级** | 可平滑迁移，不影响现有业务 |

---

## 2. 架构设计

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           变更请求                                       │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                    Lua 脚本（完全不变）                                   │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  1. 幂等检查（request_id）                                       │    │
│  │  2. 资源检查                                                     │    │
│  │  3. 变更资源：HINCRBY resource -amount                           │    │
│  │  4. 记录日志：RPUSH changelog  ←── 兜底记录，保证原子性           │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                    Go 代码（新增逻辑）                                    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  异步 Enqueue Asynq Task（快速通道，允许失败）                    │    │
│  │  - 使用 request_id 作为 TaskID，天然去重                         │    │
│  │  - 失败不阻塞主流程，有 List 兜底                                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└──────────────┬──────────────────────────────────────────┬───────────────┘
               │                                          │
               ↓                                          ↓
┌──────────────────────────┐              ┌──────────────────────────────┐
│     Asynq Worker         │              │     Fallback Job             │
│     (快速通道)           │              │     (兜底通道)               │
│  ┌────────────────────┐  │              │  ┌────────────────────────┐  │
│  │ • 实时处理单条任务 │  │              │  │ • 每分钟定时触发       │  │
│  │ • 低延迟（毫秒级） │  │              │  │ • 扫描 Redis List      │  │
│  │ • 自动重试机制     │  │              │  │ • 批量处理未消费日志   │  │
│  │ • 最多重试 3 次    │  │              │  │ • 处理 Asynq 遗漏      │  │
│  └────────────────────┘  │              │  └────────────────────────┘  │
└──────────────┬───────────┘              └──────────────┬───────────────┘
               │                                          │
               └──────────────────┬───────────────────────┘
                                  │
                                  ↓
                    ┌──────────────────────────┐
                    │        MySQL             │
                    │  ┌────────────────────┐  │
                    │  │ 幂等写入            │  │
                    │  │ INSERT IGNORE      │  │
                    │  │ 防止重复处理       │  │
                    │  └────────────────────┘  │
                    └──────────────────────────┘
```

### 2.2 数据流对比

| 阶段 | 原方案 | 现方案 |
|-----|-------|--------|
| **Lua 脚本** | 变更 + RPUSH List | **完全不变** |
| **Redis List** | 主同步通道 | 兜底通道 |
| **SyncWorker** | 定时消费 List → MySQL | 被 Asynq Fallback Job 替代 |
| **Asynq Task** | 无 | **新增**，快速通道 |
| **MySQL 写入** | 直接更新 | 幂等写入（INSERT IGNORE） |

---

## 3. 核心组件

### 3.1 Lua 脚本（不变）

```lua
-- 资源变更脚本（保持原有逻辑不变）
local key = KEYS[1]
local amount = tonumber(ARGV[1])
local request_id = ARGV[2]

-- 1. 幂等检查
local idempotent_key = "resource:idempotent:" .. request_id
if redis.call('EXISTS', idempotent_key) == 1 then
    return {-3, cached, "duplicate request"}
end

-- 2. 资源检查
local current = tonumber(redis.call('HGET', key, 'balance'))
if current < amount then
    return {-2, current, "insufficient resource"}
end

-- 3. 执行变更
local new_balance = current - amount
redis.call('HSET', key, 'balance', new_balance)
redis.call('HINCRBY', key, 'used', amount)

-- 4. 记录幂等键
redis.call('SETEX', idempotent_key, idempotent_ttl, new_balance)

-- 5. 记录变更日志（兜底）  ←── 这一步保证原子性
local log_entry = string.format(
    '{"user_id":%s,"delta":%d,"request_id":"%s","timestamp":%s}',
    user_id, -amount, request_id, timestamp
)
redis.call('RPUSH', 'resource:changelog', log_entry)

return {0, new_balance, "success"}
```

### 3.2 Go 变更逻辑（新增 Asynq 入队）

```go
func (s *ServiceWithAsynq) DeductWithAsynq(ctx context.Context,
    userID int, amount int64, requestID, reason string) (int64, error) {

    // ========================================
    // 步骤 1：Lua 原子变更（完全不变）
    // 会同时写 changelog 到 Redis List 作为兜底
    // ========================================
    balance, err := s.Service.Deduct(ctx, userID, amount, requestID, reason)
    if err != nil {
        return 0, err
    }

    // ========================================
    // 步骤 2：异步发送 Asynq 任务（新增）
    // 这是快速通道，允许失败，因为有 List 兜底
    // ========================================
    if s.useAsynq && s.asynqManager != nil {
        payload := &ResourceSyncPayload{
            UserID:    userID,
            Delta:     -amount,
            RequestID: requestID,
            Reason:    reason,
            Timestamp: time.Now().Unix(),
        }

        // 异步发送，不阻塞主流程
        go func() {
            task := asynq.NewTask(TaskTypeResourceSync, payload,
                asynq.TaskID(requestID),  // 使用 request_id 去重
                asynq.MaxRetry(3),
                asynq.Queue(QueueCritical),
            )
            if err := s.client.Enqueue(task); err != nil {
                // 记录日志，但不影响主流程（有 List 兜底）
                log.Printf("[Asynq] enqueue failed: %v (fallback active)", err)
            }
        }()
    }

    return balance, nil
}
```

### 3.3 Asynq Worker（快速通道）

```go
// 处理单条资源同步任务
func (m *AsynqSyncManager) handleResourceSync(ctx context.Context, t *asynq.Task) error {
    var payload ResourceSyncPayload
    if err := json.Unmarshal(t.Payload(), &payload); err != nil {
        return fmt.Errorf("unmarshal: %w", err)
    }

    // 幂等写入 MySQL
    return m.syncToMySQL(ctx, []ResourceSyncPayload{payload})
}
```

### 3.4 Fallback Job（兜底通道）

```go
// 定时扫描 Redis List，处理遗漏的日志
func (m *AsynqSyncManager) handleResourceFallback(ctx context.Context, t *asynq.Task) error {
    // 1. Peek 获取未处理的日志（不删除）
    logs, err := m.svc.PeekChangeLogs(ctx, 1000)
    if err != nil || len(logs) == 0 {
        return err
    }

    // 2. 批量同步到 MySQL（幂等）
    if err := m.syncToMySQL(ctx, logs); err != nil {
        return err
    }

    // 3. 同步成功后，Ack 删除已处理的日志
    return m.svc.AckChangeLogs(ctx, len(logs))
}
```

### 3.5 MySQL 幂等写入

```go
func (m *AsynqSyncManager) syncToMySQL(ctx context.Context, payloads []ResourceSyncPayload) error {
    return db.Transaction(func(tx *gorm.DB) error {
        for _, p := range payloads {
            // INSERT IGNORE：如果 request_id 已存在则跳过
            result := tx.Exec(`
                INSERT IGNORE INTO resource_sync_logs
                (request_id, user_id, delta, reason, synced_at)
                VALUES (?, ?, ?, ?, NOW())
            `, p.RequestID, p.UserID, p.Delta, p.Reason)

            if result.Error != nil {
                return result.Error
            }

            // 只有插入成功（RowsAffected > 0）才更新用户资源
            if result.RowsAffected > 0 {
                tx.Exec(`
                    UPDATE users
                    SET balance = balance + ?, used = used - ?
                    WHERE id = ? AND deleted_at IS NULL
                `, p.Delta, p.Delta, p.UserID)
            }
            // RowsAffected == 0 说明已处理过，跳过
        }
        return nil
    })
}
```

---

## 4. 幂等机制

### 4.1 三层幂等保证

```
┌─────────────────────────────────────────────────────────────────┐
│  第一层：Redis 幂等键                                            │
│  └─ Lua 脚本内检查 resource:idempotent:{request_id}             │
│     防止同一请求重复处理                                         │
├─────────────────────────────────────────────────────────────────┤
│  第二层：Asynq TaskID                                            │
│  └─ 使用 request_id 作为 TaskID                                 │
│     Asynq 自动去重，相同 TaskID 不会重复入队                     │
├─────────────────────────────────────────────────────────────────┤
│  第三层：MySQL 唯一索引                                          │
│  └─ resource_sync_logs.request_id UNIQUE                        │
│     INSERT IGNORE 防止重复写入                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 幂等表设计

```sql
CREATE TABLE resource_sync_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    request_id VARCHAR(64) NOT NULL,      -- 幂等键
    user_id INT NOT NULL,
    delta BIGINT NOT NULL,                -- 变更量（正数增加，负数减少）
    reason VARCHAR(255) DEFAULT '',
    synced_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE INDEX idx_request_id (request_id),  -- 幂等唯一索引
    INDEX idx_user_id (user_id),
    INDEX idx_synced_at (synced_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 5. 双通道协作

### 5.1 正常流程（Asynq 快速通道）

```
请求 → Lua变更+写List → Go异步Enqueue → Asynq Worker处理 → MySQL写入
                ↓
          List中有记录（等待清理）
                ↓
          Fallback Job 发现已处理（INSERT IGNORE 跳过）→ Ack清理List
```

### 5.2 Asynq 失败流程（List 兜底）

```
请求 → Lua变更+写List → Go Enqueue失败（网络/Redis故障）
                ↓
          List中有记录
                ↓
          Fallback Job 定时扫描 → 发现未处理 → MySQL写入 → Ack清理List
```

### 5.3 时序图

```
     Client          Lua脚本         Go代码        Asynq         Fallback       MySQL
        │               │              │             │              │             │
        │──变更请求────→│              │             │              │             │
        │               │──变更+写List─│             │              │             │
        │               │←─返回结果────│             │              │             │
        │←──返回结果────│              │             │              │             │
        │               │              │──Enqueue──→│              │             │
        │               │              │             │──处理任务───────────────→│
        │               │              │             │              │             │
        │               │              │             │     (每分钟)  │             │
        │               │              │             │              │──扫描List──→│
        │               │              │             │              │  (已处理,跳过)
        │               │              │             │              │──Ack清理───→│
```

---

## 6. 配置与部署

### 6.1 Asynq 配置

```go
// Asynq Server 配置
server := asynq.NewServer(
    asynq.RedisClientOpt{Addr: "localhost:6379"},
    asynq.Config{
        Concurrency: 10,                    // 并发 Worker 数
        Queues: map[string]int{
            "critical": 6,                  // 高优先级
            "default":  3,                  // 默认优先级
        },
        RetryDelayFunc: func(n int, e error, t *asynq.Task) time.Duration {
            return time.Duration(1<<uint(n)) * time.Second  // 指数退避
        },
    },
)

// 定时任务配置
scheduler := asynq.NewScheduler(redisOpt, nil)
scheduler.Register(
    "@every 1m",                            // 每分钟执行
    asynq.NewTask("resource:fallback", nil),
    asynq.Queue("default"),
)
```

### 6.2 环境变量

```bash
# Asynq 相关
ASYNQ_ENABLED=true                    # 是否启用 Asynq
ASYNQ_REDIS_ADDR=localhost:6379       # Redis 地址
ASYNQ_CONCURRENCY=10                  # Worker 并发数

# 兜底任务
ASYNQ_FALLBACK_INTERVAL=1m            # 兜底扫描间隔
ASYNQ_FALLBACK_BATCH_SIZE=1000        # 每次扫描数量
```

### 6.3 依赖安装

```bash
# 安装 Asynq
go get github.com/hibiken/asynq

# 安装监控 UI（可选）
go install github.com/hibiken/asynqmon@latest
asynqmon --redis-addr=localhost:6379 --port=8080
```

---

## 7. 监控与运维

### 7.1 Asynqmon 监控 UI

```
http://localhost:8080

功能：
├── 队列状态：待处理、处理中、已完成、失败
├── 任务详情：查看任务 payload、重试次数、错误信息
├── 实时统计：处理速率、成功率、延迟
└── 手动操作：重试失败任务、删除任务
```

### 7.2 关键指标

| 指标 | 说明 | 告警阈值 |
|-----|------|---------|
| `asynq_queue_size` | 队列积压数 | > 10000 |
| `asynq_retry_count` | 重试次数 | > 3 |
| `redis_list_length` | List 兜底积压 | > 1000 |
| `sync_latency` | 同步延迟 | > 5s |

### 7.3 日志示例

```
[Asynq] enqueue task resource:sync, request_id=req-001
[Asynq] task completed, request_id=req-001, latency=15ms
[Fallback] scanned 0 pending logs (all processed by Asynq)
[Fallback] scanned 50 pending logs, synced to MySQL
```

---

## 8. 方案对比

### 8.1 与原方案对比

| 维度 | 原方案 (Redis List) | 现方案 (双写 + Asynq) |
|-----|--------------------|-----------------------|
| **Lua 脚本** | 变更 + RPUSH | **完全不变** |
| **原子性** | ✅ | ✅ |
| **同步延迟** | 秒级（定时批量） | 毫秒级（实时处理） |
| **监控** | 需自建 | ✅ asynqmon |
| **重试机制** | 简单 | 完善（指数退避） |
| **数据安全** | Peek + Ack | 双通道 + 幂等 |
| **复杂度** | 低 | 中 |
| **依赖** | 无新增 | +asynq |

### 8.2 适用场景

| 场景 | 推荐方案 |
|-----|---------|
| 追求简洁、依赖少 | 原方案 (Redis List) |
| 需要监控 UI | 双通道方案 |
| 需要低延迟同步 | 双通道方案 |
| 需要完善重试 | 双通道方案 |
| 已有 Asynq 基础设施 | 双通道方案 |

---

## 9. 总结

### 9.1 核心要点

1. **Lua 脚本完全不变**：原子性得到保证
2. **双通道冗余**：Asynq 快速 + List 兜底
3. **三层幂等**：Redis 幂等键 + Asynq TaskID + MySQL 唯一索引
4. **平滑迁移**：可渐进式升级，支持回滚

### 9.2 一句话总结

> **在不破坏原有 Lua 原子性的前提下，通过 Asynq 双写获得更好的实时性、监控能力和重试机制，同时保留 Redis List 作为兜底保险。**

### 9.3 参考文件结构

```
service/resource/
├── lua_scripts.go           # Lua 脚本（不变）
├── service.go               # 核心服务（不变）
├── sync_worker.go           # 原 SyncWorker（迁移后可移除）
├── reconciler.go            # 对账服务（保留）
├── asynq_sync.go            # Asynq 同步管理器（新增）
├── asynq_integration.go     # Asynq 集成层（新增）
```

---

**文档版本**: v1.0.0
**最后更新**: 2026-01-28
