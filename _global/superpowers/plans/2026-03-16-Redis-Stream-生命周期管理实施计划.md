# Redis Stream 生命周期管理实施计划

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**目标：** 实现企业级 Redis Stream 消息生命周期管理，修复 Redisson 超时配置，实现三重安全检查的消息清理机制

**架构：** 修改 Redisson 客户端超时配置（60秒 → 3秒），创建 StreamLifecycleManager 组件实现定期清理任务，使用三重安全检查（时间截止 + last-delivered-id + min-pending-id）确保不删除未处理消息

**技术栈：** Spring Boot, Redisson, Redis Stream, JUnit 5, Mockito, @Scheduled

---

## 文件结构

### 新建文件
- `goto/task/src/main/java/com/freedom/stream/lifecycle/StreamLifecycleManager.java` - 生命周期管理器
- `goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleManagerTest.java` - 单元测试
- `goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleIntegrationTest.java` - 集成测试

### 修改文件
- `freedom-public/common-utils/src/main/java/com/freedom/config/RedissonConfig.java` - 修改超时配置
- `goto/task/src/main/resources/application.yml` - 添加保留策略配置

---

## Chunk 1: 核心实现

### Task 1: 修改 Redisson 超时配置

**文件：**
- Modify: `freedom-public/common-utils/src/main/java/com/freedom/config/RedissonConfig.java`

- [ ] **Step 1: 读取当前配置**

```bash
cat freedom-public/common-utils/src/main/java/com/freedom/config/RedissonConfig.java | grep -A 10 "setTimeout"
```

预期：找到 `setTimeout(60000)` 配置

- [ ] **Step 2: 修改超时配置**

将 `setTimeout(60000)` 修改为 `setTimeout(3000)`

```java
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
```

- [ ] **Step 3: 验证配置修改**

```bash
cat freedom-public/common-utils/src/main/java/com/freedom/config/RedissonConfig.java | grep "setTimeout"
```

预期：显示 `setTimeout(3000)`

- [ ] **Step 4: Commit**

```bash
git add freedom-public/common-utils/src/main/java/com/freedom/config/RedissonConfig.java
git commit -m "fix: 修复 Redisson 客户端超时配置从 60 秒改为 3 秒"
```

---

### Task 2: 创建 StreamLifecycleManager 基础结构

**文件：**
- Create: `goto/task/src/main/java/com/freedom/stream/lifecycle/StreamLifecycleManager.java`
- Create: `goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleManagerTest.java`

- [ ] **Step 1: 创建包目录**

```bash
mkdir -p goto/task/src/main/java/com/freedom/stream/lifecycle
mkdir -p goto/task/src/test/java/com/freedom/stream/lifecycle
```

预期：目录创建成功

- [ ] **Step 2: 创建 StreamLifecycleManager 类**

```java
package com.freedom.stream.lifecycle;

import lombok.extern.slf4j.Slf4j;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

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
}
```

- [ ] **Step 3: 创建测试类**

```java
package com.freedom.stream.lifecycle;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.redisson.api.RedissonClient;

import static org.assertj.core.api.Assertions.assertThat;

@ExtendWith(MockitoExtension.class)
class StreamLifecycleManagerTest {

    @Mock
    private RedissonClient redissonClient;

    private StreamLifecycleManager manager;

    @BeforeEach
    void setUp() {
        manager = new StreamLifecycleManager(redissonClient);
    }

    @Test
    void testManagerCreation() {
        assertThat(manager).isNotNull();
    }
}
```

- [ ] **Step 4: 运行测试验证基础结构**

```bash
cd goto/task
./mvnw test -Dtest=StreamLifecycleManagerTest
```

预期：测试通过

- [ ] **Step 5: Commit**

```bash
git add goto/task/src/main/java/com/freedom/stream/lifecycle/StreamLifecycleManager.java
git add goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleManagerTest.java
git commit -m "feat: 创建 StreamLifecycleManager 基础结构"
```

---

### Task 3: 实现消息 ID 比较逻辑

**文件：**
- Modify: `goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleManagerTest.java`
- Modify: `goto/task/src/main/java/com/freedom/stream/lifecycle/StreamLifecycleManager.java`

- [ ] **Step 1: 编写 compareMessageId 测试**

在 `StreamLifecycleManagerTest.java` 中添加测试方法：

```java
@Test
void testCompareMessageId_timestampDifferent() {
    // 时间戳不同
    int result = manager.compareMessageId("1000-0", "2000-0");
    assertThat(result).isLessThan(0);
}

@Test
void testCompareMessageId_sequenceDifferent() {
    // 时间戳相同，序列号不同
    int result = manager.compareMessageId("1000-0", "1000-1");
    assertThat(result).isLessThan(0);
}

@Test
void testCompareMessageId_equal() {
    // 完全相同
    int result = manager.compareMessageId("1000-0", "1000-0");
    assertThat(result).isEqualTo(0);
}

@Test
void testCompareMessageId_greater() {
    // 第一个更大
    int result = manager.compareMessageId("2000-0", "1000-0");
    assertThat(result).isGreaterThan(0);
}
```

- [ ] **Step 2: 运行测试验证失败**

```bash
cd goto/task
./mvnw test -Dtest=StreamLifecycleManagerTest#testCompareMessageId_timestampDifferent
```

预期：测试失败，提示 `compareMessageId` 方法不存在

- [ ] **Step 3: 实现 compareMessageId 方法**

在 `StreamLifecycleManager.java` 中添加方法：

```java
/**
 * 比较两个消息 ID
 * 格式：timestamp-sequence
 *
 * @return < 0 if id1 < id2, 0 if equal, > 0 if id1 > id2
 */
public int compareMessageId(String id1, String id2) {
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
```

- [ ] **Step 4: 运行测试验证通过**

```bash
cd goto/task
./mvnw test -Dtest=StreamLifecycleManagerTest
```

预期：所有测试通过

- [ ] **Step 5: Commit**

```bash
git add goto/task/src/main/java/com/freedom/stream/lifecycle/StreamLifecycleManager.java
git add goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleManagerTest.java
git commit -m "feat: 实现消息 ID 比较逻辑"
```

---

### Task 4: 实现安全边界计算逻辑

**文件：**
- Modify: `goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleManagerTest.java`
- Modify: `goto/task/src/main/java/com/freedom/stream/lifecycle/StreamLifecycleManager.java`

- [ ] **Step 1: 编写 calculateSafeMinId 测试**

在 `StreamLifecycleManagerTest.java` 中添加测试方法：

```java
@Test
void testCalculateSafeMinId_normalCase() {
    // 正常情况：选择最小的（最保守的）
    String cutoffId = "1234567890-0";
    String lastDeliveredId = "1234567900-0";
    String minPendingId = "1234567950-0";

    String safeMinId = manager.calculateSafeMinId(cutoffId, lastDeliveredId, minPendingId);

    assertThat(safeMinId).isEqualTo("1234567890-0");
}

@Test
void testCalculateSafeMinId_withPending() {
    // 有 Pending 消息：选择 minPendingId
    String cutoffId = "1000-0";
    String lastDeliveredId = "1200-0";
    String minPendingId = "800-0";

    String safeMinId = manager.calculateSafeMinId(cutoffId, lastDeliveredId, minPendingId);

    assertThat(safeMinId).isEqualTo("800-0");
}

@Test
void testCalculateSafeMinId_nullInput() {
    // 任何一个为 null，返回 null
    String safeMinId = manager.calculateSafeMinId(null, "1000-0", "1000-0");

    assertThat(safeMinId).isNull();
}
```

- [ ] **Step 2: 运行测试验证失败**

```bash
cd goto/task
./mvnw test -Dtest=StreamLifecycleManagerTest#testCalculateSafeMinId_normalCase
```

预期：测试失败，提示 `calculateSafeMinId` 方法不存在

- [ ] **Step 3: 实现 calculateSafeMinId 方法**

在 `StreamLifecycleManager.java` 中添加方法：

```java
/**
 * 计算安全的最小 ID
 * 取三个条件中最小的（最保守的）
 */
public String calculateSafeMinId(String cutoffId, String lastDeliveredId, String minPendingId) {
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
```

- [ ] **Step 4: 运行测试验证通过**

```bash
cd goto/task
./mvnw test -Dtest=StreamLifecycleManagerTest
```

预期：所有测试通过

- [ ] **Step 5: Commit**

```bash
git add goto/task/src/main/java/com/freedom/stream/lifecycle/StreamLifecycleManager.java
git add goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleManagerTest.java
git commit -m "feat: 实现安全边界计算逻辑"
```

---

### Task 5: 实现 last-delivered-id 获取

**文件：**
- Modify: `goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleManagerTest.java`
- Modify: `goto/task/src/main/java/com/freedom/stream/lifecycle/StreamLifecycleManager.java`

- [ ] **Step 1: 编写 getLastDeliveredId 测试**

在 `StreamLifecycleManagerTest.java` 中添加导入和测试方法：

```java
import org.redisson.api.RStream;
import org.redisson.api.StreamMessageId;
import org.redisson.api.stream.StreamGroup;

import java.util.Arrays;
import java.util.Collections;

import static org.mockito.Mockito.when;

// 在类中添加 Mock
@Mock
private RStream<Object, Object> stream;

// 添加测试方法
@Test
void testGetLastDeliveredId_success() {
    // Mock StreamGroup
    StreamGroup group = new StreamGroup();
    group.setName("group:task_service");
    group.setLastDeliveredId(new StreamMessageId(1234567890L, 0L));

    when(stream.listGroups()).thenReturn(Arrays.asList(group));

    String lastDeliveredId = manager.getLastDeliveredId(stream, "group:task_service");

    assertThat(lastDeliveredId).isEqualTo("1234567890-0");
}

@Test
void testGetLastDeliveredId_groupNotFound() {
    // Mock 空列表
    when(stream.listGroups()).thenReturn(Collections.emptyList());

    String lastDeliveredId = manager.getLastDeliveredId(stream, "group:task_service");

    // 应该返回当前时间
    assertThat(lastDeliveredId).matches("\\d+-0");
}

@Test
void testGetLastDeliveredId_exception() {
    // Mock 异常
    when(stream.listGroups()).thenThrow(new RuntimeException("Redis error"));

    String lastDeliveredId = manager.getLastDeliveredId(stream, "group:task_service");

    assertThat(lastDeliveredId).isNull();
}
```

- [ ] **Step 2: 运行测试验证失败**

```bash
cd goto/task
./mvnw test -Dtest=StreamLifecycleManagerTest#testGetLastDeliveredId_success
```

预期：测试失败，提示 `getLastDeliveredId` 方法不存在

- [ ] **Step 3: 实现 getLastDeliveredId 方法**

在 `StreamLifecycleManager.java` 中添加导入和方法：

```java
import org.redisson.api.RStream;
import org.redisson.api.StreamMessageId;
import org.redisson.api.stream.StreamGroup;

import java.util.List;

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
```

- [ ] **Step 4: 运行测试验证通过**

```bash
cd goto/task
./mvnw test -Dtest=StreamLifecycleManagerTest
```

预期：所有测试通过

- [ ] **Step 5: Commit**

```bash
git add goto/task/src/main/java/com/freedom/stream/lifecycle/StreamLifecycleManager.java
git add goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleManagerTest.java
git commit -m "feat: 实现 last-delivered-id 获取逻辑"
```

---

### Task 6: 实现 min-pending-id 获取

**文件：**
- Modify: `goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleManagerTest.java`
- Modify: `goto/task/src/main/java/com/freedom/stream/lifecycle/StreamLifecycleManager.java`

- [ ] **Step 1: 编写 getMinPendingMessageId 测试**

在 `StreamLifecycleManagerTest.java` 中添加导入和测试方法：

```java
import org.redisson.api.PendingEntry;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;

@Test
void testGetMinPendingMessageId_hasPending() {
    // Mock PendingEntry
    PendingEntry entry = new PendingEntry();
    entry.setId(new StreamMessageId(1234567800L, 0L));

    when(stream.listPending(
        eq("group:task_service"),
        eq(StreamMessageId.MIN),
        eq(StreamMessageId.MAX),
        eq(1)
    )).thenReturn(Arrays.asList(entry));

    String minPendingId = manager.getMinPendingMessageId(stream, "group:task_service");

    assertThat(minPendingId).isEqualTo("1234567800-0");
}

@Test
void testGetMinPendingMessageId_noPending() {
    // Mock 空列表
    when(stream.listPending(
        eq("group:task_service"),
        eq(StreamMessageId.MIN),
        eq(StreamMessageId.MAX),
        eq(1)
    )).thenReturn(Collections.emptyList());

    String minPendingId = manager.getMinPendingMessageId(stream, "group:task_service");

    // 应该返回当前时间
    assertThat(minPendingId).matches("\\d+-0");
}

@Test
void testGetMinPendingMessageId_exception() {
    // Mock 异常
    when(stream.listPending(any(), any(), any(), eq(1)))
        .thenThrow(new RuntimeException("Redis error"));

    String minPendingId = manager.getMinPendingMessageId(stream, "group:task_service");

    assertThat(minPendingId).isNull();
}
```

- [ ] **Step 2: 运行测试验证失败**

```bash
cd goto/task
./mvnw test -Dtest=StreamLifecycleManagerTest#testGetMinPendingMessageId_hasPending
```

预期：测试失败，提示 `getMinPendingMessageId` 方法不存在

- [ ] **Step 3: 实现 getMinPendingMessageId 方法**

在 `StreamLifecycleManager.java` 中添加导入和方法：

```java
import org.redisson.api.PendingEntry;

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
```

- [ ] **Step 4: 运行测试验证通过**

```bash
cd goto/task
./mvnw test -Dtest=StreamLifecycleManagerTest
```

预期：所有测试通过

- [ ] **Step 5: Commit**

```bash
git add goto/task/src/main/java/com/freedom/stream/lifecycle/StreamLifecycleManager.java
git add goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleManagerTest.java
git commit -m "feat: 实现 min-pending-id 获取逻辑"
```

---

### Task 7: 实现监控健康检查

**文件：**
- Modify: `goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleManagerTest.java`
- Modify: `goto/task/src/main/java/com/freedom/stream/lifecycle/StreamLifecycleManager.java`

- [ ] **Step 1: 编写 monitorStreamHealth 测试**

在 `StreamLifecycleManagerTest.java` 中添加测试方法：

```java
@Test
void testMonitorStreamHealth_normal() {
    // Mock 正常情况
    when(stream.listPending(
        eq("group:task_service"),
        eq(StreamMessageId.MIN),
        eq(StreamMessageId.MAX),
        eq(1001)
    )).thenReturn(Collections.emptyList());

    when(stream.size()).thenReturn(1000L);

    // 不应该抛出异常
    manager.monitorStreamHealth(stream, "group:task_service");
}

@Test
void testMonitorStreamHealth_highPending() {
    // Mock Pending 过多
    List<PendingEntry> entries = new ArrayList<>();
    for (int i = 0; i < 1001; i++) {
        PendingEntry entry = new PendingEntry();
        entry.setId(new StreamMessageId(1000L + i, 0L));
        entries.add(entry);
    }

    when(stream.listPending(
        eq("group:task_service"),
        eq(StreamMessageId.MIN),
        eq(StreamMessageId.MAX),
        eq(1001)
    )).thenReturn(entries);

    when(stream.size()).thenReturn(1000L);

    // 应该记录警告日志
    manager.monitorStreamHealth(stream, "group:task_service");
}
```

- [ ] **Step 2: 运行测试验证失败**

```bash
cd goto/task
./mvnw test -Dtest=StreamLifecycleManagerTest#testMonitorStreamHealth_normal
```

预期：测试失败，提示 `monitorStreamHealth` 方法不存在

- [ ] **Step 3: 实现 monitorStreamHealth 方法**

在 `StreamLifecycleManager.java` 中添加方法：

```java
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
```

- [ ] **Step 4: 运行测试验证通过**

```bash
cd goto/task
./mvnw test -Dtest=StreamLifecycleManagerTest
```

预期：所有测试通过

- [ ] **Step 5: Commit**

```bash
git add goto/task/src/main/java/com/freedom/stream/lifecycle/StreamLifecycleManager.java
git add goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleManagerTest.java
git commit -m "feat: 实现监控健康检查逻辑"
```

---

### Task 8: 实现定期清理任务

**文件：**
- Modify: `goto/task/src/main/java/com/freedom/stream/lifecycle/StreamLifecycleManager.java`

- [ ] **Step 1: 添加 @Scheduled 导入**

在 `StreamLifecycleManager.java` 中添加导入：

```java
import org.springframework.scheduling.annotation.Scheduled;
import org.redisson.api.stream.StreamTrimArgs;
```

- [ ] **Step 2: 实现 trimExpiredMessages 方法**

在 `StreamLifecycleManager.java` 中添加方法：

```java
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
```

- [ ] **Step 3: 验证代码编译**

```bash
cd goto/task
./mvnw compile
```

预期：编译成功

- [ ] **Step 4: Commit**

```bash
git add goto/task/src/main/java/com/freedom/stream/lifecycle/StreamLifecycleManager.java
git commit -m "feat: 实现定期清理任务"
```

---

### Task 9: 添加配置文件

**文件：**
- Modify: `goto/task/src/main/resources/application.yml`

- [ ] **Step 1: 检查配置文件是否存在**

```bash
ls -la goto/task/src/main/resources/application.yml
```

预期：文件存在

- [ ] **Step 2: 添加 Redis Stream 配置**

在 `application.yml` 中添加配置（如果已有 redis 配置，则在其下添加）：

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

- [ ] **Step 3: 验证配置格式**

```bash
cd goto/task
./mvnw validate
```

预期：验证通过

- [ ] **Step 4: Commit**

```bash
git add goto/task/src/main/resources/application.yml
git commit -m "feat: 添加 Redis Stream 生命周期管理配置"
```

---

## Chunk 2: 测试验证

### Task 10: 集成测试

**文件：**
- Create: `goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleIntegrationTest.java`

- [ ] **Step 1: 创建集成测试类**

```java
package com.freedom.stream.lifecycle;

import org.junit.jupiter.api.Test;
import org.redisson.api.PendingEntry;
import org.redisson.api.RStream;
import org.redisson.api.RedissonClient;
import org.redisson.api.StreamMessageId;
import org.redisson.api.stream.StreamAddArgs;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;

import java.util.List;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

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

        assertThat(messageId).isNotNull();

        // 2. 等待消费（最多 5 秒）
        await().atMost(5, TimeUnit.SECONDS).until(() -> {
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
        long sizeBeforeCleanup = stream.size();
        assertThat(sizeBeforeCleanup).isGreaterThan(0);

        // 5. 等待保留期过期（1 分钟 + 缓冲）
        // 使用 Awaitility 等待而不是 Thread.sleep
        await().atMost(90, TimeUnit.SECONDS)
               .pollInterval(5, TimeUnit.SECONDS)
               .until(() -> {
                   // 检查消息是否已过期（超过 1 分钟）
                   long currentTime = System.currentTimeMillis();
                   return currentTime - messageId.getId0() > 60 * 1000;
               });

        // 6. 手动触发清理
        lifecycleManager.trimExpiredMessages();

        // 7. 验证消息已删除
        long sizeAfter = stream.size();
        assertThat(sizeAfter).isLessThan(sizeBeforeCleanup);
    }
}
```

- [ ] **Step 2: 添加 Awaitility 依赖**

在 `goto/task/pom.xml` 中添加依赖（如果没有）：

```xml
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <version>4.2.0</version>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 3: 运行集成测试**

```bash
cd goto/task
./mvnw test -Dtest=StreamLifecycleIntegrationTest
```

预期：测试通过（需要 Redis 运行），验证：
1. 消息成功发送到 Stream
2. 消息被消费者处理并 ACK
3. 消息在保留期后被清理
4. Pending 消息不被删除

- [ ] **Step 4: Commit**

```bash
git add goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleIntegrationTest.java
git add goto/task/pom.xml
git commit -m "test: 添加 Redis Stream 生命周期集成测试"
```

---

### Task 11: 压力测试

**文件：**
- Modify: `goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleIntegrationTest.java`

- [ ] **Step 1: 添加压力测试方法**

在 `StreamLifecycleIntegrationTest.java` 中添加测试方法：

```java
@Test
void testHighThroughput() throws Exception {
    RStream<Object, Object> stream = redissonClient.getStream("stream:user_draw_tasks");

    // 记录初始 Stream 大小
    long initialSize = stream.size();

    // 发送 10000 条消息
    int messageCount = 10000;
    for (int i = 0; i < messageCount; i++) {
        stream.add(StreamAddArgs.entry("userDrawId", String.valueOf(i))
                                .entry("userId", "1"));
    }

    // 验证消息已添加到 Stream
    long sizeAfterAdd = stream.size();
    assertThat(sizeAfterAdd).isGreaterThanOrEqualTo(initialSize + messageCount);

    // 验证所有消息都被处理（最多 2 分钟）
    await().atMost(2, TimeUnit.MINUTES)
           .pollInterval(5, TimeUnit.SECONDS)
           .until(() -> {
               List<PendingEntry> pending = stream.listPending(
                   "group:task_service",
                   StreamMessageId.MIN,
                   StreamMessageId.MAX,
                   1
               );
               return pending.isEmpty();
           });

    // 验证所有消息都已 ACK（Pending 列表为空）
    List<PendingEntry> finalPending = stream.listPending(
        "group:task_service",
        StreamMessageId.MIN,
        StreamMessageId.MAX,
        10
    );
    assertThat(finalPending).isEmpty();

    // 验证 Stream 中仍有消息（未被清理）
    assertThat(stream.size()).isGreaterThanOrEqualTo(messageCount);
}
```

- [ ] **Step 2: 运行压力测试**

```bash
cd goto/task
./mvnw test -Dtest=StreamLifecycleIntegrationTest#testHighThroughput
```

预期：测试通过，所有消息都被处理

- [ ] **Step 3: Commit**

```bash
git add goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleIntegrationTest.java
git commit -m "test: 添加 Redis Stream 压力测试"
```

---

### Task 12: 边界测试

**文件：**
- Modify: `goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleManagerTest.java`

- [ ] **Step 1: 添加边界测试方法**

在 `StreamLifecycleManagerTest.java` 中添加导入和测试方法：

```java
import static org.junit.jupiter.api.Assertions.assertThrows;

@Test
void testTrimWithPendingMessages() {
    // 场景：有 Pending 消息时执行清理
    // Mock 有 Pending 消息
    PendingEntry entry = new PendingEntry();
    entry.setId(new StreamMessageId(System.currentTimeMillis() - 60000, 0L));

    when(stream.listPending(
        eq("group:task_service"),
        eq(StreamMessageId.MIN),
        eq(StreamMessageId.MAX),
        eq(1)
    )).thenReturn(Arrays.asList(entry));

    when(stream.listGroups()).thenReturn(Collections.emptyList());

    // 应该不删除 Pending 消息
    String minPendingId = manager.getMinPendingMessageId(stream, "group:task_service");
    assertThat(minPendingId).isNotNull();
}

@Test
void testTrimWithNoMessages() {
    // 场景：Stream 为空时执行清理
    when(stream.size()).thenReturn(0L);
    when(stream.listGroups()).thenReturn(Collections.emptyList());
    when(stream.listPending(any(), any(), any(), anyInt()))
        .thenReturn(Collections.emptyList());

    // 应该不报错，正常返回
    manager.monitorStreamHealth(stream, "group:task_service");
}

@Test
void testTrimWithInvalidGroup() {
    // 场景：消费者组不存在
    when(stream.listGroups()).thenReturn(Collections.emptyList());

    String lastDeliveredId = manager.getLastDeliveredId(stream, "invalid_group");

    // 应该返回当前时间，记录警告
    assertThat(lastDeliveredId).matches("\\d+-0");
}

@Test
void testCompareMessageId_invalidFormat() {
    // 场景：消息 ID 格式错误
    assertThrows(NumberFormatException.class, () -> {
        manager.compareMessageId("invalid", "1000-0");
    });
}
```

- [ ] **Step 2: 运行边界测试**

```bash
cd goto/task
./mvnw test -Dtest=StreamLifecycleManagerTest
```

预期：所有测试通过

- [ ] **Step 3: Commit**

```bash
git add goto/task/src/test/java/com/freedom/stream/lifecycle/StreamLifecycleManagerTest.java
git commit -m "test: 添加 Redis Stream 边界测试"
```

---

## 验证和部署

### Task 13: 运行完整测试套件

**文件：**
- 无

- [ ] **Step 1: 运行所有单元测试**

```bash
cd goto/task
./mvnw test
```

预期：所有测试通过

- [ ] **Step 2: 检查测试覆盖率**

```bash
cd goto/task
./mvnw jacoco:report
```

预期：覆盖率 > 80%

- [ ] **Step 3: 验证代码编译**

```bash
cd goto/task
./mvnw clean package
```

预期：编译成功，生成 JAR 文件

---

### Task 14: 部署到测试环境

**文件：**
- 无

- [ ] **Step 1: 确认配置**

验证 `application.yml` 中的配置：
- `redis.stream.retention.minutes: 5` （测试环境 5 分钟）
- `redis.stream.monitoring.enabled: true`

- [ ] **Step 2: 启动应用**

```bash
cd goto/task
./mvnw spring-boot:run
```

预期：应用启动成功，日志显示 StreamLifecycleManager 初始化

- [ ] **Step 3: 验证定期清理任务**

观察日志，等待 5 分钟后应该看到：
```
🧹 开始清理过期消息 - stream: stream:user_draw_tasks, 保留期: 5 分钟
✅ 清理完成 - 删除 X 条消息，安全边界: XXXXX-0
```

- [ ] **Step 4: 手动发送测试消息**

使用 Redis CLI 或测试工具发送消息，验证：
1. 消息被正常消费
2. 消费者线程不再阻塞
3. 5 分钟后消息被自动清理

---

### Task 15: 生产环境准备

**文件：**
- Modify: `goto/task/src/main/resources/application-prod.yml`

- [ ] **Step 1: 创建生产环境配置**

创建或修改 `application-prod.yml`：

```yaml
redis:
  stream:
    # 消息保留策略
    retention:
      minutes: 1440       # 生产环境：1 天
      max-length: 100000  # 最大消息数量

    # 监控配置
    monitoring:
      enabled: true
      pending-alert-threshold: 1000  # Pending 消息告警阈值
```

- [ ] **Step 2: 创建运维文档**

创建 `docs/operations/redis-stream-lifecycle.md` 文档：

```markdown
# Redis Stream 生命周期管理运维指南

## 配置参数说明

| 参数 | 说明 | 测试环境 | 生产环境 |
|------|------|----------|----------|
| `redis.stream.retention.minutes` | 消息保留时间（分钟） | 5 | 1440 (1天) |
| `redis.stream.retention.max-length` | Stream 最大长度 | 100000 | 100000 |
| `redis.stream.monitoring.enabled` | 是否启用监控 | true | true |
| `redis.stream.monitoring.pending-alert-threshold` | Pending 告警阈值 | 1000 | 1000 |

## 监控指标

- `stream.length`: Stream 总消息数（告警阈值：> 100,000）
- `stream.pending.count`: Pending 消息数（告警阈值：> 1000）
- `stream.trim.count`: 每次清理删除数
- `stream.trim.duration`: 清理任务耗时（告警阈值：> 5秒）

## 故障排查

### 问题 1：Pending 消息过多
- 检查消费者是否正常运行
- 检查消息处理是否有异常
- 查看日志中的错误信息

### 问题 2：清理任务失败
- 检查 Redis 连接是否正常
- 检查消费者组是否存在
- 查看详细错误日志
```

- [ ] **Step 3: 验证文档创建**

```bash
ls -la docs/operations/redis-stream-lifecycle.md
```

预期：文件存在且内容完整

- [ ] **Step 4: Commit**

```bash
git add goto/task/src/main/resources/application-prod.yml
git add docs/operations/redis-stream-lifecycle.md
git commit -m "feat: 添加生产环境配置和运维文档"
```

---

## 完成检查清单

- [ ] Redisson 超时配置已修改（60秒 → 3秒）
- [ ] StreamLifecycleManager 已实现
- [ ] 三重安全检查逻辑已实现
- [ ] 定期清理任务已配置（每 5 分钟）
- [ ] 监控和告警已实现
- [ ] 单元测试已编写并通过
- [ ] 集成测试已编写并通过
- [ ] 压力测试已编写并通过
- [ ] 边界测试已编写并通过
- [ ] 测试覆盖率 > 80%
- [ ] 测试环境验证通过
- [ ] 生产环境配置已准备
- [ ] 文档已更新

---

## 参考资料

- 设计文档：`docs/superpowers/specs/2026-03-16-Redis-Stream-生命周期管理设计.md`
- Redis Stream 官方文档：https://redis.io/docs/data-types/streams/
- Redisson Stream API：https://github.com/redisson/redisson/wiki/7.-distributed-collections#712-stream
- Spring Boot Scheduling：https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#scheduling

---

**计划创建日期：** 2026-03-16
**预计实施时间：** 2-3 天
**风险等级：** 低（有完整测试覆盖）

