# StreamConsumerRegistry 整合 StreamConsumerContainer 设计方案

**日期：** 2026-03-15
**作者：** Claude (Brainstorming Session)
**状态：** 待审查

---

## 1. 概述

### 1.1 问题描述

当前 `StreamConsumerRegistry` 存在致命缺陷：虽然 `StreamConsumerContainer` 已完整实现消费者逻辑，但 Registry 的 `start()` 和 `stop()` 方法只有空壳（TODO 标记），导致所有 `@StreamListener` 注解的消费者无法启动。

**影响范围：** 整个注解驱动的消息消费机制实际上是无效的。

### 1.2 解决方案

采用**依赖注入 + 容器管理**方案：
- 通过构造函数注入 `RedissonClient` 和 `ObjectMapper`
- Registry 负责创建和管理所有 `StreamConsumerContainer` 实例
- 实现完整的生命周期管理（启动、停止）

### 1.3 设计原则

- ✅ 依赖注入：符合 Spring 最佳实践
- ✅ 职责清晰：Registry 管理生命周期，Container 处理消费逻辑
- ✅ 易于测试：可以 mock 依赖
- ✅ 线程安全：使用并发安全的数据结构
- ✅ 优雅停机：确保消息处理完成后再停止

---

## 2. 架构设计

### 2.1 依赖关系

```
StreamConsumerRegistry
    ├── 依赖：RedissonClient（构造函数注入）
    ├── 依赖：ObjectMapper（构造函数注入）
    ├── 管理：List<StreamListenerEndpoint>（已有）
    └── 管理：Map<String, StreamConsumerContainer>（新增）
                    ↓
            StreamConsumerContainer
                ├── 依赖：StreamListenerEndpoint
                ├── 依赖：RedissonClient
                └── 依赖：ObjectMapper
```

### 2.2 生命周期流程

```
Spring 容器启动
    ↓
RedisStreamAutoConfiguration 创建 Bean
    ↓
StreamConsumerRegistry(redissonClient, objectMapper)
    ↓
StreamListenerBeanPostProcessor 扫描 @StreamListener
    ↓
registry.registerListener(endpoint) × N
    ↓
Spring 调用 registry.start()
    ↓
为每个 endpoint 创建 StreamConsumerContainer
    ↓
container.start() - 启动消费者线程
    ↓
应用运行中...
    ↓
Spring 关闭时调用 registry.stop()
    ↓
container.stop() - 优雅停止所有消费者
```

### 2.3 Container ID 生成策略

**格式：** `{stream}:{group}:{consumerName}`

**示例：** `order-events:order-service:order-consumer-0`

**优点：**
- 唯一性：stream + group + consumer 组合保证唯一
- 可读性：从 ID 就能看出容器的配置
- 可查询：方便根据 stream 或 group 查找容器

---

## 3. 实现细节

### 3.1 StreamConsumerRegistry 修改

#### 新增字段

```java
public class StreamConsumerRegistry implements SmartLifecycle {
    // 已有字段
    private final List<StreamListenerEndpoint> endpoints = new CopyOnWriteArrayList<>();
    private volatile boolean running = false;

    // 新增字段
    private final RedissonClient redissonClient;
    private final ObjectMapper objectMapper;
    private final Map<String, StreamConsumerContainer> containers = new ConcurrentHashMap<>();
}
```

#### 新增构造函数

```java
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
```

#### 修改 start() 方法

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
```

#### 修改 stop() 方法

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

#### 新增辅助方法

```java
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

### 3.2 RedisStreamAutoConfiguration 修改

#### 修改 Bean 创建方法

```java
@Bean
@ConditionalOnMissingBean
@ConditionalOnBean(RedissonClient.class)  // 新增条件
public StreamConsumerRegistry streamConsumerRegistry(
        RedissonClient redissonClient,
        ObjectMapper objectMapper) {
    return new StreamConsumerRegistry(redissonClient, objectMapper);
}
```

---

## 4. 错误处理

### 4.1 启动阶段

**依赖注入失败：**
- 构造函数参数校验，null 时抛出 `IllegalArgumentException`

**Container 启动失败：**
- 捕获异常，记录日志
- 继续启动其他容器
- 最终报告失败的容器列表

**没有注册端点：**
- 记录警告日志
- 正常启动但不创建容器

### 4.2 停止阶段

**Container 停止失败：**
- 捕获异常，记录日志
- 继续停止其他容器
- 最终报告成功/失败统计

**优雅停机：**
- 每个容器独立停止
- 等待消息处理完成（Container 内部实现）
- 清空容器集合

### 4.3 边界情况

**重复启动/停止：**
- 通过 `running` 标志实现幂等性

**Container ID 冲突：**
- 自动添加数字后缀
- 记录警告日志

**并发安全：**
- `CopyOnWriteArrayList` 用于 endpoints
- `ConcurrentHashMap` 用于 containers
- `volatile` 标志用于 running

---

## 5. 测试策略

### 5.1 单元测试（StreamConsumerRegistryTest）

**测试用例：**

1. **构造函数测试**
   - 正常创建 Registry
   - RedissonClient 为 null 时抛出异常
   - ObjectMapper 为 null 时抛出异常

2. **启动测试**
   - 启动成功，创建所有容器
   - 重复启动是幂等的
   - 没有端点时启动成功但不创建容器
   - 单个容器启动失败不影响其他容器
   - 启动后 running 标志为 true

3. **停止测试**
   - 停止成功，所有容器被停止
   - 重复停止是幂等的
   - 单个容器停止失败不影响其他容器
   - 停止后 containers 集合被清空
   - 停止后 running 标志为 false

4. **容器管理测试**
   - generateContainerId 生成正确格式的 ID
   - Container ID 冲突时自动添加后缀
   - getContainer 返回正确的容器
   - getAllContainers 返回不可修改的 Map

5. **并发测试**
   - 多线程同时注册端点
   - 多线程同时启动/停止

### 5.2 集成测试（StreamConsumerRegistryIntegrationTest）

**使用 Testcontainers：**
- 启动真实 Redis 容器
- 端到端测试消息流转
- 验证消费者正常工作

### 5.3 自动配置测试更新

**RedisStreamAutoConfigurationTest：**
- 验证 Bean 创建成功
- 验证依赖注入正确
- 验证条件注解生效

### 5.4 覆盖率目标

- StreamConsumerRegistry: 90%+
- RedisStreamAutoConfiguration: 85%+
- 整体项目: 80%+

### 5.5 TDD 流程

```
1. 编写测试（RED）→ 运行测试 → 失败
2. 实现代码（GREEN）→ 运行测试 → 通过
3. 重构（REFACTOR）→ 运行测试 → 仍然通过
4. 验证覆盖率 → mvn test jacoco:report
```

---

## 6. 改动文件清单

### 6.1 修改文件

- `StreamConsumerRegistry.java`
  - 添加构造函数
  - 添加字段（redissonClient, objectMapper, containers）
  - 实现 start() 方法
  - 实现 stop() 方法
  - 添加辅助方法

- `RedisStreamAutoConfiguration.java`
  - 修改 streamConsumerRegistry() Bean 创建方法
  - 添加 @ConditionalOnBean(RedissonClient.class)

### 6.2 新增测试文件

- `StreamConsumerRegistryTest.java`（单元测试）
- `StreamConsumerRegistryIntegrationTest.java`（集成测试）

### 6.3 更新测试文件

- `RedisStreamAutoConfigurationTest.java`

---

## 7. 实施计划

### 7.1 阶段划分

**阶段 1：编写测试（TDD - RED）**
- 编写 StreamConsumerRegistryTest 所有测试用例
- 运行测试，验证失败

**阶段 2：实现代码（TDD - GREEN）**
- 修改 StreamConsumerRegistry
- 修改 RedisStreamAutoConfiguration
- 运行测试，验证通过

**阶段 3：重构优化（TDD - REFACTOR）**
- 优化代码结构
- 提取重复逻辑
- 运行测试，确保仍然通过

**阶段 4：集成测试**
- 编写集成测试
- 使用 Testcontainers 验证端到端流程

**阶段 5：验证和提交**
- 验证测试覆盖率 >= 80%
- 运行所有测试
- 提交代码

### 7.2 预期时间

- 阶段 1：1 小时
- 阶段 2：2 小时
- 阶段 3：1 小时
- 阶段 4：1 小时
- 阶段 5：0.5 小时

**总计：约 5.5 小时**

---

## 8. 风险和缓解

### 8.1 风险

1. **Container 启动失败导致部分消费者不可用**
   - 缓解：独立启动每个容器，一个失败不影响其他

2. **停止时消息丢失**
   - 缓解：Container 内部已实现优雅停机

3. **并发问题**
   - 缓解：使用线程安全的数据结构

4. **测试覆盖不足**
   - 缓解：严格遵循 TDD，确保 80%+ 覆盖率

### 8.2 回滚计划

如果实施失败：
1. 回滚代码到当前版本
2. Registry 保持 TODO 状态
3. 重新评估方案

---

## 9. 验收标准

- ✅ 所有单元测试通过
- ✅ 所有集成测试通过
- ✅ 测试覆盖率 >= 80%
- ✅ 消费者能够正常启动和消费消息
- ✅ 优雅停机正常工作
- ✅ 错误处理符合预期
- ✅ 日志输出清晰完整
- ✅ 代码通过 Code Review

---

## 10. 后续优化

本次实施完成后，可以考虑的优化方向：

1. **监控增强**
   - 添加容器状态监控
   - 暴露容器管理 API

2. **动态管理**
   - 支持运行时添加/移除容器
   - 支持动态调整并发数

3. **健康检查集成**
   - 将容器状态集成到 StreamHealthIndicator

4. **性能优化**
   - 容器启动并行化
   - 批量操作优化

---

## 附录

### A. 参考文档

- [Redis Stream 通用封装实施计划](../../plans/2026-03-12-Redis-Stream-通用封装实施计划.md)
- [Redis Stream Spring Boot Starter README](../../../freedom-public/redis-stream-spring-boot-starter/README.md)
- [Redis Stream 使用文档](../../../freedom-public/redis-stream-spring-boot-starter/docs/USAGE.md)

### B. 相关 Issue

- TODO 标记位置：
  - `StreamConsumerRegistry.java:74` - start() 方法
  - `StreamConsumerRegistry.java:92` - stop() 方法
