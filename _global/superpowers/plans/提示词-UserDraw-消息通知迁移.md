# 提示词：UserDraw 消息通知 Redis Stream 迁移

## 简短版（适合快速沟通）

```
UserDraw 功能当前状态：
- 任务提交：已使用 Redis Stream ✅
- 任务消费：已使用 Redis Stream ✅
- 消息通知：仍在使用 RabbitMQ ❌

需要完成：将消息通知从 RabbitMQ 迁移到 Redis Stream

涉及文件：
1. DoUserDrawTask.java (第 249-253 行) - 发布端
2. UserDrawMessageConsumer.java - 消费端
3. 其他所有使用 QUEUE_USER_DRAW_MESSAGE_NOTICE 的地方

目标：实现完整的 Redis Stream 消息流，移除 RabbitMQ 依赖
```

## 详细版（适合技术讨论）

```
## 背景

UserDraw 功能已部分迁移到 Redis Stream：
- ✅ 任务消费：已使用 Redis Stream 消费者处理任务
- ✅ 任务提交：已使用 Redis Stream 发布任务
- ❌ 消息通知：仍在使用 RabbitMQ 发布和消费通知消息

## 问题

发送任务请求使用的是 Redis Stream，但是任务通知返回使用的是 RabbitMQ。

当前流程：
```
PlotServiceImpl
  → Redis Stream (发布任务)
  → UserDrawStreamConsumer (消费任务)
  → UserDrawRunnable (执行任务)
  → RabbitMQ (发布通知) ← 这里需要改
  → UserDrawMessageConsumer (消费通知) ← 这里需要改
  → WebSocket (推送到前端)
```

## 目标

将消息通知从 RabbitMQ 迁移到 Redis Stream：

```
PlotServiceImpl
  → Redis Stream (发布任务)
  → UserDrawStreamConsumer (消费任务)
  → UserDrawRunnable (执行任务)
  → Redis Stream (发布通知) ← 改为 Redis Stream
  → UserDrawNotificationStreamConsumer (消费通知) ← 新建消费者
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

// 需要改为（使用 Redis Stream）
redisStreamTemplate.add(
    RedisStreamConstant.STREAM_USER_DRAW_NOTIFICATION,
    messageContainer
);
```

**其他发布位置**：
- UserDrawRunnable.java - 任务执行过程中的状态通知
- UserDrawCommonUtils.java - 状态变更通知
- 所有调用 QUEUE_USER_DRAW_MESSAGE_NOTICE 的地方

### 2. 消息消费端

**UserDrawMessageConsumer.java**
- 当前：RabbitMQ 消费者，监听 QUEUE_USER_DRAW_MESSAGE_NOTICE
- 需要：创建 Redis Stream 消费者，监听 STREAM_USER_DRAW_NOTIFICATION

**功能要求**：
- 接收 UserDraw 状态变更通知
- 通过 WebSocket 推送到前端
- 处理消息重试和失败
- 保持前端接口不变

## 实施步骤

1. 设计 Redis Stream 通知机制（Stream Key、消息格式、消费者配置）
2. 实现 Redis Stream 通知发布（创建发布工具类，修改所有发布位置）
3. 实现 Redis Stream 通知消费（创建消费者，集成 WebSocket）
4. 测试和验证（功能测试、性能测试、异常测试）
5. 清理 RabbitMQ 代码（禁用消费者、删除相关代码）

## 关键技术点

- Redis Stream 消息确认：使用 XACK
- 消息重试：使用 XPENDING + XCLAIM
- 消息顺序：Redis Stream 天然有序
- WebSocket 集成：复用现有的 UserDrawWebsocket.notice() 方法

## 风险和缓解

- 风险：消息丢失 → 缓解：实现消息持久化和确认机制
- 风险：通知延迟 → 缓解：性能测试，优化消费者配置
- 风险：回滚困难 → 缓解：保留 RabbitMQ 代码，灰度发布

## 预期结果

- 完整的 Redis Stream 消息流
- 移除 UserDraw 对 RabbitMQ 的依赖
- 保持前端接口不变
- 性能和稳定性不降低
```

## 超简版（适合任务描述）

```
任务：UserDraw 消息通知迁移到 Redis Stream

当前：任务用 Redis Stream，通知用 RabbitMQ
目标：通知也改用 Redis Stream

主要工作：
1. DoUserDrawTask.java - 改发布逻辑
2. UserDrawMessageConsumer.java - 改消费逻辑
3. 测试验证
4. 删除 RabbitMQ 代码

详细计划见：docs/plans/2026-03-17-UserDraw-消息通知-Redis-Stream-迁移.md
```

---

## 使用建议

- **快速沟通**：使用简短版
- **技术讨论**：使用详细版
- **任务分配**：使用超简版
- **实施开发**：参考完整计划文档
