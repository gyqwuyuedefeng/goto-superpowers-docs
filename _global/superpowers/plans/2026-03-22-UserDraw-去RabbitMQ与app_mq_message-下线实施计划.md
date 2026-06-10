# UserDraw 去 RabbitMQ 与 app_mq_message 下线 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 下线 `UserDraw` 的 RabbitMQ 与 `app_mq_message` 旧链路，新增独立取消 Redis Stream，保留现有提交 Redis Stream 和新的直接清理任务。

**Architecture:** 保持 `customDrawPlot()` 与 `stream:user_draw_tasks` 不变，新增一条独立的 `UserDraw` 取消 Redis Stream 生产/消费链路，然后分批删除 `app_mq_message`、用户绘图专属 RabbitMQ 基础设施、旧清理任务、旧监控与旧限流配置。删除过程以“小步提交 + 定向测试 + 全局 grep 校验”为主，防止误删 `LongTimeUnusedUserDrawRemoveTask` 与 `UserDrawRemoveExecutor`。

**Tech Stack:** Java 17, Spring Boot, Redis Stream starter, Redisson, RabbitMQ（仅保留非 UserDraw 链路）, JUnit 5, Mockito, Maven

---

## File Structure

### Common 模块

**Create:**
- `goto/common/src/main/java/com/freedom/model/UserDrawCancelStreamMessage.java`

**Modify:**
- `goto/common/src/main/java/com/freedom/config/RedisStreamConfig.java`
- `goto/common/src/main/java/com/freedom/bean/Repository.java`
- `goto/common/src/main/java/com/freedom/bean/ComponentBean.java`
- `goto/common/src/main/java/com/freedom/config/RabbitmqExchangeQueueConfig.java`
- `goto/common/src/main/java/com/freedom/constant/RabbitMQConstant.java`
- `goto/common/src/main/java/com/freedom/config/RabbitPublishType.java`
- `goto/common/src/main/java/com/freedom/config/common/ReturnCallback.java`

**Delete:**
- `goto/common/src/main/java/com/freedom/domain/MqMessage.java`
- `goto/common/src/main/java/com/freedom/repository/MqMessageRepository.java`
- `goto/common/src/main/java/com/freedom/constant/MqMessageEnum.java`
- `goto/common/src/main/java/com/freedom/constant/MqMessageResentEnum.java`
- `goto/common/src/main/java/com/freedom/model/UserDrawMQMessage.java`
- `goto/common/src/main/java/com/freedom/util/UserDrawTaskRabbitMQUtils.java`
- `goto/common/src/main/java/com/freedom/config/UserDrawRabbitTemplateConfig.java`
- `goto/common/src/main/java/com/freedom/config/userDraw/ConfirmReturnCallback.java`
- `goto/common/src/main/java/com/freedom/config/userDraw/MessageIdempotentHelper.java`

### System 模块

**Create:**
- `goto/system/src/test/java/com/freedom/service/impl/PlotServiceImplCancelStreamTest.java`

**Modify:**
- `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`
- `goto/system/src/main/java/com/freedom/rest/PlotController.java`
- `goto/system/src/main/java/com/freedom/runner/DispatchInitializer.java`
- `goto/system/src/main/java/com/freedom/monitor/SystemMetricsCollector.java`
- `goto/system/src/main/java/com/freedom/config/rabbitmq/listener/ListenerConfiguration.java`
- `goto/system/src/test/java/com/freedom/quartz/QuartzSuite.java`

**Delete:**
- `goto/system/src/main/java/com/freedom/quartz/task/LongTimeUserDrawMarkAndNotice.java`
- `goto/system/src/main/java/com/freedom/quartz/task/UserDrawTmpFolderRemoveFail.java`
- `goto/system/src/main/java/com/freedom/quartz/task/MessageResendScheduler.java`
- `goto/system/src/main/java/com/freedom/quartz/task/MqMessageScheduledScan.java`
- `goto/system/src/main/java/com/freedom/quartz/task/MqMessageArchiveTask.java`
- `goto/system/src/main/java/com/freedom/scheduled/MqMessageCleanupScheduler.java`
- `goto/system/src/main/java/com/freedom/bean/mq/rabbitmq/UserDrawMessageConsumer.java`
- `goto/system/src/test/java/com/freedom/quartz/task/LongTimeUserDrawMarkAndNoticeTest.java`
- `goto/system/src/test/java/com/freedom/quartz/task/UserDrawTmpFolderRemoveFailTest.java`

### Task 模块

**Create:**
- `goto/task/src/main/java/com/freedom/runner/UserDrawCancelStreamConsumer.java`
- `goto/task/src/test/java/com/freedom/runner/UserDrawCancelStreamConsumerTest.java`

**Modify:**
- `goto/task/src/main/java/com/freedom/util/DoUserDrawTask.java`
- `goto/task/src/main/java/com/freedom/thread/UserDrawRunnable.java`
- `goto/task/src/main/java/com/freedom/task/UserDrawStreamAutoClaimTask.java`
- `goto/task/src/main/java/com/freedom/config/rabbitmq/listener/ListenerConfiguration.java`
- `goto/task/src/test/java/com/freedom/util/DoUserDrawTaskTest.java`

**Delete:**
- `goto/task/src/main/java/com/freedom/bean/mq/rabbitmq/UserDrawTaskCancelConsumer.java`

### Must Keep

以下文件是明确保留项，任何任务都不允许删除：

- `goto/system/src/main/java/com/freedom/quartz/task/LongTimeUnusedUserDrawRemoveTask.java`
- `goto/system/src/main/java/com/freedom/quartz/task/UserDrawRemoveExecutor.java`
- `goto/system/src/test/java/com/freedom/quartz/task/LongTimeUnusedUserDrawRemoveTaskTest.java`
- `goto/system/src/test/java/com/freedom/quartz/task/UserDrawRemoveExecutorTest.java`

---

### Task 1: 添加取消流契约并改造服务层入口

**Files:**
- Create: `goto/common/src/main/java/com/freedom/model/UserDrawCancelStreamMessage.java`
- Create: `goto/system/src/test/java/com/freedom/service/impl/PlotServiceImplCancelStreamTest.java`
- Modify: `goto/common/src/main/java/com/freedom/config/RedisStreamConfig.java`
- Modify: `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`

- [ ] **Step 1: 写一个失败的服务层单测，锁定取消消息的 Redis Stream 生产行为**

在 `goto/system/src/test/java/com/freedom/service/impl/PlotServiceImplCancelStreamTest.java` 新建测试，最少包含：

```java
@ExtendWith(MockitoExtension.class)
class PlotServiceImplCancelStreamTest {

    @Mock
    private ComponentBean componentBean;

    @Mock
    private UserDrawTaskProperties userDrawTaskProperties;

    @Mock
    private StreamTemplate<UserDrawCancelStreamMessage> userDrawCancelProducer;

    private PlotServiceImpl service;

    @BeforeEach
    void setUp() {
        service = new PlotServiceImpl(componentBean, userDrawTaskProperties);
        ReflectionTestUtils.setField(service, "userDrawCancelProducer", userDrawCancelProducer);
    }

    @Test
    void sendToCancelRedisStream_shouldPublishCancelMessage() {
        when(userDrawCancelProducer.send(any(UserDrawCancelStreamMessage.class))).thenReturn("1-0");

        ReflectionTestUtils.invokeMethod(service, "sendToCancelRedisStream", 123L, 456L);

        ArgumentCaptor<UserDrawCancelStreamMessage> captor =
                ArgumentCaptor.forClass(UserDrawCancelStreamMessage.class);
        verify(userDrawCancelProducer).send(captor.capture());
        assertThat(captor.getValue().getUserDrawId()).isEqualTo(123L);
        assertThat(captor.getValue().getUserId()).isEqualTo(456L);
    }
}
```

- [ ] **Step 2: 运行单测，确认它因为缺少取消消息契约而失败**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common,system -am -DskipTests=false -Dsurefire.failIfNoSpecifiedTests=false -Dtest=PlotServiceImplCancelStreamTest test
```

Expected:

- 测试编译失败，报 `UserDrawCancelStreamMessage` 或 `sendToCancelRedisStream` 不存在

- [ ] **Step 3: 新增取消流消息模型和 Redis Stream 常量**

新增 `goto/common/src/main/java/com/freedom/model/UserDrawCancelStreamMessage.java`：

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class UserDrawCancelStreamMessage implements Serializable {
    private Long userDrawId;
    private Long userId;
}
```

在 `goto/common/src/main/java/com/freedom/config/RedisStreamConfig.java` 添加：

```java
public static final String STREAM_USER_DRAW_CANCEL_TASKS = "stream:user_draw_cancel_tasks";
public static final String GROUP_TASK_CANCEL_SERVICE = "group:task_cancel_service";
```

- [ ] **Step 4: 改造 `PlotServiceImpl` 的取消与批量测试入口**

在 `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`：

- 添加取消流 producer：

```java
@StreamProducer(
    stream = RedisStreamConfig.STREAM_USER_DRAW_CANCEL_TASKS,
    serialization = SerializationType.JSON
)
private StreamTemplate<UserDrawCancelStreamMessage> userDrawCancelProducer;
```

- 添加新的 helper：

```java
private void sendToCancelRedisStream(Long userDrawId, Long userId) {
    UserDrawCancelStreamMessage message = new UserDrawCancelStreamMessage(userDrawId, userId);
    String messageId = userDrawCancelProducer.send(message);
    log.info("取消消息发送到 Redis Stream 成功: userDrawId={}, userId={}, messageId={}",
            userDrawId, userId, messageId);
}
```

- 将 `customCancelDrawPlot()` 改为：
  - 保留参数校验
  - 保留 `RHandlerWrapper`
  - 用 `sendToCancelRedisStream(plotType.getUserDraw().getId(), SecurityUtils.getCurrentUserId())`
  - 去掉 `MqMessageRepository` 查询
  - 去掉 `RabbitPublishType.taskCancelUser*`
  - 去掉 `UserDrawTaskRabbitMQUtils`
  - 将 `doHandlerWithRetry(...)` 改为 `doHandler(...)`

- 将 `batchTestCustomDrawPlot()` 改为复用现有 `sendToRedisStream()`，彻底移除该方法中的 RabbitMQ / `MqMessage` 依赖

- [ ] **Step 5: 重新运行单测，并用 grep 确认 `PlotServiceImpl` 不再依赖旧消息表与用户绘图 RabbitMQ**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common,system -am -DskipTests=false -Dsurefire.failIfNoSpecifiedTests=false -Dtest=PlotServiceImplCancelStreamTest test
cd /mnt/f/IdeaProjects/goto-software
rg -n "MqMessage|UserDrawTaskRabbitMQUtils|RabbitPublishType|UserDrawMQMessage|getMqMessageRepository\\(" \
   goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java -S
```

Expected:

- `PlotServiceImplCancelStreamTest` PASS
- `rg` 无输出

- [ ] **Step 6: 编译 `common + system` 模块**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common,system -am -DskipTests compile
```

Expected:

- `BUILD SUCCESS`

- [ ] **Step 7: Commit**

```bash
git add goto/common/src/main/java/com/freedom/model/UserDrawCancelStreamMessage.java \
        goto/common/src/main/java/com/freedom/config/RedisStreamConfig.java \
        goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java \
        goto/system/src/test/java/com/freedom/service/impl/PlotServiceImplCancelStreamTest.java
git commit -m "feat: 添加UserDraw取消Redis Stream入口"
```

---

### Task 2: 添加取消流消费者并打通状态通知

**Files:**
- Create: `goto/task/src/main/java/com/freedom/runner/UserDrawCancelStreamConsumer.java`
- Create: `goto/task/src/test/java/com/freedom/runner/UserDrawCancelStreamConsumerTest.java`

- [ ] **Step 1: 写一个失败的取消消费者单测，覆盖幂等与通知场景**

在 `goto/task/src/test/java/com/freedom/runner/UserDrawCancelStreamConsumerTest.java` 新建测试，至少覆盖：

```java
@ExtendWith(MockitoExtension.class)
class UserDrawCancelStreamConsumerTest {

    @Mock
    private ComponentBean componentBean;

    @Mock
    private Repository repository;

    @Mock
    private UserDrawRepository userDrawRepository;

    @Mock
    private StreamTemplate<UserDrawNotification> userDrawNotificationProducer;

    @Mock
    private MessageContext messageContext;

    private UserDrawCancelStreamConsumer consumer;

    @BeforeEach
    void setUp() {
        when(componentBean.getRepository()).thenReturn(repository);
        when(repository.getUserDrawRepository()).thenReturn(userDrawRepository);
        when(componentBean.getUserDrawNotificationProducer()).thenReturn(userDrawNotificationProducer);
        SystemRebootInitUtils.userDrawThreadPool = mock(UserDrawThreadPool.class);
        consumer = new UserDrawCancelStreamConsumer(componentBean);
    }

    @Test
    void consume_shouldSkipWhenUserDrawMissing() { }

    @Test
    void consume_shouldSkipWhenUserDrawAlreadyTerminal() { }

    @Test
    void consume_shouldCancelAndSendNotificationWhenUserDrawActive() { }
}
```

- [ ] **Step 2: 运行单测，确认因为消费者不存在而失败**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common,task -am -DskipTests=false -Dsurefire.failIfNoSpecifiedTests=false -Dtest=UserDrawCancelStreamConsumerTest test
```

Expected:

- 编译失败，报 `UserDrawCancelStreamConsumer` 不存在

- [ ] **Step 3: 实现取消消费者**

新增 `goto/task/src/main/java/com/freedom/runner/UserDrawCancelStreamConsumer.java`，核心结构：

```java
@Slf4j
@Component
public class UserDrawCancelStreamConsumer {

    private final ComponentBean componentBean;

    public UserDrawCancelStreamConsumer(ComponentBean componentBean) {
        this.componentBean = componentBean;
    }

    @StreamListener(
            stream = RedisStreamConfig.STREAM_USER_DRAW_CANCEL_TASKS,
            group = RedisStreamConfig.GROUP_TASK_CANCEL_SERVICE,
            concurrency = 1,
            maxRetries = 3
    )
    public void consume(UserDrawCancelStreamMessage message, MessageContext context) {
        // 1. 查询 UserDraw
        // 2. 缺失时按幂等跳过
        // 3. SUCCESS / FAIL / CANCEL / REJECTED 时按幂等跳过
        // 4. 调用 SystemRebootInitUtils.userDrawThreadPool.cancel(...)
        // 5. cancel 返回 false 时，直接落库改为 CANCEL
        // 6. 发送 UserDrawNotification(status=CANCEL, noticeType=STATUS_CHANGE)
    }
}
```

实现时强制遵守：

- 不使用 RabbitMQ
- 不使用 `MqMessageRepository`
- 不覆盖已终态任务
- 取消成功后必须发送通知
- 通知失败时抛异常，让 Redis Stream 重试整条取消消息

- [ ] **Step 4: 运行取消消费者单测**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common,task -am -DskipTests=false -Dsurefire.failIfNoSpecifiedTests=false -Dtest=UserDrawCancelStreamConsumerTest test
```

Expected:

- `UserDrawCancelStreamConsumerTest` PASS

- [ ] **Step 5: 编译 `task` 模块**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl task -am -DskipTests compile
```

Expected:

- `BUILD SUCCESS`

- [ ] **Step 6: Commit**

```bash
git add goto/task/src/main/java/com/freedom/runner/UserDrawCancelStreamConsumer.java \
        goto/task/src/test/java/com/freedom/runner/UserDrawCancelStreamConsumerTest.java
git commit -m "feat: 添加UserDraw取消流消费者"
```

---

### Task 3: 删除 `app_mq_message` 领域模型并清理任务侧残留引用

**Files:**
- Modify: `goto/common/src/main/java/com/freedom/bean/Repository.java`
- Modify: `goto/task/src/main/java/com/freedom/util/DoUserDrawTask.java`
- Modify: `goto/task/src/main/java/com/freedom/thread/UserDrawRunnable.java`
- Modify: `goto/task/src/main/java/com/freedom/task/UserDrawStreamAutoClaimTask.java`
- Modify: `goto/task/src/test/java/com/freedom/util/DoUserDrawTaskTest.java`
- Modify: `goto/system/src/main/java/com/freedom/monitor/SystemMetricsCollector.java`
- Delete: `goto/common/src/main/java/com/freedom/domain/MqMessage.java`
- Delete: `goto/common/src/main/java/com/freedom/repository/MqMessageRepository.java`
- Delete: `goto/common/src/main/java/com/freedom/constant/MqMessageEnum.java`
- Delete: `goto/common/src/main/java/com/freedom/constant/MqMessageResentEnum.java`
- Delete: `goto/system/src/main/java/com/freedom/quartz/task/MessageResendScheduler.java`
- Delete: `goto/system/src/main/java/com/freedom/quartz/task/MqMessageScheduledScan.java`
- Delete: `goto/system/src/main/java/com/freedom/quartz/task/MqMessageArchiveTask.java`
- Delete: `goto/system/src/main/java/com/freedom/scheduled/MqMessageCleanupScheduler.java`

- [ ] **Step 1: 先修正 `DoUserDrawTask` 的失败通知测试，使它准确表达保留行为**

在 `goto/task/src/test/java/com/freedom/util/DoUserDrawTaskTest.java` 调整断言：

- `title` 应为固定值 `"提交失败"`
- 详细错误应断言在 `notification.getMsg()`
- `status` 应为 `FAIL`

参考断言：

```java
assertEquals("提交失败", notification.getTitle());
assertEquals("测试错误消息", notification.getMsg());
assertEquals(String.valueOf(CommonSerializableAnalysisStatus.FAIL.getType()), notification.getStatus());
```

- [ ] **Step 2: 运行测试，确认它先失败**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common,task -am -DskipTests=false -Dsurefire.failIfNoSpecifiedTests=false -Dtest=DoUserDrawTaskTest test
```

Expected:

- 失败于旧断言与现有实现不一致

- [ ] **Step 3: 删除 `MqMessage` 领域、仓储、枚举和所有依赖它们的定时任务**

按文件删除：

- `goto/common/src/main/java/com/freedom/domain/MqMessage.java`
- `goto/common/src/main/java/com/freedom/repository/MqMessageRepository.java`
- `goto/common/src/main/java/com/freedom/constant/MqMessageEnum.java`
- `goto/common/src/main/java/com/freedom/constant/MqMessageResentEnum.java`
- `goto/system/src/main/java/com/freedom/quartz/task/MessageResendScheduler.java`
- `goto/system/src/main/java/com/freedom/quartz/task/MqMessageScheduledScan.java`
- `goto/system/src/main/java/com/freedom/quartz/task/MqMessageArchiveTask.java`
- `goto/system/src/main/java/com/freedom/scheduled/MqMessageCleanupScheduler.java`

并在 `goto/common/src/main/java/com/freedom/bean/Repository.java` 删除：

```java
private final MqMessageRepository mqMessageRepository;
```

- [ ] **Step 4: 将 `DoUserDrawTask` 收缩为仅保留 Redis Stream 通知相关能力**

在 `goto/task/src/main/java/com/freedom/util/DoUserDrawTask.java`：

- 删除 `process(...)`
- 删除 `recover(...)`
- 删除全部 `MqMessage` / `UserDrawMQMessage` / `RabbitMQ` / `Channel` / `Message` 相关 import
- 保留 `noticeFail(...)`

保留后的类最少应接近：

```java
@Slf4j
public class DoUserDrawTask {

    public static void noticeFail(ComponentBean componentBean, UserDraw userDraw, String msg) {
        UserDrawCommonUtils.changeStatus(componentBean, CommonSerializableAnalysisStatus.FAIL.getType(), userDraw, true);
        UserDrawNotification notification = UserDrawNotification.from(
                userDraw,
                String.valueOf(CommonAnalysisNoticeType.SUBMIT_FAIL.getType()),
                String.valueOf(MessageType.ERROR),
                "提交失败",
                msg
        );
        componentBean.getUserDrawNotificationProducer().send(notification);
    }
}
```

同时清理：

- `goto/task/src/main/java/com/freedom/thread/UserDrawRunnable.java` 中未使用的 `DoUserDrawTask` import
- `goto/task/src/main/java/com/freedom/task/UserDrawStreamAutoClaimTask.java` 中未使用的 `DoUserDrawTask` import
- `goto/system/src/main/java/com/freedom/monitor/SystemMetricsCollector.java` 中 `app_mq_message` 指标采集方法

- [ ] **Step 5: 重新运行 `DoUserDrawTask` 测试，并用 grep 确认活跃代码中已无 `app_mq_message` 访问**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common,task -am -DskipTests=false -Dsurefire.failIfNoSpecifiedTests=false -Dtest=DoUserDrawTaskTest test
cd /mnt/f/IdeaProjects/goto-software
rg -n "app_mq_message|getMqMessageRepository\\(|MqMessage\\b|MqMessageEnum|MqMessageResentEnum" \
   goto/common/src/main/java goto/system/src/main/java goto/task/src/main/java \
   -g '!**/DoUserDrawTaskBackup*.java' -S
```

Expected:

- `DoUserDrawTaskTest` PASS
- `rg` 无输出

- [ ] **Step 6: 编译所有三大模块**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common,system,task -am -DskipTests compile
```

Expected:

- `BUILD SUCCESS`

- [ ] **Step 7: Commit**

```bash
git add goto/common/src/main/java/com/freedom/bean/Repository.java \
        goto/task/src/main/java/com/freedom/util/DoUserDrawTask.java \
        goto/task/src/main/java/com/freedom/thread/UserDrawRunnable.java \
        goto/task/src/main/java/com/freedom/task/UserDrawStreamAutoClaimTask.java \
        goto/task/src/test/java/com/freedom/util/DoUserDrawTaskTest.java \
        goto/system/src/main/java/com/freedom/monitor/SystemMetricsCollector.java
git rm goto/common/src/main/java/com/freedom/domain/MqMessage.java \
       goto/common/src/main/java/com/freedom/repository/MqMessageRepository.java \
       goto/common/src/main/java/com/freedom/constant/MqMessageEnum.java \
       goto/common/src/main/java/com/freedom/constant/MqMessageResentEnum.java \
       goto/system/src/main/java/com/freedom/quartz/task/MessageResendScheduler.java \
       goto/system/src/main/java/com/freedom/quartz/task/MqMessageScheduledScan.java \
       goto/system/src/main/java/com/freedom/quartz/task/MqMessageArchiveTask.java \
       goto/system/src/main/java/com/freedom/scheduled/MqMessageCleanupScheduler.java
git commit -m "refactor: 清理app_mq_message旧链路"
```

---

### Task 4: 删除 UserDraw 专属 RabbitMQ 基础设施和旧队列限流

**Files:**
- Modify: `goto/common/src/main/java/com/freedom/bean/ComponentBean.java`
- Modify: `goto/common/src/main/java/com/freedom/config/RabbitmqExchangeQueueConfig.java`
- Modify: `goto/common/src/main/java/com/freedom/constant/RabbitMQConstant.java`
- Modify: `goto/common/src/main/java/com/freedom/config/RabbitPublishType.java`
- Modify: `goto/common/src/main/java/com/freedom/config/common/ReturnCallback.java`
- Modify: `goto/task/src/main/java/com/freedom/config/rabbitmq/listener/ListenerConfiguration.java`
- Modify: `goto/system/src/main/java/com/freedom/config/rabbitmq/listener/ListenerConfiguration.java`
- Modify: `goto/system/src/main/java/com/freedom/rest/PlotController.java`
- Delete: `goto/common/src/main/java/com/freedom/model/UserDrawMQMessage.java`
- Delete: `goto/common/src/main/java/com/freedom/util/UserDrawTaskRabbitMQUtils.java`
- Delete: `goto/common/src/main/java/com/freedom/config/UserDrawRabbitTemplateConfig.java`
- Delete: `goto/common/src/main/java/com/freedom/config/userDraw/ConfirmReturnCallback.java`
- Delete: `goto/common/src/main/java/com/freedom/config/userDraw/MessageIdempotentHelper.java`
- Delete: `goto/task/src/main/java/com/freedom/bean/mq/rabbitmq/UserDrawTaskCancelConsumer.java`
- Delete: `goto/system/src/main/java/com/freedom/bean/mq/rabbitmq/UserDrawMessageConsumer.java`

- [ ] **Step 1: 删除控制器上的旧队列长度限流**

在 `goto/system/src/main/java/com/freedom/rest/PlotController.java`：

- 删除 `RabbitMQConstant` import
- 删除 `customDrawPlot()` 上的 `@TotalLimit(...)`
- 删除 `batchTestCustomDrawPlot()` 上的 `@TotalLimit(...)`
- 保留现有 `@RateLimit` 与 `@Debounce`

- [ ] **Step 2: 删除用户绘图专属 RabbitMQ 常量、交换机、监听工厂和消费者**

具体要求：

- `goto/common/src/main/java/com/freedom/constant/RabbitMQConstant.java`
  - 删除 `EXCHANGE_USER_DRAW_TASK_NOTICE`
  - 删除 `QUEUE_USER_DRAW_TASK_NOTICE`
  - 删除 `EXCHANGE_USER_DRAW_TASK_CANCEL`
  - 删除 `QUEUE_USER_DRAW_TASK_CANCEL`
  - 删除 `EXCHANGE_USER_DRAW_MESSAGE_NOTICE`
  - 删除 `QUEUE_USER_DRAW_MESSAGE_NOTICE`

- `goto/common/src/main/java/com/freedom/config/RabbitmqExchangeQueueConfig.java`
  - 删除所有 `userDrawTaskNotice*`
  - 删除所有 `userDrawTaskCancel*`
  - 删除所有 `userDrawMessageNotice*`

- `goto/task/src/main/java/com/freedom/config/rabbitmq/listener/ListenerConfiguration.java`
  - 删除 `userDrawTaskListenerContainerFactory`
  - 删除 `userDrawTaskCancelListenerContainerFactory`

- `goto/system/src/main/java/com/freedom/config/rabbitmq/listener/ListenerConfiguration.java`
  - 删除 `userDrawMessageListenerContainerFactory`

- 删除以下类：
  - `UserDrawTaskCancelConsumer`
  - `UserDrawMessageConsumer`
  - `UserDrawMQMessage`
  - `UserDrawTaskRabbitMQUtils`
  - `UserDrawRabbitTemplateConfig`
  - `ConfirmReturnCallback`
  - `MessageIdempotentHelper`

- [ ] **Step 3: 去掉共享组件中的用户绘图 RabbitMQ 依赖**

在 `goto/common/src/main/java/com/freedom/bean/ComponentBean.java` 删除：

```java
@Qualifier(value = "userDrawRabbitTemplate")
private RabbitTemplate userDrawRabbitTemplate;
```

在 `goto/common/src/main/java/com/freedom/config/RabbitPublishType.java` 只保留通用或示例数据仍在使用的常量/方法，删除：

- `TASK_SUBMIT_USER`
- `TASK_CANCEL_USER`
- `taskSubUser(...)`
- `taskSubUserPrefix(...)`
- `taskCancelUser(...)`
- `taskCancelUserPrefix(...)`

在 `goto/common/src/main/java/com/freedom/config/common/ReturnCallback.java` 将消息反序列化来源改为 `componentBean.getCommonRabbitTemplate()`，不再触碰已删除的 `userDrawRabbitTemplate`。

- [ ] **Step 4: 编译并全局 grep 校验用户绘图 RabbitMQ 标识已清空**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common,system,task -am -DskipTests compile
cd /mnt/f/IdeaProjects/goto-software
rg -n "userDrawRabbitTemplate|QUEUE_USER_DRAW_TASK_NOTICE|EXCHANGE_USER_DRAW_TASK_NOTICE|QUEUE_USER_DRAW_TASK_CANCEL|EXCHANGE_USER_DRAW_TASK_CANCEL|QUEUE_USER_DRAW_MESSAGE_NOTICE|EXCHANGE_USER_DRAW_MESSAGE_NOTICE|UserDrawMQMessage|UserDrawTaskRabbitMQUtils|ConfirmReturnCallback|MessageIdempotentHelper" \
   goto/common/src/main/java goto/system/src/main/java goto/task/src/main/java -S
```

Expected:

- `BUILD SUCCESS`
- `rg` 无输出

- [ ] **Step 5: Commit**

```bash
git add goto/common/src/main/java/com/freedom/bean/ComponentBean.java \
        goto/common/src/main/java/com/freedom/config/RabbitmqExchangeQueueConfig.java \
        goto/common/src/main/java/com/freedom/constant/RabbitMQConstant.java \
        goto/common/src/main/java/com/freedom/config/RabbitPublishType.java \
        goto/common/src/main/java/com/freedom/config/common/ReturnCallback.java \
        goto/task/src/main/java/com/freedom/config/rabbitmq/listener/ListenerConfiguration.java \
        goto/system/src/main/java/com/freedom/config/rabbitmq/listener/ListenerConfiguration.java \
        goto/system/src/main/java/com/freedom/rest/PlotController.java
git rm goto/common/src/main/java/com/freedom/model/UserDrawMQMessage.java \
       goto/common/src/main/java/com/freedom/util/UserDrawTaskRabbitMQUtils.java \
       goto/common/src/main/java/com/freedom/config/UserDrawRabbitTemplateConfig.java \
       goto/common/src/main/java/com/freedom/config/userDraw/ConfirmReturnCallback.java \
       goto/common/src/main/java/com/freedom/config/userDraw/MessageIdempotentHelper.java \
       goto/task/src/main/java/com/freedom/bean/mq/rabbitmq/UserDrawTaskCancelConsumer.java \
       goto/system/src/main/java/com/freedom/bean/mq/rabbitmq/UserDrawMessageConsumer.java
git commit -m "refactor: 删除UserDraw旧RabbitMQ基础设施"
```

---

### Task 5: 删除旧清理链路并确认新直接删除链路仍然保留

**Files:**
- Modify: `goto/system/src/main/java/com/freedom/runner/DispatchInitializer.java`
- Modify: `goto/system/src/test/java/com/freedom/quartz/QuartzSuite.java`
- Delete: `goto/system/src/main/java/com/freedom/quartz/task/LongTimeUserDrawMarkAndNotice.java`
- Delete: `goto/system/src/main/java/com/freedom/quartz/task/UserDrawTmpFolderRemoveFail.java`
- Delete: `goto/system/src/test/java/com/freedom/quartz/task/LongTimeUserDrawMarkAndNoticeTest.java`
- Delete: `goto/system/src/test/java/com/freedom/quartz/task/UserDrawTmpFolderRemoveFailTest.java`

- [ ] **Step 1: 先更新 suite 和旧测试文件，避免删除后测试工程直接挂掉**

在 `goto/system/src/test/java/com/freedom/quartz/QuartzSuite.java`：

- 删除 `LongTimeUserDrawMarkAndNoticeTest` import
- 将 suite 指向保留项：

```java
@SelectClasses({
        LongTimeUnusedUserDrawRemoveTaskTest.class
})
```

删除：

- `goto/system/src/test/java/com/freedom/quartz/task/LongTimeUserDrawMarkAndNoticeTest.java`
- `goto/system/src/test/java/com/freedom/quartz/task/UserDrawTmpFolderRemoveFailTest.java`

- [ ] **Step 2: 删除旧清理任务并移除启动补旧延迟任务逻辑**

删除：

- `goto/system/src/main/java/com/freedom/quartz/task/LongTimeUserDrawMarkAndNotice.java`
- `goto/system/src/main/java/com/freedom/quartz/task/UserDrawTmpFolderRemoveFail.java`

修改 `goto/system/src/main/java/com/freedom/runner/DispatchInitializer.java`：

- 删除 `LongTimeUserDrawMarkAndNotice` import
- 删除构造注入字段
- 删除 `longTimeUserDrawMarkAndNotice.addDelayedTaskIfAbsent();`

**不要修改以下保留文件：**

- `goto/system/src/main/java/com/freedom/quartz/task/LongTimeUnusedUserDrawRemoveTask.java`
- `goto/system/src/main/java/com/freedom/quartz/task/UserDrawRemoveExecutor.java`

- [ ] **Step 3: 运行保留链路的测试，确保没有误删**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common,system -am -DskipTests=false -Dsurefire.failIfNoSpecifiedTests=false -Dtest=LongTimeUnusedUserDrawRemoveTaskTest,UserDrawRemoveExecutorTest,QuartzSuite test
```

Expected:

- 三个测试 PASS

- [ ] **Step 4: 用 grep 做“保留项 / 删除项”双向确认**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
rg -n "LongTimeUserDrawMarkAndNotice|UserDrawTmpFolderRemoveFail" goto/system/src/main/java goto/system/src/test/java -S
rg -n "LongTimeUnusedUserDrawRemoveTask|UserDrawRemoveExecutor" goto/system/src/main/java goto/system/src/test/java -S
```

Expected:

- 第一个 `rg` 无输出
- 第二个 `rg` 仍然命中保留类与测试

- [ ] **Step 5: Commit**

```bash
git add goto/system/src/main/java/com/freedom/runner/DispatchInitializer.java \
        goto/system/src/test/java/com/freedom/quartz/QuartzSuite.java
git rm goto/system/src/main/java/com/freedom/quartz/task/LongTimeUserDrawMarkAndNotice.java \
       goto/system/src/main/java/com/freedom/quartz/task/UserDrawTmpFolderRemoveFail.java \
       goto/system/src/test/java/com/freedom/quartz/task/LongTimeUserDrawMarkAndNoticeTest.java \
       goto/system/src/test/java/com/freedom/quartz/task/UserDrawTmpFolderRemoveFailTest.java
git commit -m "refactor: 删除旧UserDraw清理链路"
```

---

### Task 6: 做最终编译、测试与残留符号核对

**Files:**
- Verify only, unless this task暴露出最后的小修复

- [ ] **Step 1: 运行系统与任务模块的关键测试集**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common,system -am -DskipTests=false -Dsurefire.failIfNoSpecifiedTests=false -Dtest=PlotServiceImplCancelStreamTest,LongTimeUnusedUserDrawRemoveTaskTest,UserDrawRemoveExecutorTest,UserDrawNotificationStreamConsumerTest,QuartzSuite test
mvn -pl common,task -am -DskipTests=false -Dsurefire.failIfNoSpecifiedTests=false -Dtest=UserDrawCancelStreamConsumerTest,DoUserDrawTaskTest test
```

Expected:

- 所有指定测试 PASS

- [ ] **Step 2: 做全工程编译检查**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common,system,task -am -DskipTests compile
```

Expected:

- `BUILD SUCCESS`

- [ ] **Step 3: 对主代码做残留符号 grep**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
rg -n "app_mq_message|getMqMessageRepository\\(|userDrawRabbitTemplate|QUEUE_USER_DRAW_TASK_NOTICE|EXCHANGE_USER_DRAW_TASK_NOTICE|QUEUE_USER_DRAW_TASK_CANCEL|EXCHANGE_USER_DRAW_TASK_CANCEL|QUEUE_USER_DRAW_MESSAGE_NOTICE|EXCHANGE_USER_DRAW_MESSAGE_NOTICE|LongTimeUserDrawMarkAndNotice|UserDrawTmpFolderRemoveFail" \
   goto/common/src/main/java goto/system/src/main/java goto/task/src/main/java -g '!**/DoUserDrawTaskBackup*.java' -S
```

Expected:

- 无输出

- [ ] **Step 4: 对保留项再做一次 grep**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
rg -n "LongTimeUnusedUserDrawRemoveTask|UserDrawRemoveExecutor" \
   goto/system/src/main/java goto/system/src/test/java -S
```

Expected:

- 命中保留类与对应测试

- [ ] **Step 5: 查看改动范围**

Run:

```bash
git status --short
git diff --stat
```

Expected:

- 改动集中在 spec / plan 指定的文件范围
- 没有误触与 `UserDraw` 下线无关的模块

- [ ] **Step 6: 如果运行环境支持 GitNexus MCP，做一次变更范围核对**

Run:

```text
gitnexus_detect_changes({scope: "all"})
```

Expected:

- 影响范围主要集中在 UserDraw 提交/取消、旧 RabbitMQ 配置、旧清理任务、监控与测试

如果当前会话没有 GitNexus MCP，可记录“本步跳过，原因是 MCP 不可用”。

---

## Notes For Implementers

- `LongTimeUnusedUserDrawRemoveTask` 和 `UserDrawRemoveExecutor` 是保留项，本计划的任何步骤都不能删除它们。
- `DoUserDrawTaskBackup.java` / `DoUserDrawTaskBackup1.java` 是注释备份文件，不纳入主代码 grep 验收范围；除非编译或规范要求，不要顺手改动。
- `system` 与 `task` 模块的 `pom.xml` 默认 `skipTests=true`，运行测试时必须显式加 `-DskipTests=false`。
- `RabbitMQConstant` 与 `RabbitmqExchangeQueueConfig` 里仍有通用任务和示例数据任务常量，不要把非 `UserDraw` 路径一起删掉。
