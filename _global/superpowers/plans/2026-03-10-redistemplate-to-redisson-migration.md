# RedisTemplate 到 Redisson 迁移设计

## 概述

**目标**：将三个测试和监控文件中的 `RedisTemplate<String, Object>` 替换为 Redisson 的 RStream API

**影响范围**：
- `RedisStreamMetrics.java` - 监控指标收集
- `RedisStreamIntegrationTest.java` - 集成测试
- `RedisStreamPerformanceTest.java` - 性能测试

**技术栈**：Redisson 4.3.0 (redisson-spring-boot-starter)

---

## 设计方案

### 方案选择

采用 **Redisson RStream API** 直接替换 Spring Data Redis 的 RedisTemplate。

**理由**：
1. API 简洁，与 Spring Data Redis 类似
2. 类型安全，支持泛型
3. 性能优秀，Redisson 内部优化良好
4. 代码改动最小，风险最低

---

## API 映射关系

| Spring Data Redis | Redisson RStream |
|-------------------|------------------|
| `redisTemplate.opsForStream().add()` | `rStream.add()` |
| `redisTemplate.opsForStream().read()` | `rStream.read()` |
| `redisTemplate.opsForStream().acknowledge()` | `rStream.ack()` |
| `redisTemplate.opsForStream().pending()` | `rStream.getPendingInfo()` |
| `redisTemplate.opsForStream().size()` | `rStream.size()` |
| `redisTemplate.opsForStream().autoclaim()` | `rStream.autoClaim()` |

---

## 具体修改

### 1. RedisStreamMetrics.java

**依赖注入变更**：
```java
// 修改前
@Autowired
private RedisTemplate<String, Object> redisTemplate;

// 修改后
@Autowired
private RedissonClient redissonClient;
```

**方法修改**：
- `monitorPendingMessages()`: 使用 `rStream.getPendingInfo(group)` 获取 PEL 信息
- `monitorStreamLength()`: 使用 `rStream.size()` 获取 Stream 长度

---

### 2. RedisStreamIntegrationTest.java

**依赖注入变更**：同上

**测试方法修改**：
- `testMessageSendAndConsume()`: 使用 `rStream.add()` 发送消息
- `testTimeoutAutoclaim()`: 使用 `rStream.read()` 和 `rStream.autoClaim()`
- `testDeadLetterHandling()`: 使用 `rStream.add()` 和 `rStream.ack()`
- `testIdempotency()`: 使用 `rStream.add()` 发送消息

---

### 3. RedisStreamPerformanceTest.java

**依赖注入变更**：同上

**测试方法修改**：
- `testSendLatency()`: 使用 `rStream.add()` 测试发送延迟
- `testConsumeThroughput()`: 使用 `rStream.add()` 批量发送消息
- `testConcurrentSend()`: 使用 `rStream.add()` 并发发送消息

---

## 关键注意事项

1. **消息 ID 类型**：
   - Spring Data Redis: `RecordId`
   - Redisson: `StreamMessageId`
   - 需要适配类型转换

2. **消息体格式**：
   - 两者都使用 `Map<String, Object>`，无需修改

3. **Consumer Group 操作**：
   - Redisson 的 Consumer Group 创建和管理方式略有不同
   - 需要使用 `rStream.createGroup()` 创建消费组

4. **阻塞读取**：
   - Redisson 的 `read()` 方法支持超时参数
   - 需要适配阻塞时间设置

5. **异常处理**：
   - Redisson 的异常类型可能不同
   - 需要更新 catch 块中的异常类型

---

## 实施步骤

1. 修改 `RedisStreamMetrics.java`
2. 修改 `RedisStreamIntegrationTest.java`
3. 修改 `RedisStreamPerformanceTest.java`
4. 运行测试验证功能正常
5. 提交代码

---

## 验证清单

- [ ] 编译成功
- [ ] 单元测试通过
- [ ] 集成测试通过
- [ ] 性能测试通过
- [ ] 监控指标正常收集

---

## 回滚方案

如果迁移后出现问题，可以：
1. 回退代码到迁移前的版本
2. 重新注入 `RedisTemplate<String, Object>`
3. 恢复原有的 Spring Data Redis API 调用
