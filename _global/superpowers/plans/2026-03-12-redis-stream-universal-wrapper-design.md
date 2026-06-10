# Redis Stream 通用封装设计方案

## 文档信息

- **创建日期**：2026-03-12
- **版本**：v1.0
- **状态**：待批准

## 一、设计目标

将现有的 Redis Stream 实现（用户绘图任务）抽象为通用的 Spring Boot Starter，支持：
- ✅ 异步任务处理（长时间运行的任务）
- ✅ 通用消息队列场景
- ✅ 实时数据流处理

### 1.1 核心目标

1. **可复用性**：可在多个项目和业务场景中使用
2. **易用性**：注解驱动，符合 Spring 开发习惯
3. **类型安全**：泛型支持，编译时类型检查
4. **可靠性**：自动重试、死信队列、幂等性保证
5. **灵活性**：支持全局和细粒度配置
6. **可观测性**：完整的监控指标和健康检查

## 二、核心设计决策

| 维度 | 决策 | 理由 |
|------|------|------|
| **封装层次** | 中层次封装 - 开箱即用的组件 | 平衡易用性和灵活性 |
| **消息类型** | 强类型消息 - 泛型 + 序列化框架 | 类型安全，编译时检查 |
| **配置方式** | 注解驱动 - 类似 Spring Kafka | 符合 Spring 生态，易于理解 |
| **错误处理** | 自动重试 + 死信队列 | 可靠性高，符合现有实现 |
| **配置优先级** | 注解 > 全局配置 > 默认值 | 支持多业务场景的不同配置 |


## 三、架构设计

### 3.1 模块结构

```
freedom-public/
└── redis-stream-spring-boot-starter/  (独立通用模块)
    ├── src/main/java/com/freedom/stream/
    │   ├── autoconfigure/   (自动配置)
    │   ├── annotation/      (注解定义)
    │   ├── core/           (核心组件)
    │   ├── config/         (配置属性)
    │   ├── support/        (支持类)
    │   ├── serialization/  (序列化)
    │   └── monitoring/     (监控)
    ├── src/main/resources/
    │   └── META-INF/
    │       └── spring.factories  (自动配置注册)
    └── pom.xml
        └── 依赖: Spring Boot, Redisson, Jackson
```

### 3.2 依赖关系

```
redis-stream-spring-boot-starter (独立)
    ↓ (只依赖框架)
    Spring Boot + Redisson + Jackson

goto/common
    ↓ (依赖 starter)
    redis-stream-spring-boot-starter
    ↓ (包含业务配置)
    RedisStreamConfig.java (业务常量配置)

goto/task
    ↓ (依赖 goto/common)
    goto/common
    ↓ (使用 starter 的注解和组件)
    @StreamListener, StreamTemplate 等
```

### 3.3 职责划分

1. **redis-stream-spring-boot-starter**（通用层）：
   - 提供注解：`@EnableRedisStream`, `@StreamListener`, `@StreamProducer`
   - 提供核心组件：`StreamTemplate<T>`, `StreamConsumer<T>`, `StreamProducer<T>`
   - 提供自动配置：`RedisStreamAutoConfiguration`
   - 提供配置属性：`RedisStreamProperties`（从 application.yml 读取）
   - 提供支持类：背压控制、重试处理、死信处理等

2. **goto/common/RedisStreamConfig.java**（业务层）：
   - 定义业务常量：stream 名称、group 名称、超时时间等
   - 作为业务配置被 goto/task 使用
   - 不是 starter 的一部分，而是使用 starter 的业务代码


## 四、核心组件设计

### 4.1 StreamTemplate<T> - 核心模板类

提供类型安全的 Redis Stream 操作：

```java
public class StreamTemplate<T> {
    private final RedissonClient redissonClient;
    private final MessageSerializer<T> serializer;
    private final String streamName;
    
    // 发送消息到 Stream
    public StreamMessageId send(T message);
    
    // 批量发送消息
    public List<StreamMessageId> sendBatch(List<T> messages);
    
    // 发送消息并返回结果（用于追踪）
    public SendResult<T> sendWithResult(T message);
}
```

### 4.2 StreamProducer<T> - 生产者接口

提供更高层次的抽象，支持拦截器、事务等：

```java
public interface StreamProducer<T> {
    StreamMessageId send(T message);
    StreamMessageId send(String stream, T message);
    List<StreamMessageId> sendBatch(List<T> messages);
    SendResult<T> sendAndWait(T message);
}
```

### 4.3 AbstractStreamConsumer<T> - 消费者抽象类

提供消费者的基础实现，子类只需要实现业务逻辑：

```java
public abstract class AbstractStreamConsumer<T> implements CommandLineRunner {
    // 消费消息的主循环
    private void consumeMessages();
    
    // 处理单条消息
    private void processMessage(StreamMessageId messageId, Map<Object, Object> body);
    
    // 业务逻辑处理方法（子类实现）
    protected abstract void consume(T message, MessageContext context) throws Exception;
}
```

### 4.4 MessageContext - 消息上下文

提供消息的元数据和上下文信息：

```java
public class MessageContext {
    private final StreamMessageId messageId;
    private final Map<Object, Object> rawBody;
    private final StreamConsumerConfig config;
    private final long receiveTimestamp;
    private int retryCount;
    
    public String getMessageId();
    public int getRetryCount();
    public long getMessageDelay();
    public boolean isRetry();
    
    // ACK 消息（在业务代码的 finally 块中调用）
    public void ack();
}
```

### 4.5 支持类

**BackpressureController - 背压控制器**：
```java
public class BackpressureController {
    public boolean shouldPause();
    public double getCurrentUsage();
}
```

**RetryHandler - 重试处理器**：
```java
public class RetryHandler {
    public void handleFailure(StreamMessageId messageId, 
                             Map<Object, Object> body,
                             Exception error,
                             MessageContext context);
}
```

**DeadLetterHandler - 死信处理器**：
```java
public class DeadLetterHandler {
    public void handle(StreamMessageId messageId,
                      Map<Object, Object> body,
                      Exception error,
                      MessageContext context);
}
```


## 五、注解和自动配置设计

### 5.1 核心注解

**@EnableRedisStream - 启用 Redis Stream 支持**：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(RedisStreamAutoConfiguration.class)
public @interface EnableRedisStream {
    boolean autoConfiguration() default true;
    String[] basePackages() default {};
}
```

**@StreamListener - 消费者注解**：
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Component
public @interface StreamListener {
    // 必填参数
    String stream();
    String group();
    
    // 消费者配置
    String consumerName() default "";
    int concurrency() default 1;
    
    // 重试配置
    int maxRetries() default -1;
    long messageTimeout() default -1;
    Backoff backoff() default @Backoff();
    
    // 背压控制
    boolean enableBackpressure() default true;
    double backpressureThreshold() default -1.0;
    
    // 序列化
    SerializationType serialization() default SerializationType.JSON;
    
    // 错误处理
    String deadLetterStream() default "";
}
```

**@StreamProducer - 生产者注解**：
```java
@Target({ElementType.TYPE, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface StreamProducer {
    String stream() default "";
    SerializationType serialization() default SerializationType.JSON;
    boolean enableInterceptors() default true;
}
```

### 5.2 配置系统

**三级配置优先级**：
1. **注解级别**（最高）- 针对特定业务场景的细粒度配置
2. **全局配置**（中等）- application.yml 中的默认配置
3. **Starter 默认值**（最低）- 框架内置的合理默认值

**配置示例**：
```yaml
redis:
  stream:
    enabled: true
    backpressure:
      enabled: true
      threshold: 0.8
    retry:
      max-attempts: 5
      backoff:
        initial-interval: 1000
        multiplier: 2
        max-interval: 60000
    monitoring:
      enabled: true
      export-metrics: true
```


## 六、幂等性保证和可靠性设计（关键）

### 6.1 核心问题

**问题 1：XAUTOCLAIM 可能导致重复执行**

场景：
```
T0: 消费者 A 拉取任务 X，开始处理
T60: 任务 X 还在处理中（比如需要 90 秒才能完成）
T61: XAUTOCLAIM 定时任务执行，发现任务 X 超过 60 秒未 ACK
T61: XAUTOCLAIM 把任务 X 转移给消费者 B
     ⚠️ 现在消费者 A 和 B 都在处理同一个任务！
```

**问题 2：消费者宕机导致任务状态异常**

场景：
```
T0: 消费者 A 拉取任务 X
T1: 修改数据库状态为 ING，提交到线程池
T30: 消费者 A 宕机
     ├─ 任务 X 停止执行
     └─ 数据库状态仍然是 ING ⚠️
T61: XAUTOCLAIM 把任务 X 转移给消费者 B
     └─ 消费者 B 应该如何处理？
```

### 6.2 解决方案：分布式锁 + 正确的 ACK 位置

**关键设计原则**：
1. **分布式锁在业务逻辑执行期间持有**（不是在消息拉取时）
2. **ACK 在业务逻辑执行完成后执行**（不是在提交到线程池后）
3. **使用 Redisson 看门狗机制自动续期**

**错误的设计**（不要这样做）：
```java
// ❌ 错误：在 processMessage() 中持有锁和 ACK
private void processMessage(StreamMessageId messageId, Map<Object, Object> body) {
    RLock lock = redissonClient.getLock(lockKey);
    try {
        lock.tryLock();
        
        // 调用业务逻辑（异步提交到线程池，立即返回）
        consume(message, context);
        
        // ❌ 错误：任务还没执行完就 ACK 了
        ackMessage(messageId);
        
    } finally {
        // ❌ 错误：锁立即释放，无法保护任务执行过程
        lock.unlock();
    }
}
```

**正确的设计**（推荐）：
```java
// ✅ 正确：在业务逻辑中持有锁和 ACK
@StreamListener(stream = "stream:user_draw_tasks", group = "group:task_service")
public void handleTask(UserDrawTask task, MessageContext context) {
    RLock lock = null;
    try {
        // 1. 获取分布式锁
        String lockKey = "user_draw_lock_" + task.getUserDrawId();
        lock = redissonClient.getLock(lockKey);
        
        // 2. 尝试获取锁（使用看门狗自动续期）
        boolean acquired = lock.tryLock(0, TimeUnit.MILLISECONDS);
        
        if (!acquired) {
            log.warn("无法获取锁，任务可能正在被其他消费者处理");
            return;  // 不 ACK，等待下次重试
        }
        
        // 3. 获取到锁，说明：
        //    - 任务没有在执行，或
        //    - 之前的消费者宕机了（锁自动过期）
        
        // 4. 执行业务逻辑（看门狗自动续期）
        doDrawTask(task);
        
    } catch (Exception e) {
        log.error("任务执行失败", e);
    } finally {
        // 5. 释放分布式锁
        if (lock != null && lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
        
        // 6. ACK 消息（无论成功还是失败）
        context.ack();
    }
}
```

### 6.3 工作流程

**正常情况**：
```
消费者 A:
T0: 拉取消息 X
    └─ processMessage() 提交到线程池，立即返回（不 ACK）

T1: 业务逻辑开始执行
    ├─ 获取分布式锁 "user_draw_lock_123"
    ├─ 看门狗自动续期（每 10 秒）
    ├─ 执行业务逻辑（90 秒）
    └─ finally 块：
        ├─ 释放锁
        └─ ACK 消息 ✅
```

**消费者宕机场景**：
```
T0: 消费者 A 拉取消息 X
T1: 业务逻辑开始执行
    ├─ 获取分布式锁
    └─ 看门狗自动续期

T30: 消费者 A 宕机
     ├─ 看门狗停止续期
     └─ 锁在 30 秒后自动过期

T61: XAUTOCLAIM 把消息 X 转移给消费者 B
     └─ 消费者 B 的业务逻辑尝试获取锁 ✅ 成功
         └─ 继续执行任务
```

**任务执行时间长场景**：
```
T0: 消费者 A 拉取消息 X
T1: 业务逻辑开始执行
    ├─ 获取分布式锁
    └─ 看门狗自动续期（持续进行）

T61: XAUTOCLAIM 把消息 X 转移给消费者 B
     └─ 消费者 B 的业务逻辑尝试获取锁 ❌ 失败
         └─ 直接返回，不执行任务，不 ACK

T90: 消费者 A 完成任务
     ├─ 释放锁
     └─ ACK 消息 ✅
```

### 6.4 关键配置

```yaml
redis:
  stream:
    retry:
      # 消息超时时间应该 > 任务最大处理时间
      # 但由于使用看门狗机制，这个值主要用于 XAUTOCLAIM 的触发时间
      message-timeout: 120000  # 2 分钟
      
      # 是否使用看门狗机制（推荐）
      use-watchdog: true  # 自动续期，不需要担心任务执行时间过长
```


## 七、消息流转和数据流

### 7.1 完整的消息生命周期

```
1. 生产阶段
   业务代码调用
   ↓
   StreamTemplate.send(message)
   ├─ 序列化消息
   ├─ 添加元数据（timestamp, retryCount=0）
   ↓
   XADD 到 Redis Stream
   ↓
   stream:user_draw_tasks
   └─ messageId: 1234567890-0
      body: {userDrawId, userId, timestamp, retryCount}

2. 消费阶段
   StreamConsumerWrapper（后台线程持续运行）
   ↓
   背压控制检查
   ├─ if (线程池使用率 >= 80%) { sleep(1000); continue; }
   ↓
   XREADGROUP
   ├─ group: task_service
   ├─ consumer: consumer-xxx
   ├─ count: 1
   ├─ block: 5000ms
   ↓
   消息进入 PEL (Pending Entry List)
   ↓
   反序列化消息
   ↓
   调用业务方法（提交到线程池，立即返回）
   ├─ 不在这里 ACK！
   ↓
   业务逻辑执行（在线程池中）
   ├─ 获取分布式锁
   ├─ 看门狗自动续期
   ├─ 执行业务逻辑
   └─ finally 块：
       ├─ 释放锁
       └─ ACK 消息（从 PEL 移除）

3. 超时重试阶段
   AutoclaimScheduler（定时任务，每 30 秒执行）
   ↓
   XAUTOCLAIM
   ├─ 抢占超过 60 秒未 ACK 的消息
   ├─ 转移到当前消费者
   ↓
   更新 retryCount + 1
   ↓
   重新提交到业务处理（回到消费阶段）
```

### 7.2 数据结构设计

**消息体结构**：
```java
{
  "userDrawId": 123,
  "userId": 456,
  "timestamp": 1234567890000,
  "retryCount": 0,
  "messageType": "UserDrawTask",
  "payload": "{...}"  // 序列化后的业务数据（JSON）
}
```

**MessageContext 包含的信息**：
```java
{
  "messageId": "1234567890-0",
  "stream": "stream:user_draw_tasks",
  "group": "group:task_service",
  "consumerName": "consumer-xxx",
  "retryCount": 0,
  "timestamp": 1234567890000,
  "receiveTimestamp": 1234567890100,
  "rawBody": {...}
}
```


## 八、错误处理和重试机制

### 8.1 核心机制

1. **自动重试**：失败时不 ACK，依赖 XAUTOCLAIM 机制
2. **指数退避**：通过 `@Backoff` 注解配置退避策略
3. **死信队列**：超过最大重试次数后进入死信处理
4. **幂等性保证**：使用分布式锁确保同一消息不会被重复处理

### 8.2 重试流程

```
任务执行失败
↓
不 ACK 消息（消息留在 PEL 中）
↓
等待 XAUTOCLAIM 抢占（60 秒后）
↓
检查 retryCount
├─ retryCount < maxRetries
│   ├─ retryCount + 1
│   └─ 重新执行
└─ retryCount >= maxRetries
    └─ 进入死信处理
        ├─ 记录日志
        ├─ 发送告警（飞书/钉钉）
        ├─ 更新任务状态为 FAIL
        └─ 强制 ACK（避免继续占用 PEL）
```

### 8.3 死信处理

```java
public class DeadLetterHandler {
    public void handle(StreamMessageId messageId,
                      Map<Object, Object> body,
                      Exception error,
                      MessageContext context) {
        // 1. 记录死信日志
        log.error("消息重试次数耗尽，进入死信队列: messageId={}, retryCount={}", 
                 messageId, context.getRetryCount(), error);
        
        // 2. 发送告警通知
        sendAlert(messageId, body, error);
        
        // 3. 更新任务状态为 FAIL
        updateTaskStatus(body, "FAIL");
        
        // 4. 强制 ACK 防止继续占用 PEL
        ackMessage(messageId);
    }
}
```

## 九、监控和可观测性

### 9.1 监控指标（基于 Micrometer）

**核心指标**：
- Stream 长度（应 < 10000）
- PEL 长度（应 < 100）
- 消息处理延迟（P50, P95, P99）
- 消费者数量
- 成功/失败消息数
- 死信消息数
- 锁冲突次数
- 线程池使用率

**告警阈值**：
- Stream 长度 > 10000：警告级别
- PEL 长度 > 100：错误级别，发送飞书告警
- 消费者数量 = 0：严重级别
- 锁冲突次数频繁：说明超时时间设置过短

### 9.2 健康检查

```java
@Component
public class StreamHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        // 检查 Stream 是否存在
        // 检查 Consumer Group 是否存在
        // 检查消费者是否正常运行
        // 检查 PEL 长度是否正常
    }
}
```

## 十、测试策略

### 10.1 单元测试

- 序列化/反序列化测试
- 配置解析测试
- 重试逻辑测试
- 背压控制测试

### 10.2 集成测试

- 端到端消息流转测试
- 超时重试测试
- 死信处理测试
- 幂等性测试
- 消费者宕机恢复测试

### 10.3 性能测试

- 发送延迟（目标：< 10ms）
- 消费吞吐量
- 水平扩展效果
- 对比 RabbitMQ 方案


## 十一、使用示例

### 11.1 启动类配置

```java
@SpringBootApplication
@EnableRedisStream
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 11.2 配置文件

```yaml
redis:
  stream:
    enabled: true
    backpressure:
      enabled: true
      threshold: 0.8
    retry:
      max-attempts: 5
      use-watchdog: true  # 使用看门狗自动续期
      backoff:
        initial-interval: 1000
        multiplier: 2
        max-interval: 60000
    monitoring:
      enabled: true
      export-metrics: true
```

### 11.3 生产者使用

```java
@Service
public class UserDrawService {
    
    @Autowired
    @StreamProducer(stream = RedisStreamConfig.STREAM_USER_DRAW_TASKS)
    private StreamTemplate<UserDrawTask> streamTemplate;
    
    public void submitTask(Long userDrawId, Long userId) {
        UserDrawTask task = new UserDrawTask(userDrawId, userId);
        streamTemplate.send(task);
    }
}
```

### 11.4 消费者使用

```java
@Component
public class UserDrawConsumer {
    
    @Autowired
    private RedissonClient redissonClient;
    
    @StreamListener(
        stream = RedisStreamConfig.STREAM_USER_DRAW_TASKS,
        group = RedisStreamConfig.GROUP_TASK_SERVICE,
        maxRetries = 5,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public void handleTask(UserDrawTask task, MessageContext context) {
        RLock lock = null;
        try {
            // 获取分布式锁
            String lockKey = "user_draw_lock_" + task.getUserDrawId();
            lock = redissonClient.getLock(lockKey);
            
            // 尝试获取锁（使用看门狗自动续期）
            boolean acquired = lock.tryLock(0, TimeUnit.MILLISECONDS);
            
            if (!acquired) {
                log.warn("无法获取锁，任务可能正在被其他消费者处理");
                return;  // 不 ACK，等待下次重试
            }
            
            // 执行业务逻辑
            log.info("处理任务: userDrawId={}, retryCount={}", 
                    task.getUserDrawId(), context.getRetryCount());
            doDrawTask(task);
            
        } catch (Exception e) {
            log.error("任务执行失败", e);
            throw e;  // 抛出异常，不 ACK，等待重试
        } finally {
            // 释放锁
            if (lock != null && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
            
            // ACK 消息
            context.ack();
        }
    }
    
    private void doDrawTask(UserDrawTask task) {
        // 业务逻辑
    }
}
```

## 十二、实施计划概览

### 12.1 第一阶段：核心功能（1 周）

- 创建 Starter 模块结构
- 实现核心组件（StreamTemplate、注解、自动配置）
- 实现基础的生产者和消费者功能
- 实现消息序列化和反序列化

### 12.2 第二阶段：可靠性功能（3-5 天）

- 实现重试处理器和死信处理器
- 实现背压控制器
- 实现 XAUTOCLAIM 超时处理机制
- 完善配置系统（三级配置优先级）

### 12.3 第三阶段：监控和测试（2-3 天）

- 实现监控指标收集（Micrometer）
- 实现健康检查
- 编写单元测试和集成测试
- 编写使用文档和示例

### 12.4 第四阶段：迁移和验证（2-3 天）

- 迁移现有的 UserDrawStreamConsumer 到新框架
- 验证功能和性能
- 灰度发布
- 性能对比测试

## 十三、预期收益

### 13.1 性能提升

- **发送延迟**：从 ~100ms 降至 ~5ms（20x 提升）
- **CPU 使用率**：降低 ~30%（无轮询）
- **代码复杂度**：减少 ~40%（废弃 4 个定时任务 + 续期逻辑）

### 13.2 可维护性提升

- **定时任务**：从 4 个减少到 0 个
- **数据库表**：废弃 app_mq_message 表
- **代码行数**：减少 ~500 行
- **可复用性**：可在多个项目中使用

### 13.3 可靠性提升

- **消息丢失风险**：从"提前 ACK"降至"执行完成才 ACK"
- **补偿机制**：从"定时任务扫描"升级为"XAUTOCLAIM 自动抢占"
- **幂等性保证**：分布式锁 + 看门狗机制
- **监控指标**：从"DB 查询"简化为"Redis 命令"

## 十四、关键文件清单

### 14.1 需要新建的文件

**Starter 模块**：
- `redis-stream-spring-boot-starter/pom.xml`
- `RedisStreamAutoConfiguration.java`
- `RedisStreamProperties.java`
- `@EnableRedisStream.java`
- `@StreamListener.java`
- `@StreamProducer.java`
- `StreamTemplate.java`
- `AbstractStreamConsumer.java`
- `MessageContext.java`
- `BackpressureController.java`
- `RetryHandler.java`
- `DeadLetterHandler.java`
- `StreamMetricsCollector.java`
- `StreamHealthIndicator.java`

### 14.2 需要修改的文件

**迁移现有实现**：
- `goto/task/src/main/java/com/freedom/runner/UserDrawStreamConsumer.java` - 迁移到新框架
- `goto/task/src/main/java/com/freedom/util/DoUserDrawTask.java` - 简化逻辑
- `goto/task/src/main/java/com/freedom/thread/UserDrawRunnable.java` - 添加分布式锁和 ACK

## 十五、风险评估与缓解

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|----------|
| Redis 宕机导致消息丢失 | 高 | 低 | AOF 持久化已启用 |
| Stream 内存占用过高 | 中 | 中 | MAXLEN 限制 + 监控 |
| 消费者实例重启导致消息重复 | 低 | 高 | 幂等性设计（分布式锁） |
| XAUTOCLAIM 抢占失败 | 中 | 低 | 多实例部署 + 监控 PEL |
| 毒丸消息导致死信堆积 | 中 | 低 | 死信处理机制 + 最大重试 5 次 |
| 锁冲突频繁 | 中 | 中 | 合理配置超时时间 + 监控 |

## 十六、批准记录

- **设计批准日期**：待批准
- **批准人**：待填写
- **下一步**：创建详细实施计划

