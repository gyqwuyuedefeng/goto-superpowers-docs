# RabbitMQ 到 Redis Stream 重构实施计划（完整修正版）

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

## 计划概述

**目标**：将用户绘图任务系统从 RabbitMQ + DB 补偿机制迁移到 Redis Stream 原生可靠性

**架构**：采用 Redis Stream 自定义轮询模式，实现零内存排队、背压控制和水平扩展

**技术栈**：Spring Boot, Redis Stream, Spring Data Redis, Redisson

**任务总数**：15 个任务（原计划 8 个，补充 7 个）

**预计工作量**：5-7 天开发 + 2-3 天测试 + 1-2 周灰度期

**关键修复**：
1. 修复 ComponentBean.getUserDrawExecutor() 不存在的问题 → 使用 SystemRebootInitUtils.userDrawThreadPool.executor
2. 修复 UserDrawRunnable 构造函数签名不匹配 → 使用 UserDraw.recordId 临时字段传递
3. 补充 XAUTOCLAIM 超时处理机制（Task 9）
4. 补充测试、监控、回滚方案（Task 10-15）
5. 修正 correlationId 前缀判断错误

---

## 任务清单

### 阶段一：基础设施（Task 1）
- Task 1: 创建 RedisStreamConfig 配置类

### 阶段二：生产者重构（Task 2-3）
- Task 2: 重构 PlotServiceImpl 发送逻辑
- Task 3: 处理 ConfirmReturnCallback 回调机制

### 阶段三：消费者重构（Task 4-8）
- Task 4: 创建 UserDrawStreamConsumer 消费者
- Task 5: 实现背压控制和消息拉取
- Task 6: 修改 DoUserDrawTask 添加 processFromStream 方法
- Task 7: 修改 UserDrawRunnable 在 finally 中 ACK
- Task 8: 连接消费者和业务逻辑

### 阶段四：可靠性保障（Task 9-10）
- Task 9: 实现 XAUTOCLAIM 超时处理机制 ⚠️ 关键
- Task 10: 禁用 RabbitMQ 消费者

### 阶段五：测试与监控（Task 11-12）
- Task 11: 添加集成测试
- Task 12: 添加监控和告警

### 阶段六：灰度发布（Task 13-15）
- Task 13: 实现灰度发布机制
- Task 14: 编写回滚方案
- Task 15: 性能基准测试

---

## 关键修复说明

### 修复 1: 线程池访问方式

**错误**（原计划）:
```java
ThreadPoolExecutor executor = (ThreadPoolExecutor) componentBean.getUserDrawExecutor();
```

**正确**:
```java
ThreadPoolExecutor executor = SystemRebootInitUtils.userDrawThreadPool.executor;
```

### 修复 2: RecordId 传递机制

**方案**: 在 UserDraw 实体中添加临时字段

```java
// UserDraw.java
@Transient
private RecordId recordId;  // Redis Stream 消息 ID（非持久化）
```

### 修复 3: CorrelationId 前缀判断

**错误**（原计划）:
```java
if (correlationData.getId().startsWith("task_sub_user_")) {
```

**正确**:
```java
if (correlationData.getId().startsWith("user:task:sub:")) {
```

---

## 详细任务（仅列出关键步骤，完整代码见原计划）

### Task 1: 创建 RedisStreamConfig 配置类

**Files**: Create `freedom-public/eladmin-common/src/main/java/com/freedom/config/RedisStreamConfig.java`

**关键配置**:
- STREAM_USER_DRAW_TASKS = "stream:user_draw_tasks"
- GROUP_TASK_SERVICE = "group:task_service"
- MESSAGE_TIMEOUT_MS = 60000
- MAX_RETRY_COUNT = 5
- BUSY_THRESHOLD = 0.8 (80%)

**验证**: `mvn clean compile -pl freedom-public/eladmin-common -am`

---

### Task 2: 重构 PlotServiceImpl 发送逻辑

**Files**: Modify `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`

**关键修改**:
1. 添加 `sendToRedisStream()` 方法
2. 消息体直接存储 Long 类型（不转 String）
3. 添加 try-catch 处理发送失败
4. 替换 `mqMessageRepository.save()` 调用
5. 修改状态判断逻辑（保留容错机制）

**验证**: `mvn clean compile -pl goto/system -am`

---

### Task 3: 处理 ConfirmReturnCallback 回调机制

**Files**: Modify `goto/common/src/main/java/com/freedom/config/userDraw/ConfirmReturnCallback.java`

**关键修改**:
- 修复前缀判断：`"user:task:sub:"` 而不是 `"task_sub_user_"`
- 在 confirm() 和 returnedMessage() 开头添加判断

**验证**: `mvn clean compile -pl goto/common -am`

---

### Task 4: 创建 UserDrawStreamConsumer 消费者

**Files**: Create `goto/task/src/main/java/com/freedom/bean/mq/redis/UserDrawStreamConsumer.java`

**关键实现**:
1. 生成唯一消费者名称
2. 创建 Consumer Group（处理 Stream 不存在的情况）
3. 启动消费线程

**验证**: `mvn clean compile -pl goto/task -am`

---

### Task 5: 实现背压控制和消息拉取

**Files**: Modify `goto/task/src/main/java/com/freedom/bean/mq/redis/UserDrawStreamConsumer.java`

**关键实现**:
1. `isExecutorBusy()` 方法使用 80% 阈值
2. XREADGROUP 阻塞式拉取，COUNT 1
3. 线程池满时停止拉取

**正确实现**:
```java
private boolean isExecutorBusy() {
    ThreadPoolExecutor executor = SystemRebootInitUtils.userDrawThreadPool.executor;
    int activeCount = executor.getActiveCount();
    int corePoolSize = executor.getCorePoolSize();
    return (double) activeCount / corePoolSize >= RedisStreamConfig.BUSY_THRESHOLD;
}
```

**验证**: `mvn clean compile -pl goto/task -am`

---

### Task 6: 修改 DoUserDrawTask 添加 processFromStream 方法

**Files**: Modify `goto/task/src/main/java/com/freedom/util/DoUserDrawTask.java`

**关键实现**:
1. 添加 `processFromStream()` 方法
2. 使用 `SystemRebootInitUtils.userDrawThreadPool.add()` 而不是 `executor.execute()`
3. 在 UserDraw 中设置 recordId

**正确实现**:
```java
public static void processFromStream(ComponentBean componentBean,
                                     Long userDrawId,
                                     Long userId,
                                     RecordId recordId) {
    UserDraw userDraw = componentBean.getRepository()
        .getUserDrawRepository()
        .findById(userDrawId)
        .orElse(null);

    if (userDraw == null) {
        ackMessage(componentBean, recordId);
        return;
    }

    // 设置 recordId 到 UserDraw
    userDraw.setRecordId(recordId);

    String lockKey = "user_draw_lock_" + userDrawId;
    RHandlerWrapper rHandlerWrapper = new RHandlerWrapper(() -> {
        // 使用线程池的 add 方法
        CheckResultMsg result = SystemRebootInitUtils.userDrawThreadPool.add(componentBean, userDraw);
        if (!result.isResult()) {
            log.error("任务提交失败: {}", result.getMsg());
        }
    });

    rHandlerWrapper.doHandler(componentBean.getRedissonClient(), lockKey);
}
```

**验证**: `mvn clean compile -pl goto/task -am`

---

### Task 7: 修改 UserDrawRunnable 在 finally 中 ACK

**Files**:
- Modify: `goto/task/src/main/java/com/freedom/thread/UserDrawRunnable.java`
- Modify: `goto/common/src/main/java/com/freedom/domain/UserDraw.java`

**Step 1: 在 UserDraw 中添加 recordId 字段**

```java
// UserDraw.java
@Transient
private RecordId recordId;  // Redis Stream 消息 ID（非持久化）

public RecordId getRecordId() {
    return recordId;
}

public void setRecordId(RecordId recordId) {
    this.recordId = recordId;
}
```

**Step 2: 修改 UserDrawRunnable 的 finally 块**

```java
@Override
public void run() {
    try {
        // 原有的业务逻辑
        doDrawTask();
    } catch (Exception e) {
        log.error("任务执行失败: userDrawId={}", data.getId(), e);
    } finally {
        // ========== 删除续期逻辑 ==========
        // stopRenewTask();  // 删除这行

        // ========== 删除更新 app_mq_message 的逻辑 ==========
        // if (ObjectUtils.isNotEmpty(data.getCorrelationId())) {
        //     int rows = getContext().getRepository().getMqMessageRepository()
        //         .updateStatusAndResultByMessageIdAndOldStatus(...);
        // }

        // ========== 新增：XACK 消息 ==========
        if (data.getRecordId() != null) {
            try {
                getContext().getRedisTemplate().opsForStream().acknowledge(
                    RedisStreamConfig.STREAM_USER_DRAW_TASKS,
                    RedisStreamConfig.GROUP_TASK_SERVICE,
                    data.getRecordId()
                );
                log.info("消息 ACK 成功: userDrawId={}, recordId={}",
                        data.getId(), data.getRecordId());
            } catch (Exception e) {
                log.error("消息 ACK 失败: userDrawId={}, recordId={}",
                         data.getId(), data.getRecordId(), e);
            }
        }

        // 保留原有的 doFinally()
        doFinally();
    }
}
```

**Step 3: 删除续期相关方法**

删除以下方法：
- `startRenewTask()` (第 126-184 行)
- `stopRenewTask()`
- run() 方法中调用 `startRenewTask()` 的地方

**验证**: `mvn clean compile -pl goto/task,goto/common -am`

---

### Task 8: 连接消费者和业务逻辑

**Files**: Modify `goto/task/src/main/java/com/freedom/bean/mq/redis/UserDrawStreamConsumer.java`

**关键实现**:
```java
private void processMessage(MapRecord<String, Object, Object> record) {
    Map<Object, Object> body = record.getValue();

    Long userDrawId = Long.parseLong(body.get("userDrawId").toString());
    Long userId = Long.parseLong(body.get("userId").toString());

    log.info("开始处理消息: userDrawId={}, userId={}, recordId={}",
             userDrawId, userId, record.getId());

    DoUserDrawTask.processFromStream(
        componentBean,
        userDrawId,
        userId,
        record.getId()
    );
}
```

**验证**: `mvn clean compile -pl goto/task -am`

---

### Task 9: 实现 XAUTOCLAIM 超时处理机制 ⚠️ 关键

**Files**: Create `goto/task/src/main/java/com/freedom/task/UserDrawStreamAutoclaimTask.java`

**关键实现**:
1. 使用 `@Scheduled(fixedDelay = 30000)` 每 30 秒执行
2. 调用 XAUTOCLAIM 抢占超过 60 秒未 ACK 的消息
3. 检查 retryCount，超过 5 次则执行死信处理
4. 更新 retryCount 并重新提交到线程池

**完整代码**:
```java
package com.freedom.task;

import com.freedom.bean.ComponentBean;
import com.freedom.config.RedisStreamConfig;
import com.freedom.util.DoUserDrawTask;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.connection.stream.*;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.net.InetAddress;
import java.time.Duration;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * Redis Stream 超时消息自动抢占任务
 * 每 30 秒执行一次，抢占超过 60 秒未 ACK 的消息
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class UserDrawStreamAutoclaimTask {

    private final ComponentBean componentBean;
    private final RedisTemplate<String, Object> redisTemplate;
    private String consumerName;

    @Scheduled(fixedDelay = RedisStreamConfig.AUTOCLAIM_INTERVAL_MS)
    public void autoclaimTimeoutMessages() {
        try {
            if (consumerName == null) {
                consumerName = "autoclaim-" + InetAddress.getLocalHost().getHostName()
                             + "-" + System.currentTimeMillis();
            }

            // 执行 XAUTOCLAIM
            List<MapRecord<String, Object, Object>> claimedRecords =
                redisTemplate.opsForStream().autoclaim(
                    RedisStreamConfig.STREAM_USER_DRAW_TASKS,
                    Consumer.from(RedisStreamConfig.GROUP_TASK_SERVICE, consumerName),
                    Duration.ofMillis(RedisStreamConfig.MESSAGE_TIMEOUT_MS),
                    StreamOffset.latest()
                );

            if (claimedRecords == null || claimedRecords.isEmpty()) {
                return;
            }

            log.info("XAUTOCLAIM 抢占到 {} 条超时消息", claimedRecords.size());

            for (MapRecord<String, Object, Object> record : claimedRecords) {
                processClaimedMessage(record);
            }

        } catch (Exception e) {
            log.error("XAUTOCLAIM 执行失败", e);
        }
    }

    private void processClaimedMessage(MapRecord<String, Object, Object> record) {
        try {
            Map<Object, Object> body = record.getValue();

            Long userDrawId = Long.parseLong(body.get("userDrawId").toString());
            Long userId = Long.parseLong(body.get("userId").toString());
            int retryCount = Integer.parseInt(body.get("retryCount").toString());

            log.info("处理超时消息: userDrawId={}, retryCount={}, recordId={}",
                     userDrawId, retryCount, record.getId());

            // 检查重试次数
            if (retryCount >= RedisStreamConfig.MAX_RETRY_COUNT) {
                handleDeadLetter(record, userDrawId, retryCount);
                return;
            }

            // 更新 retryCount
            Map<String, Object> updatedBody = new HashMap<>(body);
            updatedBody.put("retryCount", retryCount + 1);

            // 重新提交到线程池
            DoUserDrawTask.processFromStream(
                componentBean,
                userDrawId,
                userId,
                record.getId()
            );

        } catch (Exception e) {
            log.error("处理超时消息失败: recordId={}", record.getId(), e);
        }
    }

    private void handleDeadLetter(MapRecord<String, Object, Object> record,
                                   Long userDrawId, int retryCount) {
        log.error("消息重试次数耗尽，进入死信队列: userDrawId={}, retryCount={}, recordId={}",
                  userDrawId, retryCount, record.getId());

        // ACK 消息，避免重复处理
        try {
            redisTemplate.opsForStream().acknowledge(
                RedisStreamConfig.STREAM_USER_DRAW_TASKS,
                RedisStreamConfig.GROUP_TASK_SERVICE,
                record.getId()
            );

            // 发送告警通知
            // TODO: 集成飞书/钉钉告警

        } catch (Exception e) {
            log.error("死信处理失败: recordId={}", record.getId(), e);
        }
    }
}
```

**验证**: `mvn clean compile -pl goto/task -am`

---

### Task 10: 禁用 RabbitMQ 消费者

**Files**: Modify `goto/task/src/main/java/com/freedom/bean/mq/rabbitmq/UserDrawTaskReceiver.java`

**关键修改**:
```java
//@Component  // 注释掉，禁用 RabbitMQ 消费者
public class UserDrawTaskReceiver extends Thread {
    // ...
}
```

**验证**:
1. `mvn clean compile -pl goto/task -am`
2. 启动应用，确认 RabbitMQ 队列不再被消费

---

### Task 11: 添加集成测试

**Files**: Create `goto/task/src/test/java/com/freedom/integration/RedisStreamIntegrationTest.java`

**测试内容**:
1. 测试消息发送和消费
2. 测试超时抢占机制
3. 测试死信处理
4. 测试背压控制
5. 测试幂等性

**验证**: `mvn test -pl goto/task -Dtest=RedisStreamIntegrationTest`

---

### Task 12: 添加监控和告警

**Files**: Create `goto/task/src/main/java/com/freedom/metrics/RedisStreamMetrics.java`

**监控指标**:
1. PEL 长度（超过 100 告警）
2. 消息处理延迟
3. 死信数量
4. 线程池状态

---

### Task 13: 实现灰度发布机制

**Files**:
- Modify: `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`
- Create: `application.yml` 配置项

**配置**:
```yaml
user-draw:
  stream:
    enabled: true  # 启用 Redis Stream
  rabbitmq:
    enabled: false  # 禁用 RabbitMQ
```

---

### Task 14: 编写回滚方案

**Files**: Create `docs/rollback/redis-stream-rollback.md`

**回滚步骤**:
1. 修改配置：`user-draw.stream.enabled=false`, `user-draw.rabbitmq.enabled=true`
2. 取消注释 `UserDrawTaskReceiver` 的 `@Component`
3. 重启应用
4. 验证 RabbitMQ 消费恢复

---

### Task 15: 性能基准测试

**Files**: Create `goto/task/src/test/java/com/freedom/performance/RedisStreamPerformanceTest.java`

**测试内容**:
1. 发送延迟（目标：< 10ms）
2. 消费吞吐量
3. 水平扩展效果
4. 对比 RabbitMQ 方案

---

## 验证清单

完成所有任务后，执行以下验证：

- [ ] 编译成功：`mvn clean compile`
- [ ] 单元测试通过：`mvn test`
- [ ] 集成测试通过
- [ ] 手动功能验证
- [ ] 监控指标正常
- [ ] 性能指标达到预期
- [ ] 回滚方案已测试

---

## 执行建议

**推荐执行方式**: 使用 `superpowers:executing-plans` 技能，按照任务顺序逐个执行，每个任务完成后进行代码审查。

**关键检查点**:
- Task 1 完成后：验证配置类编译成功
- Task 8 完成后：验证基础功能可用
- Task 9 完成后：验证超时处理机制
- Task 13 完成后：开始灰度发布

**预计时间**: 5-7 天开发 + 2-3 天测试 + 1-2 周灰度期
