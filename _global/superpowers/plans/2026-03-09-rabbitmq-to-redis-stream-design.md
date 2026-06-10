# 用户绘图任务系统重构设计方案：从 RabbitMQ 到 Redis Stream

## 文档信息

- **创建日期**：2026-03-09
- **版本**：v1.0
- **状态**：已批准

## 一、背景与目标

### 1.1 当前系统问题

当前系统使用 **RabbitMQ + app_mq_message 表** 实现消息队列和补偿机制，存在以下核心问题：

1. **提前 ACK 风险**：`DoUserDrawTask.java:106` 在消息处理完成前就 ACK，导致消息丢失风险
2. **DB 写入开销**：每条消息都要同步写入 `app_mq_message` 表，增加延迟（百毫秒级）
3. **定时任务过多**：4 个核心补偿定时任务维护成本高
   - `MqMessageScheduledScan` - 扫描 RECEIVED 状态超时消息
   - `MessageResendScheduler` - 扫描 SENT 状态超时消息
   - `MqMessageCleanupScheduler` - 清理异常消息
   - `MqMessageArchiveTask` - 归档已完成消息
4. **续期复杂性**：`UserDrawRunnable` 每 30 秒续期一次，增加系统复杂度
5. **轮询低效**：`UserDrawTaskReceiver` 使用 `while(running) + Thread.sleep(3000)` 轮询

### 1.2 重构目标

引入 **Redis Stream 消费者组 + XCLAIM 抢占机制**，实现：

- ✅ 消息可靠性由 Redis Stream 原生保证（PEL 机制）
- ✅ 发送延迟从百毫秒级降至毫秒级（无需写 DB）
- ✅ 废弃 4 个补偿定时任务，简化架构
- ✅ 自动超时抢占，无需手动续期
- ✅ 阻塞读取替代轮询，降低 CPU 消耗

## 二、方案选择

### 2.1 不推荐的方案

**方案 A：使用 StreamMessageListenerContainer（Spring 自动推送）**

- ❌ 任务会被推送到 JVM 内存，造成内存堆积
- ❌ 任务粘滞问题：已推送的任务无法被其他节点处理
- ❌ 水平扩展困难：新节点启动后无法立即分担压力

**方案 B：保留 RabbitMQ + 优化补偿机制**

- ❌ 仍需维护 app_mq_message 表和 4 个定时任务
- ❌ 提前 ACK 风险依然存在
- ❌ DB 写入开销无法消除

### 2.2 推荐方案：Redis Stream + 自定义轮询模式

**核心优势**：

1. **零内存排队** - 任务留在 Redis Stream，不在 JVM 堆积
2. **能者多劳** - 空闲节点立即拉取，忙碌节点停止拉取
3. **真正的水平扩展** - 新节点启动后立即分担压力
4. **架构简化** - 废弃 4 个定时任务和 app_mq_message 表

## 三、核心机制设计

### 3.1 消息流转机制

```
用户提交任务
    ↓
PlotServiceImpl.sendToRedisStream()
    ↓ XADD（消息体：userDrawId, userId, timestamp, retryCount=0）
Redis Stream: stream:user_draw_tasks
    ↓
Consumer Group: group:task_service
    ├─ PEL（未 ACK 的消息）
    └─ 多个消费者实例（task-instance-1, task-instance-2...）
    ↓
UserDrawStreamConsumer（自定义轮询线程）
    ├─ while(running) 循环
    ├─ isExecutorBusy() 检查 → 忙碌则 sleep(1000)
    ├─ 空闲则 XREADGROUP BLOCK 3000 COUNT 1
    ├─ 提交到线程池（不 ACK）
    └─ UserDrawRunnable 执行完成后才 XACK
    ↓
XAUTOCLAIM 定时任务（每 30 秒抢占超时消息）
```

### 3.2 背压控制机制

**关键设计**：线程池满时停止拉取，实现零内存排队

```java
// 每次循环都检查线程池状态
if (isExecutorBusy()) {
    Thread.sleep(1000);  // 线程池满，停止拉取
    continue;
}

// 线程池空闲，精准拉取 1 条消息
XREADGROUP ... COUNT 1
```

**效果**：
- 任务留在 Redis Stream 中，不在 JVM 内存堆积
- 其他空闲节点可以立即获取这些任务
- 真正的水平扩展，无任务粘滞

### 3.3 可靠性保证机制

**消息不丢失**：
- **发送**：XADD 同步返回消息 ID，发送即确认
- **消费**：只有 UserDrawRunnable 执行完成才 XACK（无论成功还是失败）
- **超时**：XAUTOCLAIM 每 30 秒抢占超过 60 秒未 ACK 的消息
- **死信**：重试 5 次后强制 ACK + 日志 + 飞书告警 + 更新任务状态为 FAIL

**幂等性保证**：
- 使用 `"user_draw_lock_" + userDrawId` 作为分布式锁 key
- 同一任务不会被重复处理

## 四、关键设计决策

基于需求讨论，以下是最终确定的关键设计决策：

### 4.1 ACK 时机（重要改进）

**决策**：UserDrawRunnable 执行完成后才 XACK（无论成功还是失败）

**原因**：
- 更强的可靠性：应用崩溃时任务不会丢失
- 真正的"至少执行一次"语义
- 避免线程池队列中的任务因重启而丢失

**实现**：
```java
// UserDrawRunnable.run()
@Override
public void run() {
    try {
        // 执行业务逻辑
        doDrawTask();
    } finally {
        // 无论成功还是失败，都在这里 ACK
        redisTemplate.opsForStream().acknowledge(
            RedisStreamConfig.STREAM_USER_DRAW_TASKS,
            RedisStreamConfig.GROUP_TASK_SERVICE,
            recordId
        );
    }
}
```

### 4.2 消息体简化

**决策**：用 userDrawId 替代 correlationId

**消息体结构**：
```java
{
  "userDrawId": 123,
  "userId": 456,
  "timestamp": 1234567890,
  "retryCount": 0
}
```

**相应调整**：
- 分布式锁 key：`"user_draw_lock_" + userDrawId`
- 日志追踪：`log.info("任务处理: userDrawId={}", userDrawId)`

### 4.3 死信处理

**决策**：组合方案（日志 + 飞书告警 + 更新任务状态为 FAIL）

**实现**：
1. 记录死信日志（包含完整消息体）
2. 发送飞书告警
3. 更新任务状态为 FAIL
4. 强制 XACK 防止继续占用 PEL

### 4.4 app_mq_message 表

**决策**：直接废弃，不再写入

**原因**：
- 简化架构，减少维护成本
- Redis Stream PEL 机制已提供可靠性保证
- 历史数据保留用于查询

### 4.5 线程池配置

**决策**：保持现有配置和 isExecutorBusy() 逻辑

**判断标准**：`activeCount >= corePoolSize` 时认为线程池忙碌

### 4.6 Redis 基础设施

**决策**：使用现有 Redis（版本 >= 5.0，AOF 持久化）

**说明**：
- 已启用 AOF 持久化，满足可靠性要求
- 暂无主从复制，单点故障风险可接受

## 五、实施路径概览

### 5.1 第一阶段：基础设施与生产者（system 模块）

1. 创建 `RedisStreamConfig` 配置类
2. 重构 `PlotServiceImpl` 发送逻辑
   - 删除 `mqMessageRepository.save()`
   - 新增 `sendToRedisStream()` 方法
   - 简化消息体（移除 correlationId）
3. 处理 `ConfirmReturnCallback` 回调机制
   - 添加消息类型判断，跳过用户绘图任务

### 5.2 第二阶段：消费者重构（task 模块）

1. 创建 `UserDrawStreamConsumer`（自定义轮询模式）
   - 实现背压控制（isExecutorBusy）
   - 实现精准拉取（COUNT 1）
   - 实现 XAUTOCLAIM 定时任务
2. 修改 `DoUserDrawTask`
   - 新增 `processFromStream()` 方法
   - 传递 recordId 到 UserDrawRunnable
3. 修改 `UserDrawRunnable`
   - 在 finally 块中执行 XACK
   - 移除续期逻辑
4. 废弃 `UserDrawTaskReceiver`
5. 修改 `TaskInitializer` 启动逻辑

### 5.3 第三阶段：废弃补偿定时任务

注释掉以下定时任务：
1. `MqMessageScheduledScan`
2. `MessageResendScheduler`
3. `MqMessageCleanupScheduler`
4. `MqMessageArchiveTask`

### 5.4 第四阶段：状态管理与监控

1. 保持 UserDraw 状态管理不变
2. 配置 Redis Stream 自动清理（XTRIM MAXLEN）
3. 迁移监控指标（`SystemMetricsCollector`）
   - 分离 RabbitMQ 和 Redis Stream 监控
   - 新增 Stream 长度、PEL 长度、消费者数量监控

## 六、风险评估与缓解

### 6.1 风险矩阵

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|----------|
| Redis 宕机导致消息丢失 | 高 | 低 | AOF 持久化已启用 |
| Stream 内存占用过高 | 中 | 中 | MAXLEN 限制 + 监控 |
| 消费者实例重启导致消息重复 | 低 | 高 | 幂等性设计（分布式锁） |
| XAUTOCLAIM 抢占失败 | 中 | 低 | 多实例部署 + 监控 PEL |
| 毒丸消息导致死信堆积 | 中 | 低 | 死信处理机制 + 最大重试 5 次 |

### 6.2 监控指标

**关键指标**：
- Stream 长度（应 < 10000）
- PEL 长度（应 < 100）
- 消息处理延迟
- 消费者数量

**告警阈值**：
- Stream 长度 > 10000：警告级别
- PEL 长度 > 100：错误级别，发送飞书告警
- 消费者数量 = 0：严重级别

## 七、验证标准

### 7.1 功能验证

- [ ] 用户提交绘图任务成功
- [ ] 消息正确发送到 Redis Stream
- [ ] 消费者正确读取并处理消息
- [ ] 任务成功提交到线程池
- [ ] UserDrawRunnable 执行完成后才 ACK
- [ ] 任务执行完成后状态正确更新
- [ ] 失败任务会自动重试
- [ ] 超时消息会被其他消费者抢占
- [ ] 重复消息不会被重复处理（幂等性）

### 7.2 性能验证

- [ ] 发送延迟 < 10ms（当前 ~100ms）
- [ ] 消息处理延迟 P99 < 100ms
- [ ] Stream 长度稳定在 < 10000
- [ ] PEL 长度稳定在 < 50
- [ ] 线程池满时消费者停止拉取（背压控制）
- [ ] 新节点启动后立即分担任务（水平扩展）

### 7.3 可靠性验证

- [ ] Redis 重启后消息不丢失（AOF 持久化）
- [ ] 消费者重启后继续处理 PEL 中的消息
- [ ] 应用崩溃时任务不会丢失（ACK 时机正确）
- [ ] 线程池满时消息会等待重试
- [ ] 分布式锁失败时消息会重试

## 八、预期收益

### 8.1 性能提升

- **发送延迟**：从 ~100ms 降至 ~5ms（20x 提升）
- **CPU 使用率**：降低 ~30%（无轮询）
- **代码复杂度**：减少 ~40%（废弃 4 个定时任务 + 续期逻辑）

### 8.2 可维护性提升

- **定时任务**：从 4 个减少到 0 个
- **数据库表**：废弃 app_mq_message 表
- **代码行数**：减少 ~500 行

### 8.3 可靠性提升

- **消息丢失风险**：从"提前 ACK"降至"执行完成才 ACK"
- **补偿机制**：从"定时任务扫描"升级为"XAUTOCLAIM 自动抢占"
- **监控指标**：从"DB 查询"简化为"Redis 命令"

## 九、关键文件清单

### 9.1 需要修改的文件

**System 模块（生产者）**：
- `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`
- `goto/common/src/main/java/com/freedom/config/userDraw/ConfirmReturnCallback.java`
- `goto/system/src/main/java/com/freedom/monitor/SystemMetricsCollector.java`

**Task 模块（消费者）**：
- `goto/task/src/main/java/com/freedom/util/DoUserDrawTask.java`
- `goto/task/src/main/java/com/freedom/thread/UserDrawRunnable.java`
- `goto/task/src/main/java/com/freedom/bean/mq/rabbitmq/UserDrawTaskReceiver.java`
- `goto/task/src/main/java/com/freedom/runner/TaskInitializer.java`

**定时任务（废弃）**：
- `goto/system/src/main/java/com/freedom/quartz/task/MqMessageScheduledScan.java`
- `goto/system/src/main/java/com/freedom/quartz/task/MessageResendScheduler.java`
- `goto/system/src/main/java/com/freedom/scheduled/MqMessageCleanupScheduler.java`
- `goto/system/src/main/java/com/freedom/quartz/task/MqMessageArchiveTask.java`

### 9.2 需要新建的文件

**配置类**：
- `freedom-public/eladmin-common/src/main/java/com/freedom/config/RedisStreamConfig.java`

**消费者**：
- `goto/task/src/main/java/com/freedom/bean/mq/redis/UserDrawStreamConsumer.java`

### 9.3 保持不变的文件

- `goto/task/src/main/java/com/freedom/bean/mq/rabbitmq/UserDrawTaskCancelConsumer.java`（取消任务继续使用 RabbitMQ）

## 十、批准记录

- **设计批准日期**：2026-03-09
- **批准人**：[待填写]
- **下一步**：创建详细实施计划
