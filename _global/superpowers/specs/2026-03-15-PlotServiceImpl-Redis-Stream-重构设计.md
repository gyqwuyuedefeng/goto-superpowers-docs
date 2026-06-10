# PlotServiceImpl Redis Stream 重构设计

> **创建日期**: 2026-03-15
> **作者**: Claude Sonnet 4.5
> **状态**: 待审查

## 1. 概述

### 1.1 重构目标

将 `PlotServiceImpl` 中的用户绘图任务消息发送从 Redisson 原生 API 迁移到 `@StreamProducer` 注解方式，并移除 RabbitMQ 相关代码，实现完全基于 Redis Stream 的消息队列架构。

### 1.2 重构范围

**包含**：
- ✅ `PlotServiceImpl.sendToRedisStream()` 方法重构
- ✅ `PlotServiceImpl.customDrawPlot()` 方法简化
- ✅ `UserDrawStreamConsumer` 配置更新
- ✅ 移除 RabbitMQ 相关代码和 MqMessage 数据库操作

**不包含**：
- ❌ 其他业务的 RabbitMQ 消费者（保持不变）
- ❌ RabbitMQ 基础设施配置（暂时保留）

### 1.3 重构策略

采用**清理冗余方案**：彻底移除用户绘图任务的 RabbitMQ 代码，完全切换到 Redis Stream，代码更简洁，减少维护负担。

---

## 2. 架构设计

### 2.1 架构对比

**重构前**：
```
PlotServiceImpl.customDrawPlot()
├─ if (useRedisStream())
│   └─ sendToRedisStream() [Redisson 原生 API]
│       ├─ 手动构建 Map 消息
│       ├─ 手动序列化
│       └─ 调用 RStream.add()
├─ else if (useRabbitMQ())
│   ├─ 创建 MqMessage 对象
│   ├─ 保存到数据库
│   ├─ 发送到 RabbitMQ
│   └─ 更新数据库状态
└─ else throw Exception
```

**重构后**：
```
PlotServiceImpl.customDrawPlot()
└─ sendToRedisStream() [StreamTemplate.send()]
    ├─ 创建 UserDrawStreamMessage 对象
    ├─ 调用 userDrawProducer.send()
    └─ 自动序列化 + 发送
```

### 2.2 消息流转

```
┌──────────────┐
│ PlotService  │
│ Impl         │
└──────┬───────┘
       │ @StreamProducer
       │ userDrawProducer.send()
       ↓
┌──────────────────────────────┐
│ Redis Stream                 │
│ stream:user_draw_tasks       │
└──────┬───────────────────────┘
       │ @StreamListener
       │ concurrency = 4
       │ maxRetries = 3
       ↓
┌──────────────────────────────┐
│ UserDrawStreamConsumer       │
│ handleUserDrawMessage()      │
└──────────────────────────────┘
```

---

## 3. 详细设计

### 3.1 生产者重构

#### 3.1.1 添加 @StreamProducer 字段注入

**文件**: `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class PlotServiceImpl implements PlotService {

    private final ComponentBean componentBean;
    private final UserDrawTaskProperties userDrawTaskProperties;

    /**
     * Redis Stream 生产者
     * 使用 @StreamProducer 注解自动注入
     */
    @StreamProducer(
        stream = RedisStreamConfig.STREAM_USER_DRAW_TASKS,
        serialization = SerializationType.JSON
    )
    private StreamTemplate<UserDrawStreamMessage> userDrawProducer;

    // ... 其他代码
}
```

**设计要点**：
- 使用 `@StreamProducer` 注解，由 Spring 自动注入
- 显式指定 `serialization = SerializationType.JSON`，确保序列化方式明确
- 泛型类型为 `UserDrawStreamMessage`，提供类型安全

#### 3.1.2 重构 sendToRedisStream() 方法

**变更前**：
```java
private void sendToRedisStream(Long userDrawId, Long userId) {
    try {
        // 使用 Redisson 原生 API
        RStream<Object, Object> stream = componentBean.getRedissonClient()
                .getStream(RedisStreamConfig.STREAM_USER_DRAW_TASKS);

        // 手动构建 Map 消息
        Map<Object, Object> messageBody = new HashMap<>();
        messageBody.put("userDrawId", userDrawId);
        messageBody.put("userId", userId);
        messageBody.put("retryCount", 0);
        messageBody.put("timestamp", System.currentTimeMillis());

        stream.add(StreamAddArgs.entries(messageBody));

        log.info("消息发送到 Redis Stream 成功: userDrawId={}, userId={}",
                userDrawId, userId);
    } catch (Exception e) {
        log.error("消息发送到 Redis Stream 失败: userDrawId={}, userId={}",
                userDrawId, userId, e);
        throw new RuntimeException("消息发送失败", e);
    }
}
```

**变更后**：
```java
/**
 * 发送消息到 Redis Stream
 *
 * @param userDrawId 用户绘图任务 ID
 * @param userId 用户 ID
 */
private void sendToRedisStream(Long userDrawId, Long userId) {
    try {
        // 创建类型安全的消息对象
        UserDrawStreamMessage message = new UserDrawStreamMessage(userDrawId, userId);

        // 使用 StreamTemplate 发送，自动序列化
        String messageId = userDrawProducer.send(message);

        log.info("消息发送到 Redis Stream 成功: userDrawId={}, userId={}, messageId={}",
                userDrawId, userId, messageId);
    } catch (Exception e) {
        log.error("消息发送到 Redis Stream 失败: userDrawId={}, userId={}",
                userDrawId, userId, e);
        throw new RuntimeException("消息发送失败", e);
    }
}
```

**改进点**：
1. **类型安全**：使用 `UserDrawStreamMessage` 对象，编译时检查
2. **代码简洁**：从 15 行减少到 5 行
3. **自动序列化**：由 `StreamTemplate` 自动处理
4. **返回 messageId**：便于追踪和调试

#### 3.1.3 简化 customDrawPlot() 方法

**移除内容**：

1. **移除灰度切换逻辑**：
```java
// 删除
if (userDrawTaskProperties.useRedisStream()) {
    // ...
} else if (userDrawTaskProperties.useRabbitMQ()) {
    // ... 大量 RabbitMQ 代码
} else {
    throw new RuntimeException("未配置任何消息队列");
}
```

2. **移除 correlationId 生成**：
```java
// 删除
String correlationId = RabbitPublishType.taskSubUser(
    componentBean.getSnowflakeDistributeIdUtils(),
    plotType.getUserDraw().getId()
);
```

3. **移除未完成消息查询**：
```java
// 删除
String correlationIdPrefix = RabbitPublishType.taskSubUserPrefix(
    plotType.getUserDraw().getId()
);
List<MqMessage> unfinishedMqMessageList = componentBean.getRepository()
    .getMqMessageRepository()
    .findByUnfinishedByMessageIdPrefix(correlationIdPrefix, MqMessageEnum.COMPLETED.name());

// 删除超时检查逻辑
if (CollectionUtils.isNotEmpty(unfinishedMqMessageList)) {
    // ... 复杂的超时判断和清理逻辑
}
```

4. **移除 MqMessage 数据库操作**：
```java
// 删除
MqMessage mqMessage = new MqMessage();
mqMessage.setUserId(SecurityUtils.getCurrentUserId());
mqMessage.setMessageId(correlationId);
// ... 设置各种字段
componentBean.getRepository().getMqMessageRepository().save(mqMessage);

// 删除
componentBean.getRepository().getMqMessageRepository()
    .updateStatusByMessageId(correlationId, MqMessageEnum.SENT.name());
```

**简化后的核心逻辑**：
```java
RHandlerWrapper<Object> rHandlerWrapper = new RHandlerWrapper<>(() -> {
    // 直接发送到 Redis Stream
    sendToRedisStream(plotType.getUserDraw().getId(), SecurityUtils.getCurrentUserId());
    log.info("任务已发送到 Redis Stream: userDrawId={}", plotType.getUserDraw().getId());
    return null;
});

try {
    rHandlerWrapper.doHandler(
        componentBean.getRedissonClient(),
        RedissonKey.getUserDrawKey(plotType.getUserDraw().getId())
    );
    return ResponseUtils.success();
} catch (Exception e) {
    log.error("任务提交失败: userDrawId={}", plotType.getUserDraw().getId(), e);
    return ResponseUtils.failureMsg("任务提交失败，请稍后重试");
}
```

**简化效果**：
- 代码行数：从 ~80 行减少到 ~15 行
- 复杂度：移除了灰度切换、数据库操作、超时检查等复杂逻辑
- 可维护性：逻辑清晰，易于理解和维护

---

### 3.2 消费者配置更新

#### 3.2.1 更新 UserDrawStreamConsumer 配置

**文件**: `goto/task/src/main/java/com/freedom/runner/UserDrawStreamConsumer.java`

**变更前**：
```java
@StreamListener(
    stream = RedisStreamConfig.STREAM_USER_DRAW_TASKS,
    group = RedisStreamConfig.GROUP_TASK_SERVICE,
    concurrency = 10,  // ❌ 与线程池配置不一致
    maxRetries = 5,    // ❌ 重试次数过多
    enableBackpressure = true
)
```

**变更后**：
```java
@StreamListener(
    stream = RedisStreamConfig.STREAM_USER_DRAW_TASKS,
    group = RedisStreamConfig.GROUP_TASK_SERVICE,
    concurrency = 4,   // ✅ 与 task.pool.user-draw.core-pool-size 一致
    maxRetries = 3,    // ✅ 合理的重试次数
    enableBackpressure = true
)
```

**配置说明**：

| 配置项 | 旧值 | 新值 | 理由 |
|--------|------|------|------|
| `concurrency` | 10 | 4 | 与 `application-product.yml` 中的 `task.pool.user-draw.core-pool-size: 4` 保持一致，避免过度并发 |
| `maxRetries` | 5 | 3 | 减少不必要的重试，加快失败任务的处理，3 次重试已足够 |
| `enableBackpressure` | true | true | 保持启用背压控制，防止消费者过载 |

---

## 4. 数据流设计

### 4.1 消息格式

**消息类**: `UserDrawStreamMessage`

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class UserDrawStreamMessage implements Serializable {
    /**
     * 用户绘图任务 ID
     */
    private Long userDrawId;

    /**
     * 用户 ID
     */
    private Long userId;
}
```

**序列化后的 JSON 格式**：
```json
{
  "userDrawId": 123456789,
  "userId": 987654321
}
```

**Redis Stream 存储格式**：
```
XADD stream:user_draw_tasks *
  payload "{\"userDrawId\":123456789,\"userId\":987654321}"
  timestamp 1710489600000
  retryCount 0
```

### 4.2 消息生命周期

```
1. 生产者发送
   PlotServiceImpl.sendToRedisStream()
   └─> userDrawProducer.send(message)
       └─> StreamTemplate 序列化为 JSON
           └─> 写入 Redis Stream

2. 消费者接收
   UserDrawStreamConsumer.handleUserDrawMessage()
   ├─> StreamTemplate 自动反序列化
   ├─> 获取分布式锁
   ├─> 添加到线程池
   └─> 自动 ACK（成功）或重试（失败）

3. 重试机制
   ├─> 第 1 次失败：立即重试
   ├─> 第 2 次失败：延迟重试
   ├─> 第 3 次失败：延迟重试
   └─> 超过 3 次：进入死信队列（如果配置）
```

---

## 5. 错误处理

### 5.1 生产者错误处理

**场景 1：序列化失败**
```java
// StreamTemplate 内部处理
catch (JsonProcessingException e) {
    throw new RuntimeException("Failed to serialize message", e);
}
```
**处理方式**：抛出异常，由调用方捕获并返回错误响应给用户

**场景 2：Redis 连接失败**
```java
catch (Exception e) {
    log.error("消息发送到 Redis Stream 失败: userDrawId={}, userId={}",
            userDrawId, userId, e);
    throw new RuntimeException("消息发送失败", e);
}
```
**处理方式**：记录日志，抛出异常，返回 "任务提交失败，请稍后重试"

### 5.2 消费者错误处理

**场景 1：UserDraw 不存在**
```java
if (userDraw == null) {
    log.error("UserDraw 不存在: userDrawId={}", userDrawId);
    return;  // 直接返回，消息被 ACK
}
```
**处理方式**：记录日志，直接返回（不重试）

**场景 2：线程池拒绝**
```java
if (!result.isResult()) {
    log.error("任务添加到线程池失败: userDrawId={}, reason={}",
            userDrawId, result.getMsg());
    if (!ThreadPoolRejectedException.class.equals(result.getErrorType())) {
        FeishuNotice.sendMsg(...);  // 发送飞书告警
    }
    return null;  // 不抛异常，消息被 ACK
}
```
**处理方式**：记录日志，发送告警，不重试（避免雪崩）

**场景 3：处理异常**
```java
catch (Exception e) {
    log.error("处理消息失败: userDrawId={}, userId={}, messageId={}",
            userDrawId, userId, context.getMessageId(), e);
    throw new RuntimeException("Failed to process message", e);
}
```
**处理方式**：抛出异常，触发重试机制（最多 3 次）

---

## 6. 配置管理

### 6.1 线程池配置

**文件**: `goto/task/src/main/resources/application-product.yml`

```yaml
task:
  pool:
    user-draw:
      core-pool-size: 4      # 核心线程数
      max-pool-size: 4       # 最大线程数
      keep-alive-seconds: 0  # 活跃时间
      queue-capacity: 0      # 队列容量
```

### 6.2 Redis Stream 配置

**文件**: `goto/common/src/main/java/com/freedom/config/RedisStreamConfig.java`

```java
public class RedisStreamConfig {
    public static final String STREAM_USER_DRAW_TASKS = "stream:user_draw_tasks";
    public static final String GROUP_TASK_SERVICE = "group:task_service";
    public static final long MESSAGE_TIMEOUT_MS = 60000;  // 60 秒
    public static final int MAX_RETRY_COUNT = 5;
    public static final double BUSY_THRESHOLD = 0.8;
    public static final long AUTOCLAIM_INTERVAL_MS = 30000;
    public static final long READ_BLOCK_MS = 5000;
    public static final int READ_COUNT = 1;
}
```

### 6.3 消费者配置对齐

| 配置项 | 配置文件 | 消费者注解 | 说明 |
|--------|----------|-----------|------|
| 并发数 | `task.pool.user-draw.core-pool-size: 4` | `concurrency = 4` | 保持一致 |
| 重试次数 | `RedisStreamConfig.MAX_RETRY_COUNT: 5` | `maxRetries = 3` | 消费者使用更保守的值 |

---

## 7. 测试策略

### 7.1 单元测试

**测试 1：生产者发送成功**
```java
@Test
void testSendToRedisStream_Success() {
    // Given
    Long userDrawId = 123L;
    Long userId = 456L;

    // When
    plotServiceImpl.sendToRedisStream(userDrawId, userId);

    // Then
    verify(userDrawProducer).send(any(UserDrawStreamMessage.class));
}
```

**测试 2：生产者发送失败**
```java
@Test
void testSendToRedisStream_Failure() {
    // Given
    when(userDrawProducer.send(any())).thenThrow(new RuntimeException("Redis error"));

    // When & Then
    assertThrows(RuntimeException.class, () -> {
        plotServiceImpl.sendToRedisStream(123L, 456L);
    });
}
```

### 7.2 集成测试

**测试 3：端到端消息流转**
```java
@Test
void testEndToEnd_MessageFlow() {
    // Given
    PlotType plotType = createTestPlotType();

    // When
    CommonResponse response = plotServiceImpl.customDrawPlot(plotType);

    // Then
    assertEquals(ResponseUtils.success(), response);

    // 等待消费者处理
    await().atMost(5, SECONDS).until(() -> {
        UserDraw userDraw = userDrawRepository.findById(plotType.getUserDraw().getId());
        return userDraw.getStatus() == CommonSerializableAnalysisStatus.ING;
    });
}
```

### 7.3 性能测试

**测试 4：并发发送性能**
```java
@Test
void testConcurrentSend_Performance() {
    // Given
    int concurrentRequests = 100;

    // When
    long startTime = System.currentTimeMillis();
    IntStream.range(0, concurrentRequests).parallel().forEach(i -> {
        plotServiceImpl.sendToRedisStream((long) i, 1L);
    });
    long endTime = System.currentTimeMillis();

    // Then
    long duration = endTime - startTime;
    assertTrue(duration < 5000, "Should complete within 5 seconds");
}
```

---

## 8. 迁移计划

### 8.1 迁移步骤

1. **代码重构**（开发环境）
   - [ ] 添加 `@StreamProducer` 字段注入
   - [ ] 重构 `sendToRedisStream()` 方法
   - [ ] 简化 `customDrawPlot()` 方法
   - [ ] 更新 `UserDrawStreamConsumer` 配置

2. **单元测试**（开发环境）
   - [ ] 编写生产者单元测试
   - [ ] 编写消费者单元测试
   - [ ] 验证所有测试通过

3. **集成测试**（测试环境）
   - [ ] 部署到测试环境
   - [ ] 执行端到端测试
   - [ ] 验证消息流转正常

4. **性能测试**（测试环境）
   - [ ] 并发发送测试
   - [ ] 消费者吞吐量测试
   - [ ] 监控 Redis Stream 性能

5. **生产部署**（生产环境）
   - [ ] 部署到生产环境
   - [ ] 监控消息队列状态
   - [ ] 观察错误日志和告警

6. **清理工作**（生产稳定后）
   - [ ] 移除 RabbitMQ 相关配置（如果不再使用）
   - [ ] 清理 MqMessage 数据库表（如果不再使用）
   - [ ] 更新文档

### 8.2 回滚方案

**如果出现问题，回滚步骤**：

1. **代码回滚**：
   ```bash
   git revert <commit-hash>
   git push origin main
   ```

2. **重新部署**：
   - 部署回滚后的代码
   - 验证 RabbitMQ 功能正常

3. **数据恢复**：
   - 如果删除了 MqMessage 表，从备份恢复

**回滚触发条件**：
- 消息发送失败率 > 5%
- 消息消费延迟 > 60 秒
- 系统出现严重错误

---

## 9. 监控和告警

### 9.1 监控指标

**生产者指标**：
- 消息发送成功率
- 消息发送延迟（P50, P95, P99）
- 消息发送失败次数

**消费者指标**：
- 消息消费速率（msg/s）
- 消息消费延迟
- 消费者并发数
- 重试次数统计

**Redis Stream 指标**：
- Stream 长度
- 消费者组延迟（lag）
- Pending 消息数量

### 9.2 告警规则

| 指标 | 阈值 | 告警级别 | 处理方式 |
|------|------|---------|---------|
| 消息发送失败率 | > 5% | 严重 | 立即检查 Redis 连接 |
| 消息消费延迟 | > 60s | 警告 | 检查消费者状态 |
| Stream 长度 | > 10000 | 警告 | 增加消费者并发数 |
| Pending 消息数 | > 1000 | 严重 | 检查消费者是否宕机 |

---

## 10. 风险评估

### 10.1 技术风险

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|---------|
| Redis Stream 性能不足 | 高 | 低 | 提前进行性能测试，准备扩容方案 |
| 消息丢失 | 高 | 低 | 启用持久化，配置死信队列 |
| 消费者宕机 | 中 | 中 | 配置多个消费者实例，启用健康检查 |
| 序列化失败 | 中 | 低 | 添加单元测试，验证消息格式 |

### 10.2 业务风险

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|---------|
| 用户任务提交失败 | 高 | 低 | 提供友好的错误提示，记录详细日志 |
| 任务处理延迟 | 中 | 中 | 监控消费延迟，及时调整并发数 |
| 历史数据不兼容 | 低 | 低 | 保留 RabbitMQ 代码作为备份 |

---

## 11. 总结

### 11.1 重构收益

1. **代码简洁**：
   - `customDrawPlot()` 方法从 ~80 行减少到 ~15 行
   - 移除了灰度切换、数据库操作等复杂逻辑

2. **类型安全**：
   - 使用 `UserDrawStreamMessage` 对象，编译时检查
   - 避免了手动构建 Map 的错误

3. **易于维护**：
   - 统一使用 `@StreamProducer` 注解
   - 符合 Redis Stream Starter 的最佳实践

4. **性能优化**：
   - 消费者并发数与线程池配置对齐
   - 减少不必要的重试次数

### 11.2 后续工作

1. **短期**（1-2 周）：
   - 完成代码重构和测试
   - 部署到生产环境
   - 监控运行状态

2. **中期**（1-2 月）：
   - 观察生产环境稳定性
   - 收集性能数据
   - 优化配置参数

3. **长期**（3-6 月）：
   - 将其他业务迁移到 Redis Stream
   - 完全移除 RabbitMQ 依赖
   - 统一消息队列架构

---

## 12. 参考文档

- [Redis Stream Spring Boot Starter - USAGE.md](../../../freedom-public/redis-stream-spring-boot-starter/docs/USAGE.md)
- [Redis Stream Spring Boot Starter - EXAMPLES.md](../../../freedom-public/redis-stream-spring-boot-starter/docs/EXAMPLES.md)
- [RabbitMQ 迁移到 Redis Stream 设计](../../plans/2026-03-09-rabbitmq-to-redis-stream-design.md)

---

**设计完成日期**: 2026-03-15
**下一步**: 等待用户审查，审查通过后创建实施计划
