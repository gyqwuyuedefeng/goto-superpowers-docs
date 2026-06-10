# Redis Stream sendRedisStreamNotification 补全实施计划

> **For agentic workers:** REQUIRED: Use superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 补全 CommonStage 中 Redis Stream 个性化通知的完整实现，使其与 RabbitMQ 通知路径对称。

**Architecture:** IThreadPool 加 STREAM_PRODUCER_FACTORY_TTL；NoticeRelationData 加 streamKey；ThreadPool.ttlSet()/ttlRemove() 同步维护；CommonStage.sendRedisStreamNotification() 提供默认实现；SystemRebootInitUtils 注入 streamProducerFactory 和 streamKey。

**Tech Stack:** Java, Spring Boot, Redisson, redis-stream-spring-boot-starter (StreamProducerFactory/StreamTemplate), TransmittableThreadLocal

---

## Chunk 1: 基础配置层

### Task 1: IThreadPool 加 STREAM_PRODUCER_FACTORY_TTL

**Files:**
- Modify: `freedom-public/common-thread-r/src/main/java/com/freedom/thread/IThreadPool.java`

- [ ] **Step 1: 加 import 和 TTL 字段**

在 `RABBIT_TEMPLATE_TTL` 下方加：

```java
import com.freedom.stream.producer.StreamProducerFactory;

public final static TransmittableThreadLocal<StreamProducerFactory> STREAM_PRODUCER_FACTORY_TTL = new TransmittableThreadLocal<>();
```

- [ ] **Step 2: 确认编译**

```bash
cd freedom-public/common-thread-r && mvn compile -q
```

---

### Task 2: NoticeRelationData 加 streamKey

**Files:**
- Modify: `freedom-public/common-thread-r/src/main/java/com/freedom/model/NoticeRelationData.java`

- [ ] **Step 1: 加 streamKey 字段**（在 `routingKey` 下方）

```java
/**
 * Redis Stream key（对应 RabbitMQ 的 exchange+routingKey）
 */
public String streamKey;
```

- [ ] **Step 2: 新增 7 参构造函数**（保留原 6 参版本不动）

```java
public NoticeRelationData(Integer noticeFormType, Integer noticeType, Integer personalizationNoticeType,
                          String exchange, String routingKey, String streamKey,
                          ThreadPoolWebSocket threadPoolWebSocket) {
    this.noticeFormType = noticeFormType;
    this.noticeType = noticeType;
    this.personalizationNoticeType = personalizationNoticeType;
    this.exchange = exchange;
    this.routingKey = routingKey;
    this.streamKey = streamKey;
    this.threadPoolWebSocket = threadPoolWebSocket;
}
```

- [ ] **Step 3: 确认编译**

```bash
cd freedom-public/common-thread-r && mvn compile -q
```

- [ ] **Step 4: Commit**

```bash
git add freedom-public/common-thread-r/src/main/java/com/freedom/thread/IThreadPool.java \
        freedom-public/common-thread-r/src/main/java/com/freedom/model/NoticeRelationData.java
git commit -m "feat: 添加 Redis Stream TTL 和 streamKey 配置支持"
```

---

## Chunk 2: ThreadPool 注入层

### Task 3: ThreadPool 加 streamProducerFactory 并维护 TTL

**Files:**
- Modify: `freedom-public/common-thread/pom.xml`
- Modify: `freedom-public/common-thread/src/main/java/com/freedom/thread/ThreadPool.java`

- [ ] **Step 1: pom.xml 加依赖**

```xml
<dependency>
    <groupId>com.freedom</groupId>
    <artifactId>redis-stream-spring-boot-starter</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <optional>true</optional>
</dependency>
```

- [ ] **Step 2: 加 import 和字段**（在 `rabbitTemplate` 下方）

```java
import com.freedom.stream.producer.StreamProducerFactory;

public StreamProducerFactory streamProducerFactory;
```

- [ ] **Step 3: ttlSet() 加一行**（在 `RABBIT_TEMPLATE_TTL.set` 下方）

```java
STREAM_PRODUCER_FACTORY_TTL.set(this.streamProducerFactory);
```

- [ ] **Step 4: ttlRemove() 加一行**（在 `RABBIT_TEMPLATE_TTL.remove()` 下方）

```java
STREAM_PRODUCER_FACTORY_TTL.remove();
```

- [ ] **Step 5: 确认编译**

```bash
cd freedom-public/common-thread && mvn compile -q
```

- [ ] **Step 6: Commit**

```bash
git add freedom-public/common-thread/pom.xml \
        freedom-public/common-thread/src/main/java/com/freedom/thread/ThreadPool.java
git commit -m "feat: ThreadPool 注入 StreamProducerFactory 到 TTL"
```

---

## Chunk 3: CommonStage 实现层

### Task 4: 实现 sendRedisStreamNotification 默认逻辑

**Files:**
- Modify: `freedom-public/common-thread-r/src/main/java/com/freedom/model/stage/CommonStage.java`

- [ ] **Step 1: 加 import**

```java
import com.freedom.stream.producer.StreamProducerFactory;
import com.freedom.stream.core.StreamTemplate;
import com.freedom.stream.annotation.SerializationType;
```

- [ ] **Step 2: 替换 sendRedisStreamNotification**

将原来的 warn 空实现整体替换为：

```java
protected void sendRedisStreamNotification(MessageContainer<T> messageContainer, BasicMessage basicMessage) {
    StreamProducerFactory factory = IThreadPool.STREAM_PRODUCER_FACTORY_TTL.get();
    String streamKey = IThreadPool.NOTICE_RELATION_DATA_TTL.get().getStreamKey();
    if (factory == null || streamKey == null) {
        log.warn("Redis Stream 通知配置缺失：factory={}, streamKey={}", factory != null ? "ok" : "null", streamKey);
        return;
    }
    try {
        StreamTemplate<MessageContainer> template = factory.createProducer(streamKey, SerializationType.JSON);
        template.send(messageContainer);
    } catch (Exception e) {
        log.error("Redis Stream 通知发送失败，streamKey={}", streamKey, e);
    }
}
```

- [ ] **Step 3: 确认编译**

```bash
cd freedom-public/common-thread-r && mvn compile -q
```

- [ ] **Step 4: Commit**

```bash
git add freedom-public/common-thread-r/src/main/java/com/freedom/model/stage/CommonStage.java
git commit -m "feat: 实现 Redis Stream 个性化通知默认逻辑"
```

---

## Chunk 4: 调用方接入层

### Task 5: SystemRebootInitUtils 注入 streamProducerFactory 和 streamKey

**Files:**
- Modify: `goto/task/src/main/java/com/freedom/util/SystemRebootInitUtils.java`

> 当前 `userDrawThreadPool` 已配置 `NoticeType.REDIS_STREAM`，但 `streamKey=null`，`streamProducerFactory` 未注入。

- [ ] **Step 1: 注入 StreamProducerFactory**

在类字段中加（`@AllArgsConstructor` 会自动注入）：

```java
private final StreamProducerFactory streamProducerFactory;
```

- [ ] **Step 2: userDrawThreadPool 设置 streamProducerFactory**

在 `userDrawThreadPool.noticeRelationData = ...` 之后加：

```java
userDrawThreadPool.streamProducerFactory = streamProducerFactory;
```

- [ ] **Step 3: 修正 userDrawThreadPool 的 NoticeRelationData**

将原来的 6 参构造改为 7 参，补上 streamKey：

```java
userDrawThreadPool.noticeRelationData = new NoticeRelationData(
    NoticeFormType.PERSONALIZATION.getType(),
    NoticeType.REDIS_STREAM.getType(),
    NoticeType.REDIS_STREAM.getType(),
    null,
    null,
    RedisStreamConfig.STREAM_USER_DRAW_NOTIFICATION,  // streamKey
    null
);
```

- [ ] **Step 4: 确认编译**

```bash
cd goto/task && mvn compile -q
```

- [ ] **Step 5: Commit**

```bash
git add goto/task/src/main/java/com/freedom/util/SystemRebootInitUtils.java
git commit -m "feat: UserDrawThreadPool 接入 Redis Stream 通知，注入 streamProducerFactory 和 streamKey"
```

---

## 验证清单

- [ ] `userDrawThreadPool` 提交任务后，Redis Stream `stream:user_draw_notification` 中能收到消息
- [ ] `factory=null` 或 `streamKey=null` 时打印 warn 日志，不抛异常
- [ ] 其他使用 RabbitMQ 的线程池（generateSampleDataThreadPool、commonTaskThreadPool）行为不受影响
