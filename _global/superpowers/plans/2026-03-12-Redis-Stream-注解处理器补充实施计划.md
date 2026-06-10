# Redis Stream 注解处理器补充实施计划

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**目标：** 补充缺失的注解处理器和消费者管理组件，让 @StreamListener 和 @StreamProducer 注解真正工作

**背景：** 原实施计划遗漏了注解处理逻辑，导致注解无法工作。本计划补充这些关键组件。

**技术栈：** Spring Boot 2.7+, Redisson 3.x, Jackson, Reflection API

---

## 阶段一：注解处理器实现

### Task 1: 实现 StreamListenerBeanPostProcessor

**目标：** 扫描和处理 @StreamListener 注解，自动注册消费者

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/StreamListenerBeanPostProcessor.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/autoconfigure/StreamListenerBeanPostProcessorTest.java`

**Step 1: 编写测试**

Test cases:
- 扫描带有 @StreamListener 的方法
- 提取注解配置
- 注册到 StreamConsumerRegistry

**Step 2: 实现 BeanPostProcessor**

Requirements:
- 实现 BeanPostProcessor 接口
- 在 postProcessAfterInitialization 中扫描 @StreamListener
- 使用反射获取方法和注解信息
- 创建 StreamListenerEndpoint 并注册

**Step 3: 运行测试验证通过**

**Step 4: Commit**

```bash
git add .
git commit -m "feat: 实现 StreamListener 注解处理器"
```

---

### Task 2: 实现 StreamConsumerRegistry

**目标：** 管理所有注册的消费者端点

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/StreamConsumerRegistry.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/StreamListenerEndpoint.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/autoconfigure/StreamConsumerRegistryTest.java`

**Step 1: 编写测试**

Test cases:
- 注册消费者端点
- 查询已注册的端点
- 启动所有消费者

**Step 2: 实现 StreamListenerEndpoint**

Requirements:
- 存储方法引用、Bean 实例、注解配置
- 提供 invoke() 方法调用业务逻辑

**Step 3: 实现 StreamConsumerRegistry**

Requirements:
- 维护端点列表
- 提供注册和查询方法
- 实现 SmartLifecycle 接口，自动启动消费者

**Step 4: 运行测试验证通过**

**Step 5: Commit**

```bash
git add .
git commit -m "feat: 实现消费者注册表和端点"
```

---

### Task 3: 实现 StreamConsumerContainer

**目标：** 管理消费者的生命周期，启动消费线程

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/StreamConsumerContainer.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/autoconfigure/StreamConsumerContainerTest.java`

**Step 1: 编写测试**

Test cases:
- 启动消费者线程
- 停止消费者线程
- 处理消息并调用端点

**Step 2: 实现 StreamConsumerContainer**

Requirements:
- 为每个端点创建消费线程
- 使用 XREADGROUP 读取消息
- 反序列化消息并调用端点方法
- 处理 ACK 和错误重试
- 支持并发消费（concurrency 配置）

**Step 3: 运行测试验证通过**

**Step 4: Commit**

```bash
git add .
git commit -m "feat: 实现消费者容器"
```

---

## 阶段二：生产者支持

### Task 4: 实现 StreamProducerBeanPostProcessor

**目标：** 处理 @StreamProducer 注解，自动注入 StreamTemplate

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/StreamProducerBeanPostProcessor.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/autoconfigure/StreamProducerBeanPostProcessorTest.java`

**Step 1: 编写测试**

Test cases:
- 扫描带有 @StreamProducer 的字段
- 创建 StreamTemplate 实例
- 注入到字段

**Step 2: 实现 BeanPostProcessor**

Requirements:
- 实现 BeanPostProcessor 接口
- 扫描 @StreamProducer 注解的字段
- 使用 StreamProducerFactory 创建实例
- 通过反射注入字段

**Step 3: 运行测试验证通过**

**Step 4: Commit**

```bash
git add .
git commit -m "feat: 实现 StreamProducer 注解处理器"
```

---

### Task 5: 实现 StreamProducerFactory

**目标：** 创建和管理 StreamTemplate 实例

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/producer/StreamProducerFactory.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/producer/StreamProducerFactoryTest.java`

**Step 1: 编写测试**

Test cases:
- 创建 StreamTemplate 实例
- 缓存已创建的实例
- 配置序列化器

**Step 2: 实现 StreamProducerFactory**

Requirements:
- 根据 stream 名称创建 StreamTemplate
- 缓存实例避免重复创建
- 支持不同的序列化类型

**Step 3: 运行测试验证通过**

**Step 4: Commit**

```bash
git add .
git commit -m "feat: 实现生产者工厂"
```

---

## 阶段三：序列化支持

### Task 6: 实现序列化器接口和实现

**目标：** 提供可扩展的序列化支持

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/serialization/MessageSerializer.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/serialization/JsonMessageSerializer.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/serialization/SerializerFactory.java`
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/serialization/JsonMessageSerializerTest.java`

**Step 1: 编写测试**

Test cases:
- JSON 序列化和反序列化
- 处理泛型类型
- 错误处理

**Step 2: 实现序列化器**

Requirements:
- MessageSerializer<T> 接口
- JsonMessageSerializer 实现（使用 Jackson）
- SerializerFactory 工厂类

**Step 3: 运行测试验证通过**

**Step 4: Commit**

```bash
git add .
git commit -m "feat: 实现消息序列化器"
```

---

## 阶段四：完善自动配置

### Task 7: 更新 RedisStreamAutoConfiguration

**目标：** 注册所有必要的 Bean

**Files:**
- Modify: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/autoconfigure/RedisStreamAutoConfiguration.java`
- Delete: `freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/config/RedisStreamAutoConfiguration.java`

**Step 1: 删除占位符文件**

```bash
rm freedom-public/redis-stream-spring-boot-starter/src/main/java/com/freedom/stream/config/RedisStreamAutoConfiguration.java
```

**Step 2: 更新自动配置类**

Add Beans:
- StreamListenerBeanPostProcessor
- StreamProducerBeanPostProcessor
- StreamConsumerRegistry
- StreamConsumerContainer
- StreamProducerFactory
- SerializerFactory
- StreamMetricsCollector
- StreamHealthIndicator

**Step 3: 修复 EnableRedisStream 导入**

Update: `@Import(RedisStreamAutoConfiguration.class)` 指向正确的包

**Step 4: 运行所有测试验证通过**

**Step 5: Commit**

```bash
git add .
git commit -m "feat: 完善自动配置，注册所有必要的 Bean"
```

---

## 阶段五：集成测试

### Task 8: 编写注解驱动的集成测试

**目标：** 验证注解处理器和消费者管理器工作正常

**Files:**
- Create: `freedom-public/redis-stream-spring-boot-starter/src/test/java/com/freedom/stream/integration/AnnotationDrivenIntegrationTest.java`

**Step 1: 编写测试**

Test cases:
- 使用 @StreamListener 消费消息
- 使用 @StreamProducer 发送消息
- 验证消息流转完整性
- 验证并发消费
- 验证重试和死信处理

**Step 2: 运行测试验证通过**

**Step 3: Commit**

```bash
git add .
git commit -m "test: 添加注解驱动的集成测试"
```

---

## 阶段六：文档更新

### Task 9: 更新使用文档

**目标：** 补充注解使用说明

**Files:**
- Modify: `freedom-public/redis-stream-spring-boot-starter/README.md`
- Modify: `freedom-public/redis-stream-spring-boot-starter/docs/USAGE.md`
- Modify: `freedom-public/redis-stream-spring-boot-starter/docs/EXAMPLES.md`

**Step 1: 更新文档**

Add sections:
- 注解处理器工作原理
- @StreamListener 详细用法
- @StreamProducer 详细用法
- 消费者生命周期管理
- 故障排查指南

**Step 2: Commit**

```bash
git add .
git commit -m "docs: 更新注解使用文档"
```

---

## 验证清单

完成所有任务后，验证以下内容：

- [ ] @StreamListener 注解能够自动注册消费者
- [ ] @StreamProducer 注解能够自动注入 StreamTemplate
- [ ] 消费者能够正常启动和消费消息
- [ ] 支持并发消费（concurrency 配置）
- [ ] 重试和死信处理正常工作
- [ ] 所有测试通过
- [ ] 文档完整

---

## 预期时间

- 阶段一：1.5 天
- 阶段二：1 天
- 阶段三：0.5 天
- 阶段四：0.5 天
- 阶段五：0.5 天
- 阶段六：0.5 天

**总计：约 4.5 天**
