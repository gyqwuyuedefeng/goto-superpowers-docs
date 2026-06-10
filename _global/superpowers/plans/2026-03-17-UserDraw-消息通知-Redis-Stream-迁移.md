# UserDraw 消息通知 Redis Stream 迁移实施计划

## 背景

UserDraw 功能已部分迁移到 Redis Stream：
- ✅ **任务消费**：已使用 Redis Stream 消费者处理任务
- ✅ **任务提交**：已使用 Redis Stream 发布任务
- ❌ **消息通知**：仍在使用 RabbitMQ 发布和消费通知消息

## 当前问题

**发送任务请求使用的是 Redis Stream，但是任务通知返回使用的是 RabbitMQ。**

### 具体表现

1. **任务提交流程**（已迁移）
   ```
   PlotServiceImpl
     → Redis Stream (发布任务)
     → UserDrawStreamConsumer (消费任务)
     → UserDrawRunnable (执行任务)
   ```

2. **消息通知流程**（未迁移，仍使用 RabbitMQ）
   ```
   UserDrawRunnable/DoUserDrawTask
     → RabbitMQ (发布通知)
     → UserDrawMessageConsumer (消费通知)
     → WebSocket (推送到前端)
   ```

### 问题分析

**不一致性**：
- 任务流使用 Redis Stream
- 通知流使用 RabbitMQ
- 导致系统依赖两套消息队列

**影响**：
- 增加系统复杂度
- 增加运维成本
- 消息流不统一

## 目标

**将 UserDraw 消息通知从 RabbitMQ 迁移到 Redis Stream，实现完整的 Redis Stream 消息流。**

迁移后的流程：
```
PlotServiceImpl
  → Redis Stream (发布任务)
  → UserDrawStreamConsumer (消费任务)
  → UserDrawRunnable (执行任务)
  → Redis Stream (发布通知)
  → UserDrawNotificationStreamConsumer (消费通知)
  → WebSocket (推送到前端)
```

## 需要迁移的代码

### 1. 消息发布端

**DoUserDrawTask.java** (第 249-253 行)
```java
// 当前代码（使用 RabbitMQ）
RabbitMQUtils.publish(
    componentBean.getCommonRabbitTemplate(),
    RabbitMQConstant.EXCHANGE_USER_DRAW_MESSAGE_NOTICE,
    RabbitMQConstant.QUEUE_USER_DRAW_MESSAGE_NOTICE,
    messageContainer
);
```

**需要改为**：
```java
// 使用 Redis Stream 发布通知
redisStreamTemplate.add(
    RedisStreamConstant.STREAM_USER_DRAW_NOTIFICATION,
    messageContainer
);
```

**其他发布位置**：
- `UserDrawRunnable.java` - 任务执行过程中的状态通知
- `UserDrawCommonUtils.java` - 状态变更通知
- 所有调用 `QUEUE_USER_DRAW_MESSAGE_NOTICE` 的地方

### 2. 消息消费端

**UserDrawMessageConsumer.java**
- 当前：RabbitMQ 消费者，监听 `QUEUE_USER_DRAW_MESSAGE_NOTICE`
- 需要：创建 Redis Stream 消费者，监听 `STREAM_USER_DRAW_NOTIFICATION`

**功能**：
- 接收 UserDraw 状态变更通知
- 通过 WebSocket 推送到前端
- 处理消息重试和失败

## 实施步骤

### 阶段 1：设计 Redis Stream 通知机制

**1.1 定义 Stream Key 和常量**

在 `RedisStreamConfig.java` 中添加：

```java
public class RedisStreamConfig {
    // 任务流（已存在）
    public static final String STREAM_USER_DRAW_TASKS = "stream:user_draw_tasks";
    public static final String GROUP_TASK_SERVICE = "group:task_service";

    // 通知流（新增）
    public static final String STREAM_USER_DRAW_NOTIFICATION = "stream:user_draw_notification";
    public static final String GROUP_NOTIFICATION_SERVICE = "group:notification_service";

    // 死信队列（新增）
    public static final String STREAM_USER_DRAW_NOTIFICATION_DLQ = "stream:user_draw_notification:dlq";
}
```

**1.2 消息格式设计**

**重要决策：创建专门的通知对象（UserDrawNotification）**

符合 `redis-stream-spring-boot-starter` 的设计理念，同时实现职责分离。

**UserDrawNotification 类设计：**

```java
package com.freedom.model;

import lombok.Data;
import java.time.LocalDateTime;

/**
 * UserDraw 通知对象
 *
 * 职责：封装通知相关的数据，与领域对象 UserDraw 分离
 */
@Data
public class UserDrawNotification {
    // 基本信息
    private Long id;                    // 任务 ID
    private Long userId;                // 用户 ID
    private String status;              // 任务状态

    // 通知信息
    private String noticeType;          // 通知类型（SUBMIT_SUCCESS, SUBMIT_FAIL 等）
    private String messageType;         // 消息类型（SUCCESS, ERROR, INFO）
    private String message;             // 消息内容

    // 元数据
    private LocalDateTime timestamp;    // 时间戳

    /**
     * 从 UserDraw 构建通知对象
     */
    public static UserDrawNotification from(UserDraw userDraw,
                                            String noticeType,
                                            String messageType,
                                            String message) {
        UserDrawNotification notification = new UserDrawNotification();
        notification.setId(userDraw.getId());
        notification.setUserId(userDraw.getUserId());
        notification.setStatus(userDraw.getStatus());
        notification.setNoticeType(noticeType);
        notification.setMessageType(messageType);
        notification.setMessage(message);
        notification.setTimestamp(LocalDateTime.now());
        return notification;
    }

    /**
     * 转换为 UserDraw 对象（用于 WebSocket 推送）
     */
    public UserDraw toUserDraw() {
        UserDraw userDraw = new UserDraw();
        userDraw.setId(this.id);
        userDraw.setUserId(this.userId);
        userDraw.setStatus(this.status);
        // 其他字段根据需要设置
        return userDraw;
    }
}
```

**Redis Stream 中的存储格式（由 starter 自动处理）：**

```json
{
  "payload": "{\"id\":123,\"userId\":456,\"status\":\"SUCCESS\",\"noticeType\":\"SUBMIT_SUCCESS\",\"messageType\":\"SUCCESS\",\"message\":\"提交成功\",\"timestamp\":\"2026-03-18T01:30:00\"}",
  "timestamp": 1710748800000,
  "retryCount": 0
}
```

**使用示例：**

```java
// 发布端：使用 StreamTemplate<UserDrawNotification>
@StreamProducer(stream = RedisStreamConfig.STREAM_USER_DRAW_NOTIFICATION)
private StreamTemplate<UserDrawNotification> notificationProducer;

UserDrawNotification notification = UserDrawNotification.from(
    userDraw,
    CommonAnalysisNoticeType.SUBMIT_SUCCESS.getType(),
    MessageType.SUCCESS,
    "提交成功"
);
notificationProducer.send(notification);

// 消费端：直接接收 UserDrawNotification 对象
@StreamListener(...)
public void consume(UserDrawNotification notification, MessageContext context) {
    // 构建 MessageContainer（仅用于 WebSocket 推送）
    MessageContainer<UserDraw> messageContainer = buildMessageContainer(notification);
    userDrawWebsocket.notice(componentBean, messageContainer);
}
```

**设计优势：**
- ✅ 职责分离：UserDraw 是领域对象，UserDrawNotification 是通知对象
- ✅ 符合 DDD 原则：明确区分业务数据和通知数据
- ✅ 更灵活：可以根据需要添加通知特有的字段
- ✅ 符合 starter 的设计理念（直接序列化业务对象）
- ✅ 代码更清晰，职责更明确
- ✅ 前端无需改动（WebSocket 接口保持不变）

**1.3 消费者配置参数**

基于现有 RabbitMQ 配置（2-5 个并发消费者），设计 Redis Stream 消费者配置：

```java
@StreamListener(
    stream = RedisStreamConfig.STREAM_USER_DRAW_NOTIFICATION,
    group = RedisStreamConfig.GROUP_NOTIFICATION_SERVICE,
    concurrency = 2,                    // 与 RabbitMQ 保持一致
    maxRetries = 3,                     // 重试 3 次
    enableBackpressure = false,         // 通知不需要背压控制
    deadLetterStream = RedisStreamConfig.STREAM_USER_DRAW_NOTIFICATION_DLQ,
    backoff = @Backoff(
        delay = 1000,                   // 初始延迟 1 秒
        multiplier = 2.0                // 指数退避：1s, 2s, 4s
    ),
    messageTimeout = 30000              // 30 秒超时
)
```

### 阶段 2：实现 Redis Stream 通知发布

**2.1 创建 UserDrawNotification 类**

在 `com.freedom.model` 包中创建：

```java
package com.freedom.model;

import lombok.Data;
import java.time.LocalDateTime;

/**
 * UserDraw 通知对象
 *
 * 职责：封装通知相关的数据，与领域对象 UserDraw 分离
 */
@Data
public class UserDrawNotification {
    // 基本信息
    private Long id;                    // 任务 ID
    private Long userId;                // 用户 ID
    private String status;              // 任务状态

    // 通知信息
    private String noticeType;          // 通知类型
    private String messageType;         // 消息类型（SUCCESS/ERROR/INFO）
    private String message;             // 消息内容

    // 元数据
    private LocalDateTime timestamp;    // 时间戳

    /**
     * 从 UserDraw 构建通知对象
     */
    public static UserDrawNotification from(UserDraw userDraw,
                                            String noticeType,
                                            String messageType,
                                            String message) {
        UserDrawNotification notification = new UserDrawNotification();
        notification.setId(userDraw.getId());
        notification.setUserId(userDraw.getUserId());
        notification.setStatus(userDraw.getStatus());
        notification.setNoticeType(noticeType);
        notification.setMessageType(messageType);
        notification.setMessage(message);
        notification.setTimestamp(LocalDateTime.now());
        return notification;
    }

    /**
     * 转换为 UserDraw 对象（用于 WebSocket 推送）
     */
    public UserDraw toUserDraw() {
        UserDraw userDraw = new UserDraw();
        userDraw.setId(this.id);
        userDraw.setUserId(this.userId);
        userDraw.setStatus(this.status);
        // 其他字段根据需要设置
        return userDraw;
    }
}
```

**2.2 创建统一的通知发布服务**

创建 `UserDrawNotificationService.java`：

```java
@Service
@Slf4j
public class UserDrawNotificationService {

    // 使用 @StreamProducer 注解自动注入 StreamTemplate
    @StreamProducer(stream = RedisStreamConfig.STREAM_USER_DRAW_NOTIFICATION)
    private StreamTemplate<UserDrawNotification> redisStreamProducer;

    private final RabbitTemplate commonRabbitTemplate;

    @Value("${notification.redis-stream.enabled:false}")
    private boolean redisStreamEnabled;

    @Value("${notification.redis-stream.percentage:0}")
    private int redisStreamPercentage; // 0-100，用于灰度发布

    /**
     * 发布 UserDraw 通知（支持灰度发布）
     */
    public void publishNotification(UserDrawNotification notification) {
        Long userId = notification.getUserId();

        // 灰度策略：根据 userId 哈希值决定使用哪个队列
        boolean useRedisStream = redisStreamEnabled &&
                                 (userId % 100 < redisStreamPercentage);

        if (useRedisStream) {
            publishToRedisStream(notification);
        } else {
            publishToRabbitMQ(notification);
        }
    }

    /**
     * 发布到 Redis Stream
     */
    private void publishToRedisStream(UserDrawNotification notification) {
        try {
            // 直接发送 UserDrawNotification 对象，由 StreamTemplate 自动序列化
            String messageId = redisStreamProducer.send(notification);

            log.info("通知已发布到 Redis Stream: taskId={}, userId={}, messageId={}",
                     notification.getId(), notification.getUserId(), messageId);

        } catch (Exception e) {
            log.error("发布通知到 Redis Stream 失败: taskId={}", notification.getId(), e);
            // 降级到 RabbitMQ
            publishToRabbitMQ(notification);
        }
    }

    /**
     * 发布到 RabbitMQ（降级方案）
     */
    private void publishToRabbitMQ(UserDrawNotification notification) {
        // 构建 MessageContainer（仅用于 RabbitMQ）
        MessageContainer<UserDraw> messageContainer = buildMessageContainer(notification);

        RabbitMQUtils.publish(
            commonRabbitTemplate,
            RabbitMQConstant.EXCHANGE_USER_DRAW_MESSAGE_NOTICE,
            RabbitMQConstant.QUEUE_USER_DRAW_MESSAGE_NOTICE,
            messageContainer
        );
    }

    /**
     * 从 UserDrawNotification 构建 MessageContainer
     *
     * 此方法用于兼容 RabbitMQ 的历史格式
     * 未来 RabbitMQ 完全下线后，此方法可以删除
     */
    private MessageContainer<UserDraw> buildMessageContainer(UserDrawNotification notification) {
        MessageContainer<UserDraw> container = new MessageContainer<>();

        // 转换为 UserDraw
        UserDraw userDraw = notification.toUserDraw();
        container.setData(userDraw);

        // 构建 BasicMessage
        BasicMessage basicMessage = new BasicMessage();
        basicMessage.setId(notification.getId());
        basicMessage.setTopBusinessType(TopBusinessType.USER_DRAW.getType());
        basicMessage.setNoticeType(notification.getNoticeType());
        basicMessage.setMessageType(notification.getMessageType());
        basicMessage.setMessage(notification.getMessage());

        container.setBasicMessage(basicMessage);
        return container;
    }
}
```

**2.2 添加监控指标**

创建 `UserDrawNotificationMetrics.java`：

```java
@Component
public class UserDrawNotificationMetrics {

    private final MeterRegistry meterRegistry;

    public void recordPublishSuccess(String channel) {
        meterRegistry.counter("user_draw.notification.publish.success",
                             "channel", channel).increment();
    }

    public void recordPublishFailure(String channel) {
        meterRegistry.counter("user_draw.notification.publish.failure",
                             "channel", channel).increment();
    }

    public void recordConsumeLatency(long latencyMs) {
        meterRegistry.timer("user_draw.notification.consume.latency")
                     .record(latencyMs, TimeUnit.MILLISECONDS);
    }

    public void recordDLQSize(long size) {
        meterRegistry.gauge("user_draw.notification.dlq.size", size);
    }
}
```

**2.3 修改所有发布位置**

使用 IDE 的"查找所有引用"功能，找到所有使用 `QUEUE_USER_DRAW_MESSAGE_NOTICE` 的地方：

- `DoUserDrawTask.java` (第 249-253 行)
- `UserDrawRunnable.java`
- `UserDrawCommonUtils.java`
- 其他相关文件

**统一改造为使用 UserDrawNotification：**

```java
// 旧代码：构建 MessageContainer 并发布到 RabbitMQ
MessageContainer<UserDraw> messageContainer = new MessageContainer<>();
messageContainer.setData(userDraw);

BasicMessage basicMessage = new BasicMessage();
basicMessage.setId(userDraw.getId());
basicMessage.setNoticeType(CommonAnalysisNoticeType.SUBMIT_FAIL.getType());
basicMessage.setMessageType(MessageType.ERROR);
basicMessage.setMessage("提交失败");
messageContainer.setBasicMessage(basicMessage);

RabbitMQUtils.publish(
    componentBean.getCommonRabbitTemplate(),
    RabbitMQConstant.EXCHANGE_USER_DRAW_MESSAGE_NOTICE,
    RabbitMQConstant.QUEUE_USER_DRAW_MESSAGE_NOTICE,
    messageContainer
);

// 新代码：使用 UserDrawNotification
UserDrawNotification notification = UserDrawNotification.from(
    userDraw,
    CommonAnalysisNoticeType.SUBMIT_FAIL.getType(),
    MessageType.ERROR,
    "提交失败"
);
userDrawNotificationService.publishNotification(notification);
```

**改造示例（DoUserDrawTask.java）：**

```java
public static void noticeFail(ComponentBean componentBean, UserDraw userDraw, String msg) {
    // 更新状态
    UserDrawCommonUtils.changeStatus(
        componentBean,
        CommonSerializableAnalysisStatus.FAIL.getType(),
        userDraw,
        true
    );

    // 构建通知对象
    UserDrawNotification notification = UserDrawNotification.from(
        userDraw,
        CommonAnalysisNoticeType.SUBMIT_FAIL.getType(),
        MessageType.ERROR,
        ObjectUtils.isEmpty(msg) ? "提交失败" : msg
    );

    // 发布通知
    componentBean.getUserDrawNotificationService().publishNotification(notification);
}
```

**改造要点：**

1. **删除 MessageContainer 构建逻辑**
2. **使用 UserDrawNotification.from() 构建通知对象**
3. **调用 userDrawNotificationService.publishNotification()**
4. **保持业务逻辑不变**（只是改变通知发送方式）

**2.4 配置灰度发布参数**

在 `application.yml` 中添加：

```yaml
notification:
  redis-stream:
    enabled: true      # 是否启用 Redis Stream
    percentage: 0      # 灰度百分比：0-100
```

**灰度发布计划：**
- 第 1 天：`percentage: 10` - 10% 流量
- 第 2 天：`percentage: 50` - 50% 流量
- 第 3 天：`percentage: 100` - 100% 流量

### 阶段 3：实现 Redis Stream 通知消费

**3.1 创建 Redis Stream 消费者**

创建 `UserDrawNotificationStreamConsumer.java`：

```java
@Slf4j
@Component
public class UserDrawNotificationStreamConsumer {

    private final ComponentBean componentBean;
    private final UserDrawWebsocket userDrawWebsocket;

    @StreamListener(
        stream = RedisStreamConfig.STREAM_USER_DRAW_NOTIFICATION,
        group = RedisStreamConfig.GROUP_NOTIFICATION_SERVICE,
        concurrency = 2,                    // 与 RabbitMQ 保持一致
        maxRetries = 3,                     // 重试 3 次
        enableBackpressure = false,         // 通知不需要背压控制
        deadLetterStream = RedisStreamConfig.STREAM_USER_DRAW_NOTIFICATION_DLQ,
        backoff = @Backoff(
            delay = 1000,                   // 初始延迟 1 秒
            multiplier = 2.0                // 指数退避
        ),
        messageTimeout = 30000              // 30 秒超时
    )
    public void consume(UserDrawNotification notification, MessageContext context) {
        long startTime = System.currentTimeMillis();

        try {
            log.info("开始处理通知: taskId={}, userId={}, messageId={}",
                     notification.getId(),
                     notification.getUserId(),
                     context.getMessageId());

            // 构建 MessageContainer（仅用于 WebSocket 推送）
            MessageContainer<UserDraw> messageContainer = buildMessageContainer(notification);

            // 推送到 WebSocket
            userDrawWebsocket.notice(componentBean, messageContainer);

            // 记录成功指标
            long latency = System.currentTimeMillis() - startTime;
            log.info("通知处理成功: taskId={}, latency={}ms", notification.getId(), latency);

        } catch (WebSocketNotMatchException e) {
            // WebSocket 连接不存在，直接 ACK，不重试
            log.warn("WebSocket 连接不存在，跳过通知: taskId={}", notification.getId());
            // 不抛出异常，消息会被自动 ACK
            return;

        } catch (Exception e) {
            log.error("处理通知失败: taskId={}, messageId={}",
                      notification.getId(), context.getMessageId(), e);
            // 抛出异常触发重试
            throw new RuntimeException("Failed to process notification", e);
        }
    }

    /**
     * 从 UserDrawNotification 构建 MessageContainer
     *
     * MessageContainer 是 WebSocket 层的概念，只在推送时构建
     */
    private MessageContainer<UserDraw> buildMessageContainer(UserDrawNotification notification) {
        MessageContainer<UserDraw> container = new MessageContainer<>();

        // 转换为 UserDraw
        UserDraw userDraw = notification.toUserDraw();
        container.setData(userDraw);

        // 构建 BasicMessage
        BasicMessage basicMessage = new BasicMessage();
        basicMessage.setId(notification.getId());
        basicMessage.setTopBusinessType(TopBusinessType.USER_DRAW.getType());
        basicMessage.setNoticeType(notification.getNoticeType());
        basicMessage.setMessageType(notification.getMessageType());
        basicMessage.setMessage(notification.getMessage());

        container.setBasicMessage(basicMessage);
        return container;
    }
}
```

**关键设计说明：**

1. **直接接收 UserDrawNotification 对象**
   - 由 `@StreamListener` 自动反序列化
   - 符合 starter 的设计理念

2. **MessageContainer 只在 WebSocket 推送时构建**
   - MessageContainer 是 WebSocket 层的概念
   - 不应该传播到消息队列层

3. **WebSocket 异常处理**
   - `WebSocketNotMatchException`：不重试，直接 ACK
   - 其他异常：重试 3 次，失败后进入死信队列

**3.2 WebSocket 异常处理说明**

**关键设计决策：**
- `WebSocketNotMatchException`：用户断开连接，**不重试**，直接 ACK
- 其他异常：系统错误，**重试 3 次**，失败后进入死信队列

**原因：**
- 用户断开连接后，重试是无意义的，只会浪费资源
- 系统错误（如序列化失败）可能是临时问题，值得重试

**3.3 死信队列监控**

创建定时任务监控死信队列：

```java
@Component
@Slf4j
public class UserDrawNotificationDLQMonitor {

    private final RedisStreamTemplate redisStreamTemplate;
    private final UserDrawNotificationMetrics metrics;

    @Scheduled(fixedRate = 60000) // 每分钟检查一次
    public void monitorDLQ() {
        try {
            Long dlqSize = redisStreamTemplate.size(
                RedisStreamConfig.STREAM_USER_DRAW_NOTIFICATION_DLQ
            );

            metrics.recordDLQSize(dlqSize);

            if (dlqSize > 100) {
                log.error("死信队列消息堆积: size={}", dlqSize);
                // 发送告警
                FeishuNotice.sendMsg(
                    FeishuRobotEnum.MSG_ERROR,
                    "UserDraw 通知死信队列告警",
                    "死信队列消息数量: " + dlqSize
                );
            }
        } catch (Exception e) {
            log.error("监控死信队列失败", e);
        }
    }
}
```

### 阶段 4：测试和灰度发布

**4.1 单元测试和集成测试**

```java
@SpringBootTest
public class UserDrawNotificationIntegrationTest {

    @Test
    public void testPublishAndConsume() {
        // 1. 发布消息
        MessageContainer<UserDraw> messageContainer = createTestMessage();
        userDrawNotificationService.publishNotification(messageContainer);

        // 2. 等待消费
        await().atMost(5, TimeUnit.SECONDS)
               .until(() -> isMessageConsumed(messageContainer.getData().getId()));

        // 3. 验证 WebSocket 收到通知
        verify(userDrawWebsocket).notice(any(), eq(messageContainer));
    }

    @Test
    public void testRetryMechanism() {
        // 1. 模拟消费失败
        doThrow(new RuntimeException("Test error"))
            .when(userDrawWebsocket).notice(any(), any());

        // 2. 发布消息
        userDrawNotificationService.publishNotification(createTestMessage());

        // 3. 验证重试 3 次
        await().atMost(10, TimeUnit.SECONDS)
               .until(() -> getRetryCount() == 3);

        // 4. 验证进入死信队列
        Long dlqSize = redisStreamTemplate.size(
            RedisStreamConfig.STREAM_USER_DRAW_NOTIFICATION_DLQ
        );
        assertThat(dlqSize).isEqualTo(1);
    }

    @Test
    public void testWebSocketNotMatchException() {
        // 1. 模拟 WebSocket 连接不存在
        doThrow(new WebSocketNotMatchException())
            .when(userDrawWebsocket).notice(any(), any());

        // 2. 发布消息
        userDrawNotificationService.publishNotification(createTestMessage());

        // 3. 验证不重试，直接 ACK
        await().atMost(5, TimeUnit.SECONDS)
               .until(() -> isMessageAcked());

        // 4. 验证不进入死信队列
        Long dlqSize = redisStreamTemplate.size(
            RedisStreamConfig.STREAM_USER_DRAW_NOTIFICATION_DLQ
        );
        assertThat(dlqSize).isEqualTo(0);
    }
}
```

**4.2 功能测试**
- [ ] 任务提交后能收到通知
- [ ] 任务执行中能收到进度通知
- [ ] 任务完成后能收到完成通知
- [ ] 任务失败后能收到失败通知
- [ ] 任务取消后能收到取消通知
- [ ] WebSocket 连接不存在时不重试

**4.3 性能测试**
- [ ] 通知延迟 < 100ms
- [ ] 高并发下通知不丢失（并发 100 个任务）
- [ ] 消息顺序性验证（同一用户的消息）

**4.4 异常测试**
- [ ] 消费者宕机后能自动恢复
- [ ] 消息重试机制正常（最多 3 次）
- [ ] 死信队列正常工作
- [ ] Redis 连接断开后能降级到 RabbitMQ

**4.5 灰度发布**

**第 1 天：10% 流量**
```yaml
notification:
  redis-stream:
    enabled: true
    percentage: 10
```
- 观察监控指标：发布成功率、消费延迟、死信队列大小
- 验证 10% 用户能正常收到通知
- 验证 90% 用户仍使用 RabbitMQ

**第 2 天：50% 流量**
```yaml
notification:
  redis-stream:
    percentage: 50
```
- 继续观察监控指标
- 对比 Redis Stream 和 RabbitMQ 的性能差异
- 验证无异常后继续

**第 3 天：100% 流量**
```yaml
notification:
  redis-stream:
    percentage: 100
```
- 全量切换到 Redis Stream
- 密切监控 24 小时
- 确认无问题后进入阶段 5

**监控指标：**
- 发布成功率 > 99.9%
- 消费延迟 < 100ms (P99)
- 死信队列大小 < 10
- 无 ERROR 级别日志

### 阶段 5：清理 RabbitMQ 代码

**5.1 禁用 RabbitMQ 消费者**
```java
//@Component  // 禁用 RabbitMQ 消费者
public class UserDrawMessageConsumer extends BaseConsumer {
    // ...
}
```

**5.2 验证功能正常**
- 在测试环境运行 1-2 天
- 确认没有 RabbitMQ 相关错误

**5.3 删除 RabbitMQ 代码**
- 删除 `UserDrawMessageConsumer.java`
- 删除相关配置
- 删除相关常量

## 技术细节

### Redis Stream vs RabbitMQ

| 特性 | RabbitMQ | Redis Stream |
|------|----------|--------------|
| 消息持久化 | ✅ | ✅ |
| 消费者组 | ✅ | ✅ |
| 消息确认 | ✅ ACK/NACK | ✅ XACK |
| 消息重试 | ✅ | ✅ XPENDING |
| 死信队列 | ✅ | ⚠️ 需要自己实现 |
| 消息顺序 | ⚠️ 需要配置 | ✅ 天然有序 |
| 性能 | 中 | 高 |

### 消息确认机制

**RabbitMQ**：
```java
channel.basicAck(deliveryTag, false);
```

**Redis Stream**：
```java
streamOperations.acknowledge(
    group,
    record.getId()
);
```

### 消息重试机制

**RabbitMQ**：
- 使用 `@Retryable` 注解
- 配置重试次数和延迟

**Redis Stream**：
- 使用 XPENDING 查询未确认消息
- 使用 XCLAIM 重新分配消息
- 定时任务自动认领超时消息

## 风险评估

### 🔴 高风险

| 风险 | 缓解措施 | 责任人 |
|------|---------|--------|
| **消息格式不兼容导致前端无法接收通知** | 1. 保持 MessageContainer 格式不变<br>2. 在测试阶段验证前端能正确解析<br>3. 使用相同的序列化方式（JSON） | 开发 |
| **多个发布位置遗漏导致部分通知仍使用 RabbitMQ** | 1. 使用 IDE "查找所有引用"功能<br>2. 创建统一的发布服务，强制使用<br>3. 在测试阶段监控 RabbitMQ 队列 | 开发 |

### 🟡 中风险

| 风险 | 缓解措施 | 责任人 |
|------|---------|--------|
| **WebSocket 连接不存在时的重试风暴** | 1. 捕获 WebSocketNotMatchException，直接 ACK<br>2. 不重试，避免资源浪费 | 开发 |
| **死信队列消息堆积** | 1. 实现死信队列监控和告警<br>2. 定期清理或人工处理<br>3. 设置告警阈值（> 100 条） | 运维 |
| **灰度发布期间两套系统并存** | 1. 统一发布服务自动路由<br>2. 监控两套系统的指标<br>3. 快速回滚机制 | 开发 |

### 🟢 低风险

| 风险 | 说明 | 缓解措施 |
|------|------|---------|
| **消息顺序性问题** | **现有 RabbitMQ 已存在此问题**（2-5 个并发消费者），迁移不会让情况变差 | 1. Redis Stream 使用 concurrency=2，与 RabbitMQ 一致<br>2. 如需严格顺序，可降低并发或前端排序 |
| **通知延迟变化** | Redis Stream 性能通常优于 RabbitMQ | 1. 性能测试验证延迟 < 100ms<br>2. 如不达标，调整 concurrency 参数 |
| **前端兼容性** | 保持 WebSocket 接口不变 | 无需前端改动 |

### 风险总结

**新增风险：** 0 个（所有风险都是现有系统已存在或可控的）

**降低的风险：**
- 系统复杂度降低（统一到 Redis Stream）
- 运维成本降低（减少一套消息队列）
- 性能提升（Redis Stream 通常更快）

## 回滚方案

如果迁移后出现问题，可以快速回滚：

1. **重新启用 RabbitMQ 消费者**
   ```java
   @Component  // 恢复注解
   public class UserDrawMessageConsumer extends BaseConsumer {
       // ...
   }
   ```

2. **切换发布端回 RabbitMQ**
   - 修改 `DoUserDrawTask.java`
   - 恢复 RabbitMQ 发布逻辑

3. **重启服务**
   - 无需数据迁移
   - 立即生效

## 时间估算

| 阶段 | 工作内容 | 工作量 | 时间 |
|------|---------|--------|------|
| 阶段 0 | **准备工作**<br>- 创建统一发布服务<br>- 添加监控指标<br>- 配置灰度参数 | 3-4 小时 | 0.5 天 |
| 阶段 1 | **设计 Redis Stream 通知机制**<br>- 定义常量和配置<br>- 设计消息格式<br>- 设计消费者参数 | 2-3 小时 | 0.5 天 |
| 阶段 2 | **实现发布端**<br>- 实现统一发布服务<br>- 修改所有发布位置<br>- 实现灰度发布逻辑 | 4-6 小时 | 1 天 |
| 阶段 3 | **实现消费端**<br>- 创建 Redis Stream 消费者<br>- 实现 WebSocket 推送<br>- 实现死信队列监控 | 6-8 小时 | 1 天 |
| 阶段 4 | **测试和灰度发布**<br>- 单元测试和集成测试<br>- 功能测试、性能测试<br>- 灰度发布（10% → 50% → 100%） | 3 天 | 3 天 |
| 阶段 5 | **清理 RabbitMQ 代码**<br>- 禁用 RabbitMQ 消费者<br>- 验证功能正常<br>- 删除相关代码 | 2-3 小时 | 0.5 天 |
| **总计** | | **约 30 小时** | **7 天** |

**说明：**
- 阶段 0 是新增的准备阶段，用于搭建基础设施
- 阶段 4 增加了灰度发布时间（3 天），每个灰度阶段观察 1 天
- 总时间从 4 天增加到 7 天，但风险大幅降低

## 下一步行动

1. ✅ 创建本实施计划
2. ✅ 完成设计审查和优化
3. ⏳ 阶段 0：准备工作（统一发布服务、监控指标）
4. ⏳ 阶段 1：设计 Redis Stream 通知机制
5. ⏳ 阶段 2：实现 Redis Stream 通知发布
6. ⏳ 阶段 3：实现 Redis Stream 通知消费
7. ⏳ 阶段 4：测试和灰度发布
8. ⏳ 阶段 5：清理 RabbitMQ 代码

---

**创建时间**：2026-03-17
**更新时间**：2026-03-18
**状态**：设计审查完成，待实施
**优先级**：高

## 附录：关键设计决策

### 1. 消息格式设计
**决策：** 创建专门的 UserDrawNotification 对象
**原因：**
- ✅ 职责分离：UserDraw 是领域对象，UserDrawNotification 是通知对象
- ✅ 符合 DDD 原则：明确区分业务数据和通知数据
- ✅ 更灵活：可以根据需要添加通知特有的字段
- ✅ 符合 starter 的设计理念（直接序列化业务对象）
- ✅ 代码更清晰，职责更明确

**备选方案：** 在 UserDraw 中添加 `@Transient` 字段
- 缺点：职责不清晰，领域对象被污染

### 2. 使用 @StreamProducer 注解
**决策：** 使用 `@StreamProducer` 和 `StreamTemplate<UserDrawNotification>`
**原因：**
- 符合 starter 的注解驱动设计
- 自动注入，无需手动创建 StreamTemplate
- 类型安全，编译时检查

### 3. 并发消费者数量
**决策：** 使用 `concurrency = 2`，与现有 RabbitMQ 一致
**原因：** 现有系统已验证此配置可行，消息顺序性不是关键需求

### 4. WebSocket 异常处理
**决策：** `WebSocketNotMatchException` 不重试，直接 ACK
**原因：** 用户断开连接后重试无意义，避免资源浪费

### 5. 灰度发布策略
**决策：** 10% → 50% → 100%，每阶段观察 1 天
**原因：** 逐步验证，降低风险，快速发现问题

### 6. 降级方案
**决策：** Redis Stream 发布失败时自动降级到 RabbitMQ
**原因：** 确保通知不丢失，提高系统可用性

### 7. MessageContainer 的使用范围
**决策：** MessageContainer 只在 WebSocket 层使用
**原因：**
- MessageContainer 是 WebSocket 的接口格式
- 不应该传播到消息队列层
- 消息队列层使用更简洁的业务对象（UserDrawNotification）
- 职责更清晰，层次更分明
