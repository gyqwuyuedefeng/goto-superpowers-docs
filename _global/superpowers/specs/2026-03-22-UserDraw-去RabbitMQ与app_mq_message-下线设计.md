# UserDraw 去 RabbitMQ 与 app_mq_message 下线设计

## 背景

当前 `UserDraw` 相关链路处于新旧并存状态：

- 提交主链路已经迁移到 Redis Stream
- 取消链路仍然依赖 RabbitMQ
- 旧的 `app_mq_message` 本地消息表链路仍然残留在定时任务、仓储、回调、监控和若干 `UserDraw` 代码中
- `UserDraw` 清理链路同时存在新旧两套任务

已确认的现状如下：

1. 新的直接删除链路已经存在，并且必须保留：
   - `LongTimeUnusedUserDrawRemoveTask`
   - `UserDrawRemoveExecutor`
2. 旧的清理链路仍然存在：
   - `LongTimeUserDrawMarkAndNotice`
   - `UserDrawTmpFolderRemoveFail`
   - `DispatchInitializer` 启动时仍会补旧延迟删除任务
3. `app_mq_message` 旧链路仍然存在：
   - `MqMessage`
   - `MqMessageRepository`
   - `MessageResendScheduler`
   - `MqMessageScheduledScan`
   - `MqMessageArchiveTask`
   - `MqMessageCleanupScheduler`
4. `UserDraw` 的取消链路仍然依赖 RabbitMQ：
   - `PlotServiceImpl.customCancelDrawPlot()`
   - `UserDrawTaskCancelConsumer`
   - 用户绘图专属 RabbitMQ exchange / queue / template / callback
5. `PlotController` 仍然使用基于 RabbitMQ 队列长度的 `@TotalLimit`

本次设计目标是一次性下线 `UserDraw` 的 RabbitMQ 旧链路和 `app_mq_message` 相关代码，只保留 Redis Stream 和新的直接删除链路。

## 目标

本次设计的目标如下：

1. 下线 `app_mq_message` 整条代码链路
2. 下线 `UserDraw` 的 RabbitMQ 提交、取消、回调、补偿和通知旧链路
3. 保留现有 `UserDraw` 提交 Redis Stream 主链路
4. 将 `UserDraw` 取消能力迁移为独立 Redis Stream 链路
5. 保留 `LongTimeUnusedUserDrawRemoveTask` 和 `UserDrawRemoveExecutor`
6. 删除旧的 `UserDraw` 标记删除、邮件通知、补偿清理、启动补队列逻辑
7. 去掉 `PlotController` 中依赖 RabbitMQ 队列长度的 `@TotalLimit`

## 非目标

本次设计不包含以下内容：

- 不改变现有 `UserDraw` 提交 Redis Stream 的消息结构和消费语义
- 不新增页面、接口或新的前端协议
- 不为取消功能设计第二套通知机制，继续复用现有 `UserDrawNotification` Redis Stream
- 不保留 RabbitMQ 与 Redis Stream 双写兼容期
- 不扩展新的清理策略，继续保留并使用现有 `LongTimeUnusedUserDrawRemoveTask`
- 不改造与 `UserDraw` 无关的通用 RabbitMQ 业务链路

## 方案对比

### 方案一：独立取消流，提交流保持不动

做法：

- 保持 `stream:user_draw_tasks` 提交流不变
- 新增独立的取消流，例如 `stream:user_draw_cancel_tasks`
- `customCancelDrawPlot()` 改为发送取消消息到新 Stream
- 新增取消消费者处理线程池取消、状态落库和通知发送

优点：

- 对已上线的提交流侵入最小
- 提交和取消职责分离，边界清晰
- 旧 RabbitMQ 链路可以整段删除，不需要保留中间兼容层
- 规划和测试范围清楚，风险集中在取消链路

缺点：

- 需要新增一个 Stream 常量、一个消息模型和一个消费者

### 方案二：提交和取消合并到同一条 Redis Stream

做法：

- 给现有 `UserDrawStreamMessage` 增加动作类型
- 由同一个消费者根据动作分支处理提交或取消

优点：

- 基础设施最少

缺点：

- 会修改已经稳定的提交消息契约
- 消费者复杂度上升
- 提交流回归面扩大

### 方案三：取消改为同步直调

做法：

- `customCancelDrawPlot()` 不再发消息
- 直接在服务层获取锁、取消线程池任务、修改状态

优点：

- 表面上代码最少

缺点：

- 行为变化最大
- 取消和任务执行边界重新耦合
- 不符合当前异步任务体系的演进方向

## 方案决策

采用方案一：**独立取消流，提交流保持不动**。

原因：

- 当前提交流已经稳定使用 Redis Stream，没有必要为了取消迁移去改提交消息契约
- 用户已经明确要彻底下线 `UserDraw` 的 RabbitMQ 链路，独立取消流最利于整体收口
- 取消流独立后，后续问题定位、日志排查和测试组织都会更清晰

## 设计概览

### 保留的链路

以下链路必须保留：

1. `UserDraw` 提交 Redis Stream 主链路
2. `UserDrawNotification` Redis Stream 通知链路
3. `LongTimeUnusedUserDrawRemoveTask`
4. `UserDrawRemoveExecutor`

其中第 3 点和第 4 点是本次明确的保留项，不能误删。

### 下线的链路

以下链路整体下线：

1. `app_mq_message` 本地消息表代码链路
2. `UserDraw` RabbitMQ 提交链路残留
3. `UserDraw` RabbitMQ 取消链路
4. `UserDraw` RabbitMQ 通知链路
5. 旧的 `UserDraw` 标记删除 / 邮件通知 / 延迟队列补偿清理链路

## 架构设计

### 提交流

提交流保持不变：

- `PlotServiceImpl.customDrawPlot()` 继续发送到 `stream:user_draw_tasks`
- `UserDrawStreamConsumer` 继续消费提交消息
- 现有 `UserDrawStreamAutoClaimTask` 继续负责抢占超时未 ACK 消息

本次不修改提交流消息模型，不修改其 ACK 语义，不修改其线程池执行主流程。

### 取消流

新增独立取消流，例如：

- Stream：`stream:user_draw_cancel_tasks`
- Group：新增专用消费组，例如 `group:task_cancel_service`

建议新增消息模型：

- `UserDrawCancelStreamMessage`

字段只保留取消真正需要的信息：

- `userDrawId`
- `userId`

不再保留以下旧语义：

- `correlationId`
- 本地消息状态
- 重发计数
- 本地消息表锁定时间

### 通知流

继续复用现有：

- `stream:user_draw_notification`

取消链路成功处理后，直接发送 `UserDrawNotification`，不再通过 RabbitMQ 通知队列中转。

## 组件设计

### `PlotServiceImpl`

#### `customDrawPlot()`

保持现状，不做行为改动。

#### `customCancelDrawPlot()`

改造为：

1. 参数校验
2. 基于现有 `userDraw` 分布式锁执行取消提交
3. 发送 `UserDrawCancelStreamMessage` 到取消流
4. 不再查询 `MqMessageRepository`
5. 不再使用 `RabbitPublishType.taskCancelUser*`
6. 不再使用 `UserDrawTaskRabbitMQUtils`

### 取消消息 Producer

取消消息 producer 建议与提交 producer 一样，直接放在 `PlotServiceImpl` 中通过 `@StreamProducer` 注入。

原因：

- 取消消息目前只由 `PlotServiceImpl` 发出
- 不需要提升到 `ComponentBean` 形成跨模块共享依赖
- 与当前提交流写法保持一致，便于理解

### `UserDrawCancelStreamConsumer`

新增一个取消消费者，放在任务侧，与现有 `UserDrawStreamConsumer` 保持同类职责。

职责如下：

1. 读取取消流消息
2. 根据 `userDrawId` 和 `userId` 查询 `UserDraw`
3. 做幂等判断
4. 调用线程池取消
5. 必要时直接更新数据库状态为 `CANCEL`
6. 发送取消状态通知到 `UserDrawNotification` Redis Stream

### 幂等规则

取消消费者按以下规则处理：

1. 查不到 `UserDraw`
   - 视为幂等完成
   - 输出日志后直接结束
2. `UserDraw` 已处于终态：
   - `SUCCESS`
   - `FAIL`
   - `CANCEL`
   - `REJECTED`
   - 视为幂等完成
   - 不再覆盖状态
3. `UserDraw` 仍可取消：
   - 优先尝试从线程池取消
   - 如果线程池中不存在该任务，但数据库状态仍未进入终态，则直接落库为 `CANCEL`

这样可以避免旧取消链路中“无条件覆盖状态为 CANCEL”的风险。

### 取消成功后的通知

取消成功后，发送一条 `UserDrawNotification` 到现有通知流，通知前端状态变更。

通知复用现有协议：

- `noticeType = STATUS_CHANGE`
- `status = CANCEL`

不新增新的通知模型。

### 通知失败的处理

如果取消已经成功，但发送通知失败：

- 不回滚取消状态
- 记录错误并抛出异常，由 Redis Stream 重试整条取消消息消费

这样即使消息被再次消费，前面的取消逻辑也会按幂等规则安全跳过或重复收敛。

如果最终通知仍失败，任务状态已经以数据库为准，前端仍可通过现有 `get_task_status` 查询兜底。

## 删除范围

### 一、删除旧的 `UserDraw` 清理链路

以下代码删除：

- `LongTimeUserDrawMarkAndNotice`
- `UserDrawTmpFolderRemoveFail`
- `DispatchInitializer` 中补旧延迟删除任务的逻辑
- 对应测试和测试套件引用

### 二、删除 `app_mq_message` 相关代码

以下代码删除：

- `MqMessage`
- `MqMessageRepository`
- `MqMessageEnum`
- `MqMessageResentEnum`
- `MessageResendScheduler`
- `MqMessageScheduledScan`
- `MqMessageArchiveTask`
- `MqMessageCleanupScheduler`
- 所有 `getMqMessageRepository()` 调用点

### 三、删除 `UserDraw` RabbitMQ 提交 / 取消旧链路

以下代码删除：

- `UserDrawMQMessage`
- `UserDrawTaskRabbitMQUtils`
- `ConfirmReturnCallback`
- `MessageIdempotentHelper`
- `UserDrawRabbitTemplateConfig`
- `UserDrawTaskCancelConsumer`
- `RabbitPublishType` 中仅服务 `UserDraw` 的常量与方法
- 用户绘图专属 task notice / task cancel 队列、交换机、绑定和监听容器工厂

### 四、删除旧的 RabbitMQ 通知链路

以下代码删除：

- `UserDrawMessageConsumer`
- 用户绘图专属 message notice 队列、交换机、绑定

### 五、删除 `app_mq_message` 相关监控

以下代码删除或裁剪：

- `SystemMetricsCollector` 中统计 `WAIT_SEND / SENT / RECEIVED / COMPLETED` 的逻辑

保留 JVM 指标收集逻辑，不删除整个类。

## 保留范围

以下代码明确保留：

- `LongTimeUnusedUserDrawRemoveTask`
- `UserDrawRemoveExecutor`
- `UserDrawStreamConsumer`
- `UserDrawStreamAutoClaimTask`
- `UserDrawNotification` Redis Stream 链路
- 与 `UserDraw` 无关的通用 RabbitMQ 任务和消息链路

其中 `LongTimeUnusedUserDrawRemoveTask` 是本次必须保留的清理任务，不属于删除范围。

## 定向改造范围

### `DoUserDrawTask`

`DoUserDrawTask` 不能直接删除。

原因：

- 当前 Redis Stream 执行链路里仍然有类对其进行复用

本次应做的不是“整类删除”，而是：

- 删除其中所有 `MqMessage` / RabbitMQ 专属语义
- 保留仍被 Redis Stream 链路使用的失败通知能力
- 本次不强制继续拆分类结构，优先以“保留类并瘦身”为实现方向，避免扩大范围

### `ComponentBean`

需要：

- 删除 `userDrawRabbitTemplate`
- 保留 `commonRabbitTemplate`
- 保留现有 `userDrawNotificationProducer`

本次不在 `ComponentBean` 新增取消流 producer。

原因：

- 取消流 producer 已在设计中明确由 `PlotServiceImpl` 通过 `@StreamProducer` 持有
- `ComponentBean` 只保留跨模块共享的公共 producer，避免新增不必要的共享依赖

### `ReturnCallback`

`ReturnCallback` 属于通用 RabbitMQ 配置，不应随本次一起删除。

但当前实现中存在对 `userDrawRabbitTemplate` 的误依赖，用户绘图专属 template 删除后，这里需要顺手改为不依赖该字段的实现方式。

### `PlotController`

需要删除以下旧限流逻辑：

- `customDrawPlot()` 上依赖 RabbitMQ 队列长度的 `@TotalLimit`
- `batchTestCustomDrawPlot()` 上依赖 RabbitMQ 队列长度的 `@TotalLimit`

保留：

- `@RateLimit`
- `@Debounce`

本次不扩展新的 Redis Stream 总限流方案，避免额外扩大范围。

## 数据与兼容策略

### 数据库表

代码层面不再允许任何模块继续访问 `app_mq_message`。

如果仓库内存在数据库迁移脚本，则应在实施阶段同步删除或废弃该表相关脚本。

如果仓库内不存在数据库迁移脚本，则应在实施计划中明确加入一条运维/DBA 操作：

- 确认线上已无代码访问 `app_mq_message`
- 再执行表下线

### 接口兼容

以下接口签名保持不变：

- `customDrawPlot`
- `customCancelDrawPlot`
- `get_task_status`

本次只修改其内部实现，不改变接口出参与调用方式。

### 灰度策略

本次不保留 RabbitMQ / Redis Stream 双写或双读兼容逻辑。

原因：

- 用户已明确要求彻底下线 `UserDraw` 的 RabbitMQ 链路
- 当前目标是清理死代码和旧依赖，而不是做长期共存方案

## 日志设计

### 取消消息日志

至少输出以下日志：

- 收到取消消息：`userDrawId`、`userId`、`messageId`
- 幂等跳过原因：不存在 / 已终态
- 取消执行结果：线程池取消成功 / 直接落库取消
- 通知发送结果

### 删除链路日志

继续保留现有 `LongTimeUnusedUserDrawRemoveTask` 的任务开始、结束汇总和单条失败日志。

## 测试设计

至少包含以下测试：

### 一、取消流测试

1. `customCancelDrawPlot()` 发送取消消息成功
2. 取消消费者处理正常取消成功
3. 取消消费者处理“任务不存在”幂等成功
4. 取消消费者处理“任务已成功 / 已失败 / 已取消”幂等成功
5. 线程池中不存在任务但数据库状态可取消时，能够直接落库为 `CANCEL`
6. 取消成功后，能够发送 `UserDrawNotification`

### 二、提交流回归测试

1. 提交 Redis Stream 主链路保持可用
2. `UserDrawStreamConsumer` 不受本次删除影响
3. `UserDrawStreamAutoClaimTask` 不受本次删除影响

### 三、清理链路测试

1. 删除旧的 `LongTimeUserDrawMarkAndNotice` / `UserDrawTmpFolderRemoveFail` 测试
2. 保留并通过 `LongTimeUnusedUserDrawRemoveTask` 相关测试
3. 验证保留项没有被误删

### 四、编译与启动回归

至少验证：

1. `goto/common`
2. `goto/system`
3. `goto/task`

在删除 `MqMessage` 相关依赖后均能正常编译。

同时验证 Spring 启动阶段不存在以下问题：

- 已删除 Bean 仍被注入
- 已删除 RabbitMQ 配置仍被引用
- 已删除仓储仍被调用

## 风险与注意事项

### 风险一：误删新的直接删除链路

本次最需要防止的误操作是把新的直接删除链路一起删掉。

必须明确：

- 删除的是 `LongTimeUserDrawMarkAndNotice`
- 删除的是 `UserDrawTmpFolderRemoveFail`
- 保留的是 `LongTimeUnusedUserDrawRemoveTask`
- 保留的是 `UserDrawRemoveExecutor`

### 风险二：遗漏 `MqMessageRepository` 引用

`app_mq_message` 的引用已经扩散到服务、回调、定时任务、监控和消费者中。

实施前必须先做全局搜索，确保所有访问点都被纳入修改范围。

### 风险三：删除用户绘图专属 RabbitMQ 后遗留旧常量和限流配置

如果只删生产者/消费者，不删：

- `RabbitMQConstant`
- `RabbitmqExchangeQueueConfig`
- `ListenerConfiguration`
- `PlotController` 上的 `@TotalLimit`

则会留下编译错误或逻辑死代码。

## 实施顺序建议

建议按以下顺序实施：

1. 新增取消流常量、消息模型和取消消费者
2. 改造 `customCancelDrawPlot()` 为 Redis Stream 取消流
3. 删除 `app_mq_message` 相关调用点
4. 删除 `MqMessage` 领域、仓储、定时任务、回调、帮助类
5. 删除用户绘图专属 RabbitMQ 配置、消费者、常量和旧通知链路
6. 删除旧清理任务与启动补队列逻辑
7. 删除 `@TotalLimit` 和消息监控旧逻辑
8. 跑编译与测试，确认保留项仍然存在

## 结论

本次设计采用“保留 Redis Stream 提交流 + 新增独立取消流 + 整体下线 RabbitMQ 与 `app_mq_message` 旧链路”的方案。

实施完成后，`UserDraw` 将只保留三类有效链路：

1. 提交：Redis Stream
2. 通知：Redis Stream
3. 清理：`LongTimeUnusedUserDrawRemoveTask`

旧的 RabbitMQ、`app_mq_message`、标记删除、邮件通知、延迟队列补偿链路全部退出代码主路径。
