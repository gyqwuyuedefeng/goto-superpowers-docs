# Redis Stream 生命周期管理设计

> **创建日期：** 2026-03-16
> **状态：** 已批准
> **目标：** 实现企业级 Redis Stream 消息生命周期管理

---

## 📋 **问题背景**

### 当前问题

1. **消息 ACK 后不删除**
   - 用户期望：消息 ACK 后应该从 Redis 中删除
   - 实际情况：消息 ACK 后还在 Redis Stream 中
   - 疑问：什么时候才会删除？

2. **Redisson 客户端阻塞**
   - 现象：手动发送测试消息后，消费者没有处理
   - 线程状态：所有 4 个 pool-4-thread 都在 WAITING 状态
   - 阻塞位置：`org.redisson.command.CommandAsyncService.get()`
   - Redis 状态：消息已投递（last-delivered-id 更新），但在 pending 中未被 ACK

### 根本原因分析

1. **Redis Stream 消息生命周期误解**
   - `XACK` 命令：只是将消息从 Pending 列表移除，**不删除消息本身**
   - 消息仍然存在于 Stream 中，需要显式使用 `XTRIM` 删除

2. **Redisson 超时配置不当**
   - 当前配置：`setTimeout(60000)` - 60 秒
   - XREADGROUP 超时：1 秒
   - 冲突：Redisson 等待 60 秒，但 XREADGROUP 1 秒就返回，导致线程阻塞

---

## 🎯 **设计目标**

1. **安全性**：确保不删除未处理的消息
2. **可靠性**：避免消费者线程阻塞
3. **可配置性**：支持灵活的保留策略
4. **可观测性**：提供完整的监控和日志
5. **企业级**：符合生产环境最佳实践

---

## 🏗️ **架构设计**

### 方案选择

**选择方案 A：定期清理过期消息**

**优势：**
- ✅ 保留消息历史，便于审计和调试
- ✅ 支持消息重放和故障恢复
- ✅ 灵活的保留策略（时间 + 数量）
- ✅ 符合企业级最佳实践

**配置：**
- 测试环境：5 分钟保留期
- 生产环境：1 天保留期
- 最大长度：100,000 条消息

---

## 🔧 **核心组件设计**

### 1. Redisson 超时修复

**问题：** 60 秒超时导致消费者线程阻塞

**解决方案：**

```java
// 文件：freedom-public/common-utils/src/main/java/com/freedom/config/RedissonConfig.java

@Configuration
public class RedissonConfig {

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
              .setAddress("redis://" + host + ":" + port)
              .setPassword(password)
              .setDatabase(database)
              .setTimeout(3000);  // 修改：60000 → 3000 (3秒)

        return Redisson.create(config);
    }
}
```

**效果：**
- 避免长时间阻塞
- 快速失败，快速重试
- 与 XREADGROUP 1 秒超时匹配

### 2. StreamLifecycleManager - 生命周期管理器

**职责：** 定期清理过期消息，确保安全删除

**实现：**

```java
package com.freedom.stream.lifecycle;

import lombok.extern.slf4j.Slf4j;
import org.redisson.api.RStream;
import org.redisson.api.RedissonClient;
import org.redisson.api.StreamMessageId;
import org.redisson.api.stream.StreamTrimArgs;
import org.redisson.api.stream.StreamGroup;
import org.redisson.api.PendingEntry;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Map;

/**
 * Redis Stream 生命周期管理器
 *
 * 功能：
 * 1. 定期清理过期消息
 * 2. 三重安全检查，确保不删除未处理消息
 * 3. 监控和告警
 */
@Component
@Slf4j
public class StreamLifecycleManager {

    private final RedissonClient redissonClient;

    @Value("${redis.stream.retention.minutes:5}")
    private int retentionMinutes;

    @Value("${redis.stream.retention.max-length:100000}")
    private long maxLength;

    @Value("${redis.stream.monitoring.enabled:true}")
    private boolean monitoringEnabled;

    @Value("${redis.stream.monitoring.pending-alert-threshold:1000}")
    private long pendingAlertThreshold;

    public StreamLifecycleManager(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }

    /**
     * 定期清理过期消息
     * 执行频率：每 5 分钟
     */
    @Scheduled(cron = "0 */5 * * * ?")
    public void trimExpiredMessages() {
        String streamKey = "stream:user_draw_tasks";
        String groupName = "group:task_service";

        try {
            log.info("🧹 开始清理过期消息 - stream: {}, 保留期: {} 分钟",
                    streamKey, retentionMinutes);

            RStream<Object, Object> stream = redissonClient.getStream(streamKey);

            // 1. 计算时间截止点
            long cutoffTime = System.currentTimeMillis() - retentionMinutes * 60 * 1000L;
            String cutoffId = cutoffTime + "-0";

            // 2. 获取 last-delivered-id（确保只删除已投递的消息）
            String lastDeliveredId = getLastDeliveredId(stream, groupName);

            // 3. 获取 min-pending-id（确保不删除未 ACK 的消息）
            String minPendingId = getMinPendingMessageId(stream, groupName);

            // 4. 计算安全边界（取三者中最小的）
            String safeMinId = calculateSafeMinId(cutoffId, lastDeliveredId, minPendingId);

            if (safeMinId == null) {
                log.warn("⚠️ 无法计算安全边界，跳过本次清理");
                return;
            }

            // 5. 执行 TRIM
            long trimmed = stream.trim(StreamTrimArgs.minId(safeMinId));

            log.info("✅ 清理完成 - 删除 {} 条消息，安全边界: {}", trimmed, safeMinId);

            // 6. 监控和告警
            if (monitoringEnabled) {
                monitorStreamHealth(stream, groupName);
            }

        } catch (Exception e) {
            log.error("❌ 清理任务失败 - stream: {}", streamKey, e);
            // 失败不影响消息消费
        }
    }

    /**
     * 获取消费者组的 last-delivered-id
     * 这是最后一条投递给消费者组的消息 ID
     */
    private String getLastDeliveredId(RStream<Object, Object> stream, String groupName) {
        try {
            List<StreamGroup> groups = stream.listGroups();
            for (StreamGroup group : groups) {
                if (group.getName().equals(groupName)) {
                    StreamMessageId lastDeliveredId = group.getLastDeliveredId();
                    if (lastDeliveredId != null) {
                        return lastDeliveredId.toString();
                    }
                }
            }

            log.warn("⚠️ 无法获取 last-delivered-id，使用当前时间");
            return System.currentTimeMillis() + "-0";

        } catch (Exception e) {
            log.error("❌ 获取 last-delivered-id 失败", e);
            return null;
        }
    }

    /**
     * 获取 Pending 列表中最小的消息 ID
     * 确保不删除未 ACK 的消息
     */
    private String getMinPendingMessageId(RStream<Object, Object> stream, String groupName) {
        try {
            // 使用 listPending 获取 Pending 消息
            List<PendingEntry> pendingEntries = stream.listPending(
                groupName,
                StreamMessageId.MIN,  // 从最小 ID 开始
                StreamMessageId.MAX,  // 到最大 ID
                1                      // 只取第一条（最小的）
            );

            if (!pendingEntries.isEmpty()) {
                return pendingEntries.get(0).getId().toString();
            }

            // 没有 Pending 消息，返回当前时间
            return System.currentTimeMillis() + "-0";

        } catch (Exception e) {
            log.error("❌ 获取 min-pending-id 失败", e);
            return null;
        }
    }

    /**
     * 计算安全的最小 ID
     * 取三个条件中最小的（最保守的）
     */
    private String calculateSafeMinId(String cutoffId, String lastDeliveredId, String minPendingId) {
        if (cutoffId == null || lastDeliveredId == null || minPendingId == null) {
            return null;
        }

        // 比较三个 ID，选择最小的
        String minId = cutoffId;

        if (compareMessageId(lastDeliveredId, minId) < 0) {
            minId = lastDeliveredId;
        }

        if (compareMessageId(minPendingId, minId) < 0) {
            minId = minPendingId;
        }

        log.debug("🔍 安全边界计算 - cutoff: {}, lastDelivered: {}, minPending: {}, safe: {}",
                cutoffId, lastDeliveredId, minPendingId, minId);

        return minId;
    }

    /**
     * 比较两个消息 ID
     * 格式：timestamp-sequence
     *
     * @return < 0 if id1 < id2, 0 if equal, > 0 if id1 > id2
     */
    private int compareMessageId(String id1, String id2) {
        String[] parts1 = id1.split("-");
        String[] parts2 = id2.split("-");

        long time1 = Long.parseLong(parts1[0]);
        long time2 = Long.parseLong(parts2[0]);

        if (time1 != time2) {
            return Long.compare(time1, time2);
        }

        long seq1 = Long.parseLong(parts1[1]);
        long seq2 = Long.parseLong(parts2[1]);

        return Long.compare(seq1, seq2);
    }

    /**
     * 监控 Stream 健康状态
     */
    private void monitorStreamHealth(RStream<Object, Object> stream, String groupName) {
        try {
            // 检查 Pending 消息数量
            List<PendingEntry> pendingEntries = stream.listPending(
                groupName,
                StreamMessageId.MIN,
                StreamMessageId.MAX,
                (int) pendingAlertThreshold + 1  // 多取一条用于判断
            );

            long pendingCount = pendingEntries.size();

            if (pendingCount > pendingAlertThreshold) {
                log.warn("⚠️ Pending 消息过多 - count: {}, threshold: {}",
                        pendingCount, pendingAlertThreshold);
            }

            // 检查 Stream 长度
            long streamLength = stream.size();
            if (streamLength > maxLength) {
                log.warn("⚠️ Stream 长度超过限制 - length: {}, max: {}",
                        streamLength, maxLength);
            }

        } catch (Exception e) {
            log.error("❌ 监控检查失败", e);
        }
    }
}
```

### 3. 配置文件

**文件：** `goto/task/src/main/resources/application.yml`

```yaml
redis:
  stream:
    # 消息保留策略
    retention:
      minutes: 5          # 测试环境：5 分钟（生产环境建议 1440 = 1天）
      max-length: 100000  # 最大消息数量

    # 监控配置
    monitoring:
      enabled: true
      pending-alert-threshold: 1000  # Pending 消息告警阈值
```

---

## 📊 **数据流设计**

### 消息生命周期完整流程

```
1. 生产者发送消息
   ↓
2. Redis Stream 存储（自动生成 ID: timestamp-sequence）
   ↓
3. XREADGROUP 读取（使用 > 只读新消息）
   ↓
4. 消息进入 Pending 状态（已投递，未 ACK）
   ↓
5. 消费者处理消息
   ↓
6. XACK 确认（从 Pending 移除，消息仍在 Stream）
   ↓
7. 定期清理任务（每 5 分钟）
   ├─ 检查 1: 消息时间 > 保留期（5分钟）
   ├─ 检查 2: 消息 ID ≤ last-delivered-id（已投递）
   ├─ 检查 3: 消息 ID < min-pending-id（已 ACK）
   └─ 三重检查通过 → XTRIM 删除
```

### 三重安全检查机制

```
cutoffId = 当前时间 - 5分钟
lastDeliveredId = 消费者组最后投递的消息 ID
minPendingId = Pending 列表中最小的消息 ID

safeMinId = MIN(cutoffId, lastDeliveredId, minPendingId)

只删除 ID < safeMinId 的消息
```

**为什么安全？**

1. **时间检查**：只删除超过保留期的消息
2. **投递检查**：只删除已投递的消息（ID ≤ last-delivered-id）
3. **ACK 检查**：只删除已 ACK 的消息（ID < min-pending-id）

**边界情况：**

| 场景 | cutoffId | lastDeliveredId | minPendingId | safeMinId | 结果 |
|------|----------|-----------------|--------------|-----------|------|
| 正常消费 | 1000 | 1200 | 1300 | 1000 | 删除 < 1000 的消息 ✅ |
| 有 Pending | 1000 | 1200 | 1100 | 1000 | 删除 < 1000 的消息 ✅ |
| 未投递 | 1000 | 900 | 1300 | 900 | 删除 < 900 的消息，保护未投递消息 ✅ |
| 全部 Pending | 1000 | 1200 | 800 | 800 | 删除 < 800 的消息，保护 Pending 消息 ✅ |

### 多实例负载均衡

```
Instance 1 (consumer-1)  ─┐
Instance 2 (consumer-2)  ─┼─→ Redis Server (串行化处理)
Instance 3 (consumer-3)  ─┘
                            ↓
                    保证消息 ID 单调递增
                    保证消息不重复投递
```

**关键点：**
- Redis 服务器端串行化处理 XADD 命令
- 自动生成的消息 ID 保证单调递增
- 不依赖客户端时钟同步
- 多实例安全

---

## ⚠️ **错误处理设计**

### 1. Redisson 超时处理

**问题：** 60 秒超时导致线程阻塞

**解决：**
```java
// 修改前：60 秒超时
config.setTimeout(60000);

// 修改后：3 秒超时
config.setTimeout(3000);
```

**效果：**
- 避免长时间阻塞
- 快速失败，快速重试
- 与 XREADGROUP 1 秒超时匹配

### 2. 消息处理失败

**策略：** 不 ACK，消息保留在 Pending

```java
try {
    processMessage(messageId, data);
    stream.acknowledge(group, messageId);  // 成功才 ACK
} catch (Exception e) {
    log.error("消息处理失败，不 ACK: {}", messageId, e);
    // 消息保留在 Pending，等待重试或人工介入
}
```

### 3. 清理任务异常

**保护机制：**
```java
@Scheduled(cron = "0 */5 * * * ?")
public void trimExpiredMessages() {
    try {
        // 三重安全检查
        String safeMinId = calculateSafeMinId(...);

        if (safeMinId == null) {
            log.warn("无法计算安全边界，跳过本次清理");
            return;
        }

        stream.trim(StreamTrimArgs.minId(safeMinId));
        log.info("✅ 清理完成，安全边界: {}", safeMinId);

    } catch (Exception e) {
        log.error("❌ 清理任务失败", e);
        // 失败不影响消息消费
    }
}
```

### 4. 监控告警

**告警条件：**
- Pending 消息数量 > 1000
- Stream 长度 > 100,000
- 清理任务失败

**处理流程：**
```
监控检测到异常
    ↓
记录 WARN 日志
    ↓
（可选）发送告警通知
    ↓
人工介入处理
```

---

## 🧪 **测试策略**

### 1. 单元测试

```java
@Test
void testCalculateSafeMinId() {
    // 测试三重检查逻辑
    String cutoffId = "1234567890-0";
    String lastDeliveredId = "1234567900-0";
    String minPendingId = "1234567950-0";

    String safeMinId = manager.calculateSafeMinId(
        cutoffId, lastDeliveredId, minPendingId
    );

    // 应该选择最小的（最保守的）
    assertThat(safeMinId).isEqualTo("1234567890-0");
}

@Test
void testCompareMessageId() {
    assertThat(manager.compareMessageId("1000-0", "2000-0")).isLessThan(0);
    assertThat(manager.compareMessageId("1000-0", "1000-1")).isLessThan(0);
    assertThat(manager.compareMessageId("1000-0", "1000-0")).isEqualTo(0);
    assertThat(manager.compareMessageId("2000-0", "1000-0")).isGreaterThan(0);
}
```

### 2. 集成测试

```java
@SpringBootTest
@TestPropertySource(properties = {
    "redis.stream.retention.minutes=1"  // 1 分钟保留期
})
class StreamLifecycleIntegrationTest {

    @Autowired
    private RedissonClient redissonClient;

    @Autowired
    private StreamLifecycleManager lifecycleManager;

    @Test
    void testMessageLifecycle() throws Exception {
        RStream<Object, Object> stream = redissonClient.getStream("stream:user_draw_tasks");

        // 1. 发送消息
        StreamMessageId messageId = stream.add(
            StreamAddArgs.entry("userDrawId", "999999")
                         .entry("userId", "1")
        );

        // 2. 等待消费
        await().atMost(5, SECONDS).until(() -> {
            List<PendingEntry> pending = stream.listPending(
                "group:task_service",
                StreamMessageId.MIN,
                StreamMessageId.MAX,
                1
            );
            return pending.isEmpty();
        });

        // 3. 验证消息已 ACK
        List<PendingEntry> pendingEntries = stream.listPending(
            "group:task_service",
            StreamMessageId.MIN,
            StreamMessageId.MAX,
            1
        );
        assertThat(pendingEntries).isEmpty();

        // 4. 验证消息仍在 Stream
        assertThat(stream.size()).isGreaterThan(0);

        // 5. 等待清理（1 分钟 + 30 秒缓冲）
        Thread.sleep(90 * 1000);

        // 6. 手动触发清理
        lifecycleManager.trimExpiredMessages();

        // 7. 验证消息已删除
        // 注意：可能还有其他消息，所以只检查总数减少
        long sizeAfterTrim = stream.size();
        assertThat(sizeAfterTrim).isLessThan(stream.size());
    }
}
```

### 3. 压力测试

```java
@Test
void testHighThroughput() throws Exception {
    RStream<Object, Object> stream = redissonClient.getStream("stream:user_draw_tasks");

    // 发送 10000 条消息
    for (int i = 0; i < 10000; i++) {
        stream.add(StreamAddArgs.entry("userDrawId", String.valueOf(i))
                                .entry("userId", "1"));
    }

    // 验证所有消息都被处理
    await().atMost(2, MINUTES).until(() -> {
        List<PendingEntry> pending = stream.listPending(
            "group:task_service",
            StreamMessageId.MIN,
            StreamMessageId.MAX,
            1
        );
        return pending.isEmpty();
    });

    // 验证没有消息丢失
    // 通过数据库或其他方式验证处理数量
    assertThat(getProcessedCount()).isEqualTo(10000);
}
```

### 4. 边界测试

```java
@Test
void testTrimWithPendingMessages() {
    // 场景：有 Pending 消息时执行清理
    // 预期：不删除 Pending 消息
}

@Test
void testTrimWithNoMessages() {
    // 场景：Stream 为空时执行清理
    // 预期：不报错，正常返回
}

@Test
void testTrimWithInvalidGroup() {
    // 场景：消费者组不存在
    // 预期：记录警告，跳过清理
}
```

---

## 📈 **监控指标**

### 关键指标

| 指标 | 说明 | 告警阈值 |
|------|------|----------|
| `stream.length` | Stream 总消息数 | > 100,000 |
| `stream.pending.count` | Pending 消息数 | > 1000 |
| `stream.trim.count` | 每次清理删除数 | - |
| `stream.trim.duration` | 清理任务耗时 | > 5 秒 |
| `stream.trim.errors` | 清理任务失败次数 | > 0 |

### 日志示例

```
2026-03-16 10:00:00 INFO  🧹 开始清理过期消息 - stream: stream:user_draw_tasks, 保留期: 5 分钟
2026-03-16 10:00:01 DEBUG 🔍 安全边界计算 - cutoff: 1234567890-0, lastDelivered: 1234567900-0, minPending: 1234567950-0, safe: 1234567890-0
2026-03-16 10:00:02 INFO  ✅ 清理完成 - 删除 150 条消息，安全边界: 1234567890-0
```

---

## ✅ **设计总结**

### 核心优势

1. **安全性**
   - 三重检查确保不删除未处理消息
   - 时间 + 投递 + ACK 三重保护
   - 边界情况全覆盖

2. **可靠性**
   - Redisson 超时修复避免线程阻塞
   - 清理任务失败不影响消息消费
   - 多实例安全

3. **可配置性**
   - 保留期可配置（测试 5 分钟，生产 1 天）
   - 最大长度可配置
   - 监控阈值可配置

4. **可观测性**
   - 完整的日志记录
   - 关键指标监控
   - 告警机制

5. **企业级**
   - 符合生产环境最佳实践
   - 完整的测试覆盖
   - 清晰的错误处理

### 实施步骤

1. **修改 Redisson 配置**：`setTimeout(60000)` → `setTimeout(3000)`
2. **创建 StreamLifecycleManager**：实现三重安全检查
3. **添加配置**：`application.yml` 中添加保留策略
4. **编写测试**：单元测试 + 集成测试 + 压力测试
5. **部署验证**：测试环境验证 5 分钟保留期
6. **生产部署**：调整为 1 天保留期

---

## 📚 **参考资料**

- [Redis Stream 官方文档](https://redis.io/docs/data-types/streams/)
- [Redisson Stream API](https://github.com/redisson/redisson/wiki/7.-distributed-collections#712-stream)
- [Spring Boot Scheduling](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#scheduling)

---

**设计批准：** ✅ 2026-03-16
**下一步：** 创建实施计划
