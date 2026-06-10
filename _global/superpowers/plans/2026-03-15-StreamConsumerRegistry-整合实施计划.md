# StreamConsumerRegistry 整合 StreamConsumerContainer 实施计划

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**目标：** 实现 StreamConsumerRegistry 与 StreamConsumerContainer 的整合，使 @StreamListener 注解的消费者能够正常启动和工作

**架构：** 通过构造函数注入 RedissonClient 和 ObjectMapper 到 Registry，Registry 负责创建和管理所有 Container 实例，实现完整的生命周期管理

**技术栈：** Spring Boot 3.x, Redisson 3.x, Jackson, JUnit 5, Mockito

**设计规范：** [StreamConsumerRegistry 整合 StreamConsumerContainer 设计方案](../specs/2026-03-15-StreamConsumerRegistry-整合-StreamConsumerContainer-设计.md)

---

## 文件结构概览

### 修改文件

- `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/StreamConsumerRegistry.java`
  - 职责：管理所有消费者容器的生命周期
  - 修改：添加构造函数、字段、实现 start/stop 方法、添加辅助方法

- `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/RedisStreamAutoConfiguration.java`
  - 职责：自动配置 Bean 创建
  - 修改：修改 streamConsumerRegistry() Bean 创建方法，注入依赖

### 新增测试文件

- `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/autoconfigure/StreamConsumerRegistryTest.java`
  - 职责：StreamConsumerRegistry 单元测试
  - 覆盖：构造函数、启动、停止、容器管理、并发测试

### 更新测试文件

- `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/autoconfigure/RedisStreamAutoConfigurationTest.java`
  - 职责：自动配置测试
  - 修改：验证依赖注入和条件注解

---

## Chunk 1: 构造函数和字段（TDD）

### Task 1: 构造函数参数校验测试

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/autoconfigure/StreamConsumerRegistryTest.java`

- [ ] **Step 1: 创建测试类骨架**

```java
package com.freedom.stream.autoconfigure;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.redisson.api.RedissonClient;

import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
class StreamConsumerRegistryTest {

    @Mock
    private RedissonClient redissonClient;

    @Mock
    private RedissonClient redissonClient;

    @Mock
    private ObjectMapper objectMapper;
}
```

- [ ] **Step 2: 编写构造函数 null 校验测试**

```java
@Test
void constructor_withNullRedissonClient_shouldThrowException() {
    IllegalArgumentException exception = assertThrows(
        IllegalArgumentException.class,
        () -> new StreamConsumerRegistry(null, objectMapper)
    );
    assertEquals("RedissonClient cannot be null", exception.getMessage());
}

@Test
void constructor_withNullObjectMapper_shouldThrowException() {
    IllegalArgumentException exception = assertThrows(
        IllegalArgumentException.class,
        () -> new StreamConsumerRegistry(redissonClient, null)
    );
    assertEquals("ObjectMapper cannot be null", exception.getMessage());
}

@Test
void constructor_withValidParameters_shouldCreateInstance() {
    StreamConsumerRegistry registry = new StreamConsumerRegistry(redissonClient, objectMapper);
    assertNotNull(registry);
}
```

- [ ] **Step 3: 运行测试验证失败**

Run:
```bash
cd freedom-public/redis-stream-spring-boot-starter
mvn test -Dtest=StreamConsumerRegistryTest
```

Expected: FAIL - 编译错误，构造函数不存在

- [ ] **Step 4: 实现构造函数**

Modify: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/StreamConsumerRegistry.java`

在类中添加字段和构造函数：

```java
public class StreamConsumerRegistry implements SmartLifecycle {

    private static final Logger log = LoggerFactory.getLogger(StreamConsumerRegistry.class);

    // 已有字段
    private final List<StreamListenerEndpoint> endpoints = new CopyOnWriteArrayList<>();
    private volatile boolean running = false;

    // 新增字段
    private final RedissonClient redissonClient;
    private final ObjectMapper objectMapper;
    private final Map<String, StreamConsumerContainer> containers = new ConcurrentHashMap<>();

    // 新增构造函数
    public StreamConsumerRegistry(RedissonClient redissonClient, ObjectMapper objectMapper) {
        if (redissonClient == null) {
            throw new IllegalArgumentException("RedissonClient cannot be null");
        }
        if (objectMapper == null) {
            throw new IllegalArgumentException("ObjectMapper cannot be null");
        }
        this.redissonClient = redissonClient;
        this.objectMapper = objectMapper;
    }

    // ... 其他已有方法保持不变
}
```

需要添加的 import：
```java
import com.fasterxml.jackson.databind.ObjectMapper;
import org.redisson.api.RedissonClient;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.Collections;
import java.util.ArrayList;
```

- [ ] **Step 5: 运行测试验证通过**

Run:
```bash
cd freedom-public/redis-stream-spring-boot-starter
mvn test -Dtest=StreamConsumerRegistryTest
```

Expected: PASS - 所有构造函数测试通过

- [ ] **Step 6: Commit**

```bash
git add freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/StreamConsumerRegistry.java
git add freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/autoconfigure/StreamConsumerRegistryTest.java
git commit -m "feat: 添加 StreamConsumerRegistry 构造函数和依赖注入

- 添加 RedissonClient 和 ObjectMapper 字段
- 添加构造函数参数校验
- 添加 containers Map 用于管理容器
- 添加构造函数测试用例

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Chunk 2: start() 方法实现（TDD）

### Task 2: start() 方法测试

**Files:**
- Modify: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/autoconfigure/StreamConsumerRegistryTest.java`

- [ ] **Step 1: 添加测试辅助方法**

在 StreamConsumerRegistryTest 类中添加：

```java
import com.freedom.stream.annotation.SerializationType;
import org.redisson.api.RStream;
import java.lang.reflect.Method;

@Mock
private RStream<Object, Object> rStream;

private StreamListenerEndpoint createTestEndpoint(String stream, String group, String consumer) {
    try {
        Method method = StreamConsumerRegistryTest.class.getDeclaredMethod("dummyHandler", Object.class);
        return new StreamListenerEndpoint(
            stream, group, consumer,
            1, 3, 60000L,
            1000L, 2.0, 60000L,
            true, 0.8,
            SerializationType.JSON, stream + "-dlq",
            this, method
        );
    } catch (NoSuchMethodException e) {
        throw new RuntimeException(e);
    }
}

// Dummy handler method for testing
public void dummyHandler(Object message) {
    // Test handler
}
```

- [ ] **Step 2: 编写 start() 测试用例**

```java
import org.junit.jupiter.api.BeforeEach;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.when;

@BeforeEach
void setUp() {
    when(redissonClient.getStream(anyString())).thenReturn(rStream);
}

@Test
void start_withNoEndpoints_shouldStartSuccessfully() {
    StreamConsumerRegistry registry = new StreamConsumerRegistry(redissonClient, objectMapper);

    registry.start();

    assertTrue(registry.isRunning());
    assertEquals(0, registry.getAllContainers().size());
}

@Test
void start_withEndpoints_shouldCreateAndStartContainers() {
    StreamConsumerRegistry registry = new StreamConsumerRegistry(redissonClient, objectMapper);
    StreamListenerEndpoint endpoint = createTestEndpoint("test-stream", "test-group", "test-consumer");
    registry.registerListener(endpoint);

    registry.start();

    assertTrue(registry.isRunning());
    assertEquals(1, registry.getAllContainers().size());

    String containerId = "test-stream:test-group:test-consumer";
    assertNotNull(registry.getContainer(containerId));
}

@Test
void start_whenAlreadyRunning_shouldBeIdempotent() {
    StreamConsumerRegistry registry = new StreamConsumerRegistry(redissonClient, objectMapper);

    registry.start();
    int containerCount = registry.getAllContainers().size();

    registry.start(); // 第二次启动

    assertEquals(containerCount, registry.getAllContainers().size());
    assertTrue(registry.isRunning());
}
```

- [ ] **Step 3: 运行测试验证失败**

Run:
```bash
cd freedom-public/redis-stream-spring-boot-starter
mvn test -Dtest=StreamConsumerRegistryTest#start_withNoEndpoints_shouldStartSuccessfully
mvn test -Dtest=StreamConsumerRegistryTest#start_withEndpoints_shouldCreateAndStartContainers
mvn test -Dtest=StreamConsumerRegistryTest#start_whenAlreadyRunning_shouldBeIdempotent
```

Expected: FAIL - start() 方法未实现，getAllContainers() 和 getContainer() 方法不存在

- [ ] **Step 4: 实现 start() 方法和辅助方法**

Modify: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/StreamConsumerRegistry.java`

替换现有的 start() 方法：

```java
@Override
public void start() {
    if (running) {
        return;
    }

    if (endpoints.isEmpty()) {
        log.warn("No endpoints registered, StreamConsumerRegistry will start but do nothing");
        running = true;
        return;
    }

    log.info("Starting StreamConsumerRegistry with {} endpoints", endpoints.size());

    List<String> failedContainers = new ArrayList<>();

    for (StreamListenerEndpoint endpoint : endpoints) {
        String containerId = generateContainerId(endpoint);

        try {
            log.debug("Creating container for stream={}, group={}, concurrency={}",
                    endpoint.getStream(), endpoint.getGroup(), endpoint.getConcurrency());

            StreamConsumerContainer container = new StreamConsumerContainer(
                    endpoint, redissonClient, objectMapper);

            containers.put(containerId, container);
            container.start();

            log.info("Container started: {}", containerId);
        } catch (Exception e) {
            log.error("Failed to start container: {}", containerId, e);
            failedContainers.add(containerId);
            // 继续启动其他容器
        }
    }

    running = true;

    if (!failedContainers.isEmpty()) {
        log.warn("StreamConsumerRegistry started with {} failed containers: {}",
                failedContainers.size(), failedContainers);
    } else {
        log.info("StreamConsumerRegistry started successfully with {} containers",
                containers.size());
    }
}

/**
 * 生成容器 ID
 * 格式：{stream}:{group}:{consumerName}
 */
private String generateContainerId(StreamListenerEndpoint endpoint) {
    String baseId = String.format("%s:%s:%s",
            endpoint.getStream(),
            endpoint.getGroup(),
            endpoint.getConsumerName());

    // 处理 ID 冲突
    String containerId = baseId;
    int suffix = 1;
    while (containers.containsKey(containerId)) {
        containerId = baseId + "-" + suffix;
        suffix++;
        log.warn("Container ID conflict detected, using: {}", containerId);
    }

    return containerId;
}

/**
 * 获取指定容器
 */
public StreamConsumerContainer getContainer(String containerId) {
    return containers.get(containerId);
}

/**
 * 获取所有容器
 */
public Map<String, StreamConsumerContainer> getAllContainers() {
    return Collections.unmodifiableMap(containers);
}
```

- [ ] **Step 5: 运行测试验证通过**

Run:
```bash
cd freedom-public/redis-stream-spring-boot-starter
mvn test -Dtest=StreamConsumerRegistryTest
```

Expected: PASS - 所有 start() 相关测试通过

- [ ] **Step 6: Commit**

```bash
git add freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/StreamConsumerRegistry.java
git add freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/autoconfigure/StreamConsumerRegistryTest.java
git commit -m "feat: 实现 StreamConsumerRegistry start() 方法

- 实现容器创建和启动逻辑
- 添加 generateContainerId() 辅助方法
- 添加 getContainer() 和 getAllContainers() 方法
- 处理空端点和启动失败场景
- 添加 start() 方法测试用例

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Chunk 3: stop() 方法实现（TDD）

### Task 3: stop() 方法测试

**Files:**
- Modify: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/autoconfigure/StreamConsumerRegistryTest.java`

- [ ] **Step 1: 编写 stop() 测试用例**

```java
@Test
void stop_shouldStopAllContainers() {
    StreamConsumerRegistry registry = new StreamConsumerRegistry(redissonClient, objectMapper);
    StreamListenerEndpoint endpoint = createTestEndpoint("test-stream", "test-group", "test-consumer");
    registry.registerListener(endpoint);
    registry.start();

    registry.stop();

    assertFalse(registry.isRunning());
    assertEquals(0, registry.getAllContainers().size());
}

@Test
void stop_whenNotRunning_shouldBeIdempotent() {
    StreamConsumerRegistry registry = new StreamConsumerRegistry(redissonClient, objectMapper);

    registry.stop(); // 第一次停止
    registry.stop(); // 第二次停止

    assertFalse(registry.isRunning());
}

@Test
void stop_afterStart_shouldStopAllContainers() {
    StreamConsumerRegistry registry = new StreamConsumerRegistry(redissonClient, objectMapper);
    StreamListenerEndpoint endpoint1 = createTestEndpoint("stream1", "group1", "consumer1");
    StreamListenerEndpoint endpoint2 = createTestEndpoint("stream2", "group2", "consumer2");
    registry.registerListener(endpoint1);
    registry.registerListener(endpoint2);
    registry.start();

    assertEquals(2, registry.getAllContainers().size());

    registry.stop();

    assertFalse(registry.isRunning());
    assertEquals(0, registry.getAllContainers().size());
}
```

- [ ] **Step 2: 运行测试验证失败**

Run:
```bash
cd freedom-public/redis-stream-spring-boot-starter
mvn test -Dtest=StreamConsumerRegistryTest#stop_shouldStopAllContainers
mvn test -Dtest=StreamConsumerRegistryTest#stop_whenNotRunning_shouldBeIdempotent
mvn test -Dtest=StreamConsumerRegistryTest#stop_afterStart_shouldStopAllContainers
```

Expected: FAIL - stop() 方法未正确实现

- [ ] **Step 3: 实现 stop() 方法**

Modify: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/StreamConsumerRegistry.java`

替换现有的 stop() 方法：

```java
@Override
public void stop() {
    if (!running) {
        return;
    }

    log.info("Stopping StreamConsumerRegistry with {} containers", containers.size());

    int successCount = 0;
    int failureCount = 0;

    for (Map.Entry<String, StreamConsumerContainer> entry : containers.entrySet()) {
        String containerId = entry.getKey();
        StreamConsumerContainer container = entry.getValue();

        try {
            log.debug("Stopping container: {}", containerId);
            container.stop();
            log.info("Container stopped: {}", containerId);
            successCount++;
        } catch (Exception e) {
            log.error("Failed to stop container: {}", containerId, e);
            failureCount++;
        }
    }

    containers.clear();
    running = false;

    log.info("StreamConsumerRegistry stopped: {} succeeded, {} failed",
            successCount, failureCount);
}
```

- [ ] **Step 4: 运行测试验证通过**

Run:
```bash
cd freedom-public/redis-stream-spring-boot-starter
mvn test -Dtest=StreamConsumerRegistryTest
```

Expected: PASS - 所有测试通过

- [ ] **Step 5: Commit**

```bash
git add freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/StreamConsumerRegistry.java
git add freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/autoconfigure/StreamConsumerRegistryTest.java
git commit -m "feat: 实现 StreamConsumerRegistry stop() 方法

- 实现容器停止和清理逻辑
- 处理停止失败场景
- 添加成功/失败统计
- 添加 stop() 方法测试用例

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Chunk 4: RedisStreamAutoConfiguration 修改（TDD）

### Task 4: 修改自动配置 Bean 创建

**Files:**
- Modify: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/autoconfigure/RedisStreamAutoConfigurationTest.java`
- Modify: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/RedisStreamAutoConfiguration.java`

- [ ] **Step 1: 更新自动配置测试**

Modify: `RedisStreamAutoConfigurationTest.java`

添加测试用例：

```java
@Test
void streamConsumerRegistry_shouldBeCreatedWithDependencies() {
    contextRunner
        .withUserConfiguration(RedissonTestConfig.class)
        .run(context -> {
            assertThat(context).hasSingleBean(StreamConsumerRegistry.class);

            StreamConsumerRegistry registry = context.getBean(StreamConsumerRegistry.class);
            assertNotNull(registry);

            // 验证依赖注入成功（不会抛出 NPE）
            assertNotNull(registry.getAllContainers());
        });
}

@Test
void streamConsumerRegistry_withoutRedissonClient_shouldNotBeCreated() {
    contextRunner
        .run(context -> {
            assertThat(context).doesNotHaveBean(StreamConsumerRegistry.class);
        });
}
```

- [ ] **Step 2: 运行测试验证失败**

Run:
```bash
cd freedom-public/redis-stream-spring-boot-starter
mvn test -Dtest=RedisStreamAutoConfigurationTest#streamConsumerRegistry_shouldBeCreatedWithDependencies
```

Expected: FAIL - Bean 创建失败，缺少构造函数参数

- [ ] **Step 3: 修改 RedisStreamAutoConfiguration**

Modify: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/RedisStreamAutoConfiguration.java`

找到 streamConsumerRegistry() 方法并修改：

```java
/**
 * 创建消费者注册表 Bean
 * <p>
 * 负责注册和管理所有 @StreamListener 监听器端点
 * </p>
 *
 * @param redissonClient Redisson 客户端
 * @param objectMapper   JSON 序列化器
 * @return StreamConsumerRegistry 实例
 */
@Bean
@ConditionalOnMissingBean
@ConditionalOnBean(RedissonClient.class)
public StreamConsumerRegistry streamConsumerRegistry(
        RedissonClient redissonClient,
        ObjectMapper objectMapper) {
    return new StreamConsumerRegistry(redissonClient, objectMapper);
}
```

- [ ] **Step 4: 运行测试验证通过**

Run:
```bash
cd freedom-public/redis-stream-spring-boot-starter
mvn test -Dtest=RedisStreamAutoConfigurationTest
```

Expected: PASS - 所有自动配置测试通过

- [ ] **Step 5: Commit**

```bash
git add freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/RedisStreamAutoConfiguration.java
git add freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/autoconfigure/RedisStreamAutoConfigurationTest.java
git commit -m "feat: 修改 RedisStreamAutoConfiguration Bean 创建

- 修改 streamConsumerRegistry() 方法注入依赖
- 添加 @ConditionalOnBean(RedissonClient.class) 条件
- 更新自动配置测试用例

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Chunk 5: 验证和最终测试

### Task 5: 运行完整测试套件

**Files:**
- All test files

- [ ] **Step 1: 运行所有单元测试**

Run:
```bash
cd freedom-public/redis-stream-spring-boot-starter
mvn clean test
```

Expected: PASS - 所有测试通过

- [ ] **Step 2: 验证测试覆盖率**

Run:
```bash
cd freedom-public/redis-stream-spring-boot-starter
mvn test jacoco:report
```

查看报告：
```bash
open target/site/jacoco/index.html
```

Expected:
- StreamConsumerRegistry 覆盖率 >= 90%
- RedisStreamAutoConfiguration 覆盖率 >= 85%
- 整体覆盖率 >= 80%

- [ ] **Step 3: 运行完整构建**

Run:
```bash
cd freedom-public
mvn clean install -pl redis-stream-spring-boot-starter
```

Expected: BUILD SUCCESS

- [ ] **Step 4: 最终 Commit**

```bash
git add .
git commit -m "feat: 完成 StreamConsumerRegistry 整合 StreamConsumerContainer

实现内容：
- 添加构造函数依赖注入（RedissonClient, ObjectMapper）
- 实现 start() 方法：创建和启动所有容器
- 实现 stop() 方法：停止和清理所有容器
- 添加容器管理方法（getContainer, getAllContainers, generateContainerId）
- 修改 RedisStreamAutoConfiguration Bean 创建
- 添加完整的单元测试（覆盖率 90%+）
- 更新自动配置测试

测试覆盖：
- 构造函数参数校验
- 启动/停止生命周期
- 容器管理
- 错误处理
- 幂等性

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## 验收标准

完成所有任务后，验证以下标准：

- [ ] 所有单元测试通过
- [ ] 测试覆盖率 >= 80%（StreamConsumerRegistry >= 90%）
- [ ] 构建成功（mvn clean install）
- [ ] 代码符合项目规范（无 TODO 标记）
- [ ] 日志输出清晰完整
- [ ] 错误处理符合设计规范
- [ ] 幂等性测试通过
- [ ] 并发安全测试通过

---

## 故障排查

### 问题 1：测试编译失败

**症状：** 找不到 StreamConsumerContainer 类

**解决：**
```bash
cd freedom-public/redis-stream-spring-boot-starter
mvn clean compile
```

### 问题 2：Mock 对象行为不符合预期

**症状：** 测试中 container.start() 抛出异常

**解决：** 检查 Mock 配置，确保 redissonClient.getStream() 返回 mock 的 RStream

### 问题 3：覆盖率不足

**症状：** 覆盖率低于 80%

**解决：** 添加边界情况测试（空端点、启动失败、停止失败等）

---

## 参考资料

- 设计规范：[StreamConsumerRegistry 整合 StreamConsumerContainer 设计方案](../specs/2026-03-15-StreamConsumerRegistry-整合-StreamConsumerContainer-设计.md)
- TDD 技能：@superpowers:test-driven-development
- Spring Boot 测试：https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing
