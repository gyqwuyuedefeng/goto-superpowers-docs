# Redis Stream 通用封装实施计划

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**目标：** 创建独立的 Spring Boot Starter（redis-stream-spring-boot-starter），提供类型安全、注解驱动的 Redis Stream 封装

**架构：** 基于 Redisson 客户端，提供 StreamTemplate 核心模板类、注解驱动的消费者、自动配置、重试机制、背压控制和监控指标

**技术栈：** Spring Boot 2.7+, Redisson 3.x, Jackson, Micrometer, JUnit 5, Mockito

---

## 阶段一：项目结构和基础配置

### Task 1: 创建 Starter 模块结构

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/pom.xml`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/resources/META-INF/spring.factories`

**Step 1: 创建模块目录结构**

Run:
```bash
cd freedom-public
mkdir -p redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/{autoconfigure,annotation,core,config,support,serialization,monitoring}
mkdir -p redis-stream-spring-boot-starter/src/main/resources/META-INF
mkdir -p redis-stream-spring-boot-starter/src/test/java/com/freedom/stream
cd ..
```

Expected: 目录结构创建成功

**Step 2: 创建 pom.xml**

Create file: `freedom-public/redis-stream-spring-boot-starter/pom.xml`

Content: (见设计文档中的依赖配置)

**Step 3: 将模块添加到父 pom.xml**

Modify: `freedom-public/pom.xml`

Add module:
```xml
<module>redis-stream-spring-boot-starter</module>
```

**Step 4: 创建 spring.factories**

Create file: `freedom-public/redis-stream-spring-boot-starter/src/main/resources/META-INF/spring.factories`

Content:
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.freedom.stream.autoconfigure.RedisStreamAutoConfiguration
```

**Step 5: 验证模块结构**

Run:
```bash
cd freedom-public
mvn clean compile -pl redis-stream-spring-boot-starter
```

Expected: BUILD SUCCESS

**Step 6: Commit**

```bash
git add freedom-public/redis-stream-spring-boot-starter
git add freedom-public/pom.xml
git commit -m "feat: 创建 Redis Stream Starter 模块结构"
```

---

## 阶段二：核心配置和注解

### Task 2: 实现配置属性类

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/config/RedisStreamProperties.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/config/RedisStreamPropertiesTest.java`

**Step 1: 编写配置属性测试**

Create test file with basic property binding tests

**Step 2: 运行测试验证失败**

Run: `mvn test -Dtest=RedisStreamPropertiesTest`
Expected: FAIL (class not found)

**Step 3: 实现 RedisStreamProperties**

Create configuration properties class with:
- enabled flag
- backpressure settings
- retry settings
- monitoring settings

**Step 4: 运行测试验证通过**

Run: `mvn test -Dtest=RedisStreamPropertiesTest`
Expected: PASS

**Step 5: Commit**

```bash
git add .
git commit -m "feat: 实现 Redis Stream 配置属性类"
```

### Task 3: 实现核心注解

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/annotation/EnableRedisStream.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/annotation/StreamListener.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/annotation/StreamProducer.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/annotation/Backoff.java`

**Step 1: 实现 @EnableRedisStream 注解**

Content: (见设计文档)

**Step 2: 实现 @StreamListener 注解**

Content: (见设计文档)

**Step 3: 实现 @StreamProducer 注解**

Content: (见设计文档)

**Step 4: 实现 @Backoff 注解**

Content: (见设计文档)

**Step 5: 编译验证**

Run: `mvn clean compile`
Expected: BUILD SUCCESS

**Step 6: Commit**

```bash
git add .
git commit -m "feat: 实现 Redis Stream 核心注解"
```

---

## 阶段三：核心组件实现

### Task 4: 实现 MessageContext

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/core/MessageContext.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/core/MessageContextTest.java`

**Step 1: 编写 MessageContext 测试**

Test cases:
- 创建 MessageContext
- 获取消息元数据
- ACK 消息
- 重试计数

**Step 2: 运行测试验证失败**

Run: `mvn test -Dtest=MessageContextTest`
Expected: FAIL

**Step 3: 实现 MessageContext**

Implement with:
- messageId
- rawBody
- config
- receiveTimestamp
- retryCount
- ack() method

**Step 4: 运行测试验证通过**

Run: `mvn test -Dtest=MessageContextTest`
Expected: PASS

**Step 5: Commit**

```bash
git add .
git commit -m "feat: 实现 MessageContext 消息上下文"
```

### Task 5: 实现 StreamTemplate

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/core/StreamTemplate.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/core/StreamTemplateTest.java`

**Step 1: 编写 StreamTemplate 测试**

Test cases:
- 发送单条消息
- 批量发送消息
- 序列化消息
- 添加元数据

**Step 2: 运行测试验证失败**

Run: `mvn test -Dtest=StreamTemplateTest`
Expected: FAIL

**Step 3: 实现 StreamTemplate**

Implement with:
- send(T message)
- sendBatch(List<T> messages)
- serialization support

**Step 4: 运行测试验证通过**

Run: `mvn test -Dtest=StreamTemplateTest`
Expected: PASS

**Step 5: Commit**

```bash
git add .
git commit -m "feat: 实现 StreamTemplate 核心模板类"
```

### Task 6: 实现 AbstractStreamConsumer

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/core/AbstractStreamConsumer.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/core/AbstractStreamConsumerTest.java`

**Step 1: 编写 AbstractStreamConsumer 测试**

Test cases:
- 消费消息主循环
- 处理单条消息
- 反序列化消息
- 调用业务逻辑

**Step 2: 运行测试验证失败**

Run: `mvn test -Dtest=AbstractStreamConsumerTest`
Expected: FAIL

**Step 3: 实现 AbstractStreamConsumer**

Implement with:
- consumeMessages() main loop
- processMessage() handler
- abstract consume() method
- XREADGROUP integration

**Step 4: 运行测试验证通过**

Run: `mvn test -Dtest=AbstractStreamConsumerTest`
Expected: PASS

**Step 5: Commit**

```bash
git add .
git commit -m "feat: 实现 AbstractStreamConsumer 消费者抽象类"
```

---

## 阶段四：可靠性功能

### Task 7: 实现背压控制器

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/support/BackpressureController.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/support/BackpressureControllerTest.java`

**Step 1: 编写背压控制器测试**

Test cases:
- 检查是否应该暂停
- 计算当前使用率
- 阈值判断

**Step 2: 运行测试验证失败**

Run: `mvn test -Dtest=BackpressureControllerTest`
Expected: FAIL

**Step 3: 实现 BackpressureController**

Implement with:
- shouldPause() method
- getCurrentUsage() method
- threshold configuration

**Step 4: 运行测试验证通过**

Run: `mvn test -Dtest=BackpressureControllerTest`
Expected: PASS

**Step 5: Commit**

```bash
git add .
git commit -m "feat: 实现背压控制器"
```

### Task 8: 实现重试处理器

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/support/RetryHandler.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/support/RetryHandlerTest.java`

**Step 1: 编写重试处理器测试**

Test cases:
- 处理失败消息
- 检查重试次数
- 指数退避计算

**Step 2: 运行测试验证失败**

Run: `mvn test -Dtest=RetryHandlerTest`
Expected: FAIL

**Step 3: 实现 RetryHandler**

Implement with:
- handleFailure() method
- retry count check
- backoff calculation

**Step 4: 运行测试验证通过**

Run: `mvn test -Dtest=RetryHandlerTest`
Expected: PASS

**Step 5: Commit**

```bash
git add .
git commit -m "feat: 实现重试处理器"
```

### Task 9: 实现死信处理器

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/support/DeadLetterHandler.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/support/DeadLetterHandlerTest.java`

**Step 1: 编写死信处理器测试**

Test cases:
- 处理死信消息
- 记录日志
- 发送告警
- 强制 ACK

**Step 2: 运行测试验证失败**

Run: `mvn test -Dtest=DeadLetterHandlerTest`
Expected: FAIL

**Step 3: 实现 DeadLetterHandler**

Implement with:
- handle() method
- logging
- alert notification
- force ACK

**Step 4: 运行测试验证通过**

Run: `mvn test -Dtest=DeadLetterHandlerTest`
Expected: PASS

**Step 5: Commit**

```bash
git add .
git commit -m "feat: 实现死信处理器"
```

---

## 阶段五：自动配置

### Task 10: 实现自动配置类

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/RedisStreamAutoConfiguration.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/autoconfigure/RedisStreamAutoConfigurationTest.java`

**Step 1: 编写自动配置测试**

Test cases:
- 配置类加载
- Bean 创建
- 条件注解生效

**Step 2: 运行测试验证失败**

Run: `mvn test -Dtest=RedisStreamAutoConfigurationTest`
Expected: FAIL

**Step 3: 实现 RedisStreamAutoConfiguration**

Implement with:
- @Configuration
- @EnableConfigurationProperties
- Bean definitions
- Conditional annotations

**Step 4: 运行测试验证通过**

Run: `mvn test -Dtest=RedisStreamAutoConfigurationTest`
Expected: PASS

**Step 5: Commit**

```bash
git add .
git commit -m "feat: 实现 Redis Stream 自动配置"
```

---

## 阶段六：监控和健康检查

### Task 11: 实现监控指标收集器

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/monitoring/StreamMetricsCollector.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/monitoring/StreamMetricsCollectorTest.java`

**Step 1: 编写监控指标测试**

Test cases:
- 收集 Stream 长度
- 收集 PEL 长度
- 收集消息处理延迟
- 收集成功/失败计数

**Step 2: 运行测试验证失败**

Run: `mvn test -Dtest=StreamMetricsCollectorTest`
Expected: FAIL

**Step 3: 实现 StreamMetricsCollector**

Implement with Micrometer:
- Stream length gauge
- PEL length gauge
- Message latency timer
- Success/failure counters

**Step 4: 运行测试验证通过**

Run: `mvn test -Dtest=StreamMetricsCollectorTest`
Expected: PASS

**Step 5: Commit**

```bash
git add .
git commit -m "feat: 实现监控指标收集器"
```

### Task 12: 实现健康检查

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/monitoring/StreamHealthIndicator.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/monitoring/StreamHealthIndicatorTest.java`

**Step 1: 编写健康检查测试**

Test cases:
- 检查 Stream 存在
- 检查 Consumer Group 存在
- 检查消费者运行状态
- 检查 PEL 长度

**Step 2: 运行测试验证失败**

Run: `mvn test -Dtest=StreamHealthIndicatorTest`
Expected: FAIL

**Step 3: 实现 StreamHealthIndicator**

Implement HealthIndicator:
- Check stream exists
- Check consumer group exists
- Check consumer running
- Check PEL length

**Step 4: 运行测试验证通过**

Run: `mvn test -Dtest=StreamHealthIndicatorTest`
Expected: PASS

**Step 5: Commit**

```bash
git add .
git commit -m "feat: 实现健康检查"
```

---

## 阶段七：集成测试

### Task 13: 编写端到端集成测试

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/integration/RedisStreamIntegrationTest.java`

**Step 1: 编写集成测试**

Test cases:
- 完整的消息流转
- 超时重试机制
- 死信处理
- 幂等性保证
- 消费者宕机恢复

**Step 2: 运行集成测试**

Run: `mvn test -Dtest=RedisStreamIntegrationTest`
Expected: PASS

**Step 3: Commit**

```bash
git add .
git commit -m "test: 添加端到端集成测试"
```

---

## 阶段八：迁移现有实现

### Task 14: 迁移 UserDrawStreamConsumer

**Files:**
- Modify: `goto/task/src/main/java/com/freedom/runner/UserDrawStreamConsumer.java`
- Modify: `goto/task/src/main/java/com/freedom/thread/UserDrawRunnable.java`
- Modify: `goto/task/src/main/java/com/freedom/util/DoUserDrawTask.java`

**Step 1: 添加 Starter 依赖到 goto/task**

Modify: `goto/task/pom.xml`

Add dependency:
```xml
<dependency>
    <groupId>com.freedom</groupId>
    <artifactId>redis-stream-spring-boot-starter</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

**Step 2: 重构 UserDrawStreamConsumer 使用新框架**

Use @StreamListener annotation and implement business logic with distributed lock

**Step 3: 简化 DoUserDrawTask**

Remove manual ACK and retry logic

**Step 4: 更新 UserDrawRunnable**

Add distributed lock and proper ACK in finally block

**Step 5: 运行测试验证**

Run: `mvn test -pl goto/task`
Expected: PASS

**Step 6: Commit**

```bash
git add goto/task
git commit -m "refactor: 迁移 UserDrawStreamConsumer 到新框架"
```

---

## 阶段九：文档和示例

### Task 15: 编写使用文档

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/README.md`
- Create: `freedom-public/redis-stream-spring-boot-starter/docs/USAGE.md`
- Create: `freedom-public/redis-stream-spring-boot-starter/docs/EXAMPLES.md`

**Step 1: 编写 README**

Include:
- 功能介绍
- 快速开始
- 配置说明
- 使用示例

**Step 2: 编写详细使用文档**

Include:
- 生产者使用
- 消费者使用
- 配置详解
- 最佳实践

**Step 3: 编写示例代码**

Include:
- 基础示例
- 高级示例
- 常见问题

**Step 4: Commit**

```bash
git add freedom-public/redis-stream-spring-boot-starter/README.md
git add freedom-public/redis-stream-spring-boot-starter/docs/
git commit -m "docs: 添加使用文档和示例"
```

---

## 验证清单

完成所有任务后，验证以下内容：

- [ ] 所有单元测试通过
- [ ] 所有集成测试通过
- [ ] 代码覆盖率 >= 80%
- [ ] 文档完整
- [ ] 示例代码可运行
- [ ] 现有功能迁移成功
- [ ] 性能测试通过

---

## 注意事项

1. **TDD 原则**：每个任务都遵循"写测试 → 运行测试 → 实现代码 → 运行测试 → 提交"的流程
2. **分布式锁**：在业务逻辑执行期间持有，使用 Redisson 看门狗机制自动续期
3. **ACK 位置**：在业务逻辑执行完成后的 finally 块中执行
4. **频繁提交**：每完成一个任务就提交代码
5. **测试覆盖**：确保 80% 以上的代码覆盖率

---

## 预期时间

- 阶段一：0.5 天
- 阶段二：0.5 天
- 阶段三：1 天
- 阶段四：1 天
- 阶段五：0.5 天
- 阶段六：0.5 天
- 阶段七：0.5 天
- 阶段八：1 天
- 阶段九：0.5 天

**总计：约 6 天**
