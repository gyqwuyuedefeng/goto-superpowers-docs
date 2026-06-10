# PlotServiceImpl Redis Stream 重构实施计划

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**目标**: 将 PlotServiceImpl 从 Redisson 原生 API 迁移到 @StreamProducer 注解方式，移除 RabbitMQ 相关代码

**架构**: 使用 @StreamProducer 注解注入 StreamTemplate，简化消息发送逻辑，更新消费者配置与线程池对齐

**技术栈**: Spring Boot, Redis Stream, Redisson, @StreamProducer 注解

---

## Chunk 1: 生产者重构

### 文件结构

**修改文件**:
- `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`
  - 添加 @StreamProducer 字段注入
  - 重构 sendToRedisStream() 方法
  - 简化 customDrawPlot() 方法

**依赖文件**（只读）:
- `goto/common/src/main/java/com/freedom/config/RedisStreamConfig.java` - Stream 配置常量
- `goto/task/src/main/java/com/freedom/model/UserDrawStreamMessage.java` - 消息类
- `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/annotation/StreamProducer.java` - 注解定义
- `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/core/StreamTemplate.java` - 模板类

---

### Task 1: 验证依赖配置

**文件**: `goto/system/pom.xml`

- [ ] **Step 1: 检查 redis-stream-spring-boot-starter 依赖**

```bash
grep -A 5 "redis-stream-spring-boot-starter" goto/system/pom.xml
```

预期：找到依赖配置，类似：
```xml
<dependency>
    <groupId>com.freedom</groupId>
    <artifactId>redis-stream-spring-boot-starter</artifactId>
</dependency>
```

如果没有找到，需要先添加依赖。

- [ ] **Step 2: 验证 UserDrawStreamMessage 类存在**

```bash
cat goto/task/src/main/java/com/freedom/model/UserDrawStreamMessage.java
```

预期：文件存在，包含 userDrawId 和 userId 字段

---

### Task 2: 添加 @StreamProducer 字段注入

**文件**: `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`

- [ ] **Step 1: 读取文件确认当前结构**

```bash
head -70 goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java
```

预期：看到类定义和现有字段

- [ ] **Step 2: 添加必要的 import 语句**

在文件顶部的 import 区域（约第 16-53 行之间）添加：

```java
import com.freedom.stream.annotation.StreamProducer;
import com.freedom.stream.annotation.SerializationType;
import com.freedom.stream.core.StreamTemplate;
import com.freedom.model.UserDrawStreamMessage;
```

位置：在现有 import 语句之后，class 定义之前

- [ ] **Step 3: 添加 @StreamProducer 字段**

在类中找到 `private final UserDrawTaskProperties userDrawTaskProperties;` 字段（约第 67 行），在其之后添加：

```java
/**
 * Redis Stream 生产者
 * 使用 @StreamProducer 注解自动注入
 */
@StreamProducer(
    stream = RedisStreamConfig.STREAM_USER_DRAW_TASKS,
    serialization = SerializationType.JSON
)
private StreamTemplate<UserDrawStreamMessage> userDrawProducer;
```

- [ ] **Step 4: 验证编译**

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/system
mvn compile -DskipTests
```

预期：BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java
git commit -m "feat: 添加 StreamProducer 字段注入到 PlotServiceImpl"
```

---

### Task 3: 重构 sendToRedisStream() 方法

**文件**: `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`

- [ ] **Step 1: 定位 sendToRedisStream() 方法**

```bash
grep -n "private void sendToRedisStream" goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java
```

预期：找到方法定义的行号（约第 332 行）

- [ ] **Step 2: 读取当前方法实现**

```bash
sed -n '332,355p' goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java
```

预期：看到使用 Redisson 原生 API 的实现

- [ ] **Step 3: 替换方法实现**

将 `sendToRedisStream()` 方法（约第 332-355 行）替换为：

```java
/**
 * 发送消息到 Redis Stream
 * 使用 @StreamProducer 注解注入的 StreamTemplate
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

- [ ] **Step 4: 检查是否可以移除 import**

检查以下 import 是否在文件其他地方使用：

```bash
grep -n "RStream\|StreamAddArgs" goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java | grep -v "import"
```

如果没有其他引用，删除这些 import：
```java
import org.redisson.api.RStream;
import org.redisson.api.stream.StreamAddArgs;
```

注意：`HashMap` 和 `Map` 可能在其他地方使用，需要检查后再决定是否删除。

- [ ] **Step 5: 验证编译**

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/system
mvn compile -DskipTests
```

预期：BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java
git commit -m "refactor: 重构 sendToRedisStream 使用 StreamTemplate"
```

---

### Task 4: 简化 customDrawPlot() 方法

**文件**: `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`

- [ ] **Step 1: 定位 customDrawPlot() 方法**

```bash
grep -n "public CommonResponse customDrawPlot" goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java
```

预期：找到方法定义的行号（约第 152 行）

- [ ] **Step 2: 定位灰度切换代码块**

在 customDrawPlot() 方法中搜索：

```bash
grep -n "灰度发布" goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java
```

预期：找到灰度切换逻辑的起始位置

- [ ] **Step 3: 读取 RHandlerWrapper 代码块**

找到 `RHandlerWrapper<Object> rHandlerWrapper = new RHandlerWrapper<>` 开始的代码块，确认其结构

- [ ] **Step 4: 简化 RHandlerWrapper 内的逻辑**

将 RHandlerWrapper 内的代码简化为：

```java
RHandlerWrapper<Object> rHandlerWrapper = new RHandlerWrapper<>(() -> {
    // 发送到 Redis Stream
    sendToRedisStream(plotType.getUserDraw().getId(), SecurityUtils.getCurrentUserId());
    log.info("任务已发送到 Redis Stream: userDrawId={}", plotType.getUserDraw().getId());
    return null;
});
```

移除以下内容：
- `if (userDrawTaskProperties.useRedisStream())` 条件判断
- `else if (userDrawTaskProperties.useRabbitMQ())` 分支及其内部的所有代码
- `else throw new RuntimeException("未配置任何消息队列")` 分支
- `String correlationId = RabbitPublishType.taskSubUser(...)` 生成
- `String correlationIdPrefix = RabbitPublishType.taskSubUserPrefix(...)` 查询
- 所有 `MqMessage` 相关的数据库操作

- [ ] **Step 5: 简化 doHandlerWithRetry 调用**

将：
```java
rHandlerWrapper.doHandlerWithRetry(
    componentBean.getRedissonClient(),
    RedissonKey.getUserDrawKey(plotType.getUserDraw().getId()),
    () -> {
        String correlationIdPrefix = RabbitPublishType.taskSubUserPrefix(...);
        List<MqMessage> mqMessages = ...;
        return CollectionUtils.isNotEmpty(mqMessages);
    }
);
```

简化为：
```java
rHandlerWrapper.doHandler(
    componentBean.getRedissonClient(),
    RedissonKey.getUserDrawKey(plotType.getUserDraw().getId())
);
```

- [ ] **Step 6: 简化异常处理**

将 `catch (RLockException e)` 块简化为：

```java
} catch (Exception e) {
    log.error("任务提交失败: userDrawId={}", plotType.getUserDraw().getId(), e);
    return ResponseUtils.failureMsg("任务提交失败，请稍后重试");
}
```

- [ ] **Step 7: 检查并移除不再使用的 import**

检查以下 import 是否在文件其他地方使用：

```bash
grep -n "RabbitPublishType\|MqMessageEnum\|RabbitMQConstant\|MqMessage\|UserDrawMQMessage\|UserDrawTaskRabbitMQUtils\|JSONUtil" goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java | grep -v "import"
```

如果没有其他引用，删除这些 import：
```java
import com.freedom.config.RabbitPublishType;
import com.freedom.constant.MqMessageEnum;
import com.freedom.constant.RabbitMQConstant;
import com.freedom.domain.MqMessage;
import com.freedom.model.UserDrawMQMessage;
import com.freedom.util.UserDrawTaskRabbitMQUtils;
import cn.hutool.json.JSONUtil;
```

- [ ] **Step 8: 验证编译**

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/system
mvn compile -DskipTests
```

预期：BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java
git commit -m "refactor: 简化 customDrawPlot 移除 RabbitMQ 代码"
```

---

## Chunk 2: 消费者配置更新

### 文件结构

**修改文件**:
- `goto/task/src/main/java/com/freedom/runner/UserDrawStreamConsumer.java`
  - 更新 @StreamListener 配置

---

### Task 5: 更新 UserDrawStreamConsumer 配置

**文件**: `goto/task/src/main/java/com/freedom/runner/UserDrawStreamConsumer.java`

- [ ] **Step 1: 读取当前配置**

```bash
grep -A 6 "@StreamListener" goto/task/src/main/java/com/freedom/runner/UserDrawStreamConsumer.java
```

预期：看到当前的 @StreamListener 配置

- [ ] **Step 2: 定位配置行号**

```bash
grep -n "concurrency = " goto/task/src/main/java/com/freedom/runner/UserDrawStreamConsumer.java
```

预期：找到 concurrency 配置的行号（约第 37 行）

- [ ] **Step 3: 更新配置参数**

将 @StreamListener 注解中的配置修改为：

```java
@StreamListener(
        stream = RedisStreamConfig.STREAM_USER_DRAW_TASKS,
        group = RedisStreamConfig.GROUP_TASK_SERVICE,
        concurrency = 4,   // 与 task.pool.user-draw.core-pool-size 一致
        maxRetries = 3,    // 减少重试次数
        enableBackpressure = true
)
```

变更：
- `concurrency = 10` → `concurrency = 4`
- `maxRetries = 5` → `maxRetries = 3`

- [ ] **Step 4: 验证编译**

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/task
mvn compile -DskipTests
```

预期：BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add goto/task/src/main/java/com/freedom/runner/UserDrawStreamConsumer.java
git commit -m "refactor: 更新 UserDrawStreamConsumer 配置参数"
```

---

## Chunk 3: 验证和测试

### Task 6: 编译验证

- [ ] **Step 1: 编译 system 模块**

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/system
mvn clean compile -DskipTests
```

预期：BUILD SUCCESS

- [ ] **Step 2: 编译 task 模块**

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/task
mvn clean compile -DskipTests
```

预期：BUILD SUCCESS

- [ ] **Step 3: 编译整个项目**

```bash
cd /mnt/f/IdeaProjects/goto-software
mvn clean compile -DskipTests
```

预期：BUILD SUCCESS

---

### Task 7: 代码审查检查

- [ ] **Step 1: 查看 PlotServiceImpl 变更摘要**

```bash
git diff --stat goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java
```

预期：显示增删行数统计

- [ ] **Step 2: 查看 PlotServiceImpl 详细变更**

```bash
git diff goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java
```

验证：
- ✅ 添加了 @StreamProducer 字段
- ✅ sendToRedisStream() 使用 StreamTemplate
- ✅ customDrawPlot() 移除了 RabbitMQ 代码
- ✅ 移除了不必要的 import

- [ ] **Step 3: 查看 UserDrawStreamConsumer 变更**

```bash
git diff goto/task/src/main/java/com/freedom/runner/UserDrawStreamConsumer.java
```

验证：
- ✅ concurrency 改为 4
- ✅ maxRetries 改为 3

- [ ] **Step 4: 查看所有变更文件列表**

```bash
git status
```

预期：只有以下文件被修改
- `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`
- `goto/task/src/main/java/com/freedom/runner/UserDrawStreamConsumer.java`

---

### Task 8: 配置验证

- [ ] **Step 1: 验证 Redis Stream 配置**

```bash
grep -A 2 "STREAM_USER_DRAW_TASKS\|GROUP_TASK_SERVICE" goto/common/src/main/java/com/freedom/config/RedisStreamConfig.java
```

预期：
- STREAM_USER_DRAW_TASKS = "stream:user_draw_tasks"
- GROUP_TASK_SERVICE = "group:task_service"

- [ ] **Step 2: 验证线程池配置**

```bash
grep -A 4 "user-draw:" goto/task/src/main/resources/application-product.yml
```

预期：
- core-pool-size: 4

- [ ] **Step 3: 验证 UserDrawStreamMessage 结构**

```bash
grep -A 5 "class UserDrawStreamMessage" goto/task/src/main/java/com/freedom/model/UserDrawStreamMessage.java
```

预期：包含 userDrawId 和 userId 字段

---

## 实施完成检查清单

### 代码变更
- [ ] PlotServiceImpl 添加 @StreamProducer 字段
- [ ] PlotServiceImpl 重构 sendToRedisStream() 方法
- [ ] PlotServiceImpl 简化 customDrawPlot() 方法
- [ ] PlotServiceImpl 移除 RabbitMQ 相关代码
- [ ] UserDrawStreamConsumer 更新配置参数

### 编译验证
- [ ] system 模块编译成功
- [ ] task 模块编译成功
- [ ] 整个项目编译成功

### 代码审查
- [ ] 所有变更符合设计文档
- [ ] 移除了不必要的 import
- [ ] 代码格式正确
- [ ] 注释清晰

### 配置验证
- [ ] Redis Stream 配置正确
- [ ] 线程池配置正确
- [ ] 消息类结构正确

---

## 下一步

实施计划完成后，建议进行以下测试：

1. **手动测试**：在开发环境启动应用，调用 customDrawPlot() API，验证消息发送和消费
2. **集成测试**：部署到测试环境，执行端到端测试
3. **性能测试**：执行并发测试，验证性能指标
4. **生产部署**：部署到生产环境，监控运行状态

---

**计划创建日期**: 2026-03-15
**预计实施时间**: 2-3 小时
**风险等级**: 中（移除了 RabbitMQ 代码，需要充分测试）
