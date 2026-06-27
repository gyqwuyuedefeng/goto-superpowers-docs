# Redis Stream 长任务稳定性与通知一致性设计

## 背景

goto 后端计划将 RabbitMQ 异步链路统一替换为 Redis Stream。统计分析、R 脚本执行、文件转换等任务属于长任务，不能只依赖 Redis Stream 的 Pending List 判断业务状态，也不能用一个长数据库事务包住完整执行过程。

本设计约定：Redis Stream 负责消息至少一次投递，数据库 Job Ledger 负责业务任务执行权和状态机，幂等策略负责重复投递下的副作用控制，通知链路只发布当前有效执行版本对应的状态变化。

## 目标

- 支持 `PENDING`、`RUNNING`、`SUCCEEDED`、`FAILED_RETRYABLE`、`FAILED_FINAL`、`CANCELLED` 六类长任务状态。
- 服务实例宕机、卡顿、网络中断、续期失败后，任务不会永久卡在 `RUNNING`。
- 同一任务不会被多个 task 服务实例同时提交成功结果。
- 旧执行线程失权后不能写成功、不能写失败、不能发前端通知。
- Redis Stream 至少一次投递导致重复消费时，业务副作用保持幂等。
- WebSocket 通知不会因旧 owner、乱序消息或重复消息导致前端状态混乱。

## 三种幂等策略

### Redis 默认策略

适用于短消息、普通通知、允许用 Redis 标记证明“已处理”的轻量场景。

执行模型：

```text
解析 idempotencyKey
→ 获取 Redis 分布式锁
→ 检查 completed 标记
→ 执行业务
→ 写 completed 标记
→ ACK
→ 释放锁
```

该策略能防止并发重复执行和短期重复投递，但不能证明数据库事务、文件结果或外部 R 进程已经真实完成。因此它不能作为长任务的唯一可靠依据。

### Transactional Inbox 策略

适用于短数据库事务，例如创建任务记录、取消任务、更新数据库状态。

核心约束：幂等记录和业务数据修改必须在同一个数据库事务内提交。事务提交成功后再 ACK Redis Stream 消息。

典型表结构：

```sql
create table stream_message_inbox (
    id bigint primary key auto_increment,
    consumer_name varchar(128) not null,
    idempotency_key varchar(200) not null,
    message_id varchar(100) not null,
    status varchar(32) not null,
    result_json json null,
    error_message text null,
    created_at datetime not null,
    completed_at datetime null,
    unique key uk_consumer_idempotency (consumer_name, idempotency_key)
);
```

取消任务示例：

```text
开启事务
→ insert stream_message_inbox，唯一键为 statistics:cancel:job-10001
→ update statistics_job set status = CANCELLED where status not in terminal states
→ update stream_message_inbox set status = COMPLETED
→ 提交事务
→ XACK
```

如果数据库提交成功但 ACK 前服务宕机，消息会再次投递。第二次消费插入 inbox 时触发唯一键冲突，查询发现状态已完成，直接 ACK，不重复执行业务修改。

### Job Ledger 策略

适用于统计分析、R 脚本、文件转换、图片处理等长任务。

长任务不能持有几十分钟数据库事务。它应先通过短事务认领任务，再在事务外执行任务，执行期间定期续期，最终通过短事务提交成功、失败或取消状态。

典型表结构：

```sql
create table stream_job_execution (
    id bigint primary key auto_increment,
    consumer_name varchar(128) not null,
    idempotency_key varchar(200) not null,
    message_id varchar(100) not null,
    business_type varchar(64) not null,
    business_id varchar(128) not null,
    status varchar(32) not null,
    owner_id varchar(128) null,
    worker_id varchar(128) null,
    fencing_token bigint not null default 0,
    lease_until datetime null,
    heartbeat_at datetime null,
    retry_count int not null default 0,
    status_version bigint not null default 0,
    progress int null,
    progress_version bigint not null default 0,
    result_json json null,
    error_message text null,
    created_at datetime not null,
    started_at datetime null,
    completed_at datetime null,
    unique key uk_consumer_idempotency (consumer_name, idempotency_key)
);
```

## owner_id、worker_id 与 fencing_token

`owner_id` 表示拥有任务执行权的 task 服务实例，不表示线程。每次服务启动都应生成新的实例标识，避免重启后误继承旧执行权。

示例：

```text
owner_id = goto-task:node-b:boot-uuid-002
worker_id = statistics-executor-4
fencing_token = 2
```

`worker_id` 只用于排查实例内部哪个线程执行任务。跨实例执行权判断必须依赖 `owner_id + fencing_token`。

`fencing_token` 是每次认领或接管任务时递增的执行版本。执行线程在认领成功时把当时的 token 保存到内存上下文，后续续期、进度、成功、失败、ACK 前校验都必须使用该旧 token，不能重新查询并套用数据库中的最新 token。

## 任务认领与接管

认领任务必须使用数据库条件更新，只有更新成功的服务实例拥有执行权。

```sql
update stream_job_execution
set status = 'RUNNING',
    owner_id = :ownerId,
    worker_id = :workerId,
    lease_until = :leaseUntil,
    heartbeat_at = :now,
    fencing_token = fencing_token + 1,
    status_version = status_version + 1,
    started_at = coalesce(started_at, :now)
where idempotency_key = :idempotencyKey
  and (
      status in ('PENDING', 'FAILED_RETRYABLE')
      or (status = 'RUNNING' and lease_until < :now)
  );
```

当 A 因 GC、网络中断、数据库连接池阻塞、容器暂停或机器卡顿导致未能续期时，`lease_until` 到期后 B 可以接管。B 接管后 `fencing_token` 递增。A 后续即使恢复，也只能拿旧 token 提交状态，数据库条件更新会失败。

## 自动续期与超时判断

执行线程启动后，配套续期器定期更新租约。续期必须绑定认领时的 `owner_id + fencing_token`。

```sql
update stream_job_execution
set lease_until = :nextLeaseUntil,
    heartbeat_at = :now
where id = :jobExecutionId
  and owner_id = :ownerId
  and fencing_token = :fencingToken
  and status = 'RUNNING';
```

判断任务是否超时应结合三类信号：

- `lease_until < now`：业务执行权租约已过期，可被接管。
- Redis Stream PEL idle 超过阈值：消息长时间未 ACK，可通过 `XAUTOCLAIM` 重新投递。
- `heartbeat_at` 过旧：用于监控告警和辅助排查。

Redis Stream 的 Pending 只表示消息未 ACK。数据库 Job Ledger 的 `RUNNING` 才表示业务执行权归属。即使 `XAUTOCLAIM` 抢到了消息，只要 Job Ledger 显示其他 owner 的租约仍有效，新消费者也不能执行业务。

## 状态机

允许状态流转：

```text
PENDING → RUNNING
PENDING → CANCELLED
RUNNING → SUCCEEDED
RUNNING → FAILED_RETRYABLE
RUNNING → FAILED_FINAL
RUNNING → CANCELLED
RUNNING → RUNNING     // 续期、心跳、进度更新
FAILED_RETRYABLE → PENDING
FAILED_RETRYABLE → RUNNING
```

终态：

```text
SUCCEEDED
FAILED_FINAL
CANCELLED
```

终态不可被旧 owner 或旧通知覆盖。确实需要重新运行时，应创建新的 `idempotencyKey` 或新的任务 attempt，不应把终态原地改回 `RUNNING`。

状态含义：

- `PENDING`：任务已登记，等待执行或等待重试。
- `RUNNING`：某个服务实例已认领任务，租约未过期时视为正在运行。
- `SUCCEEDED`：业务结果已确认落地，消息可以 ACK。
- `FAILED_RETRYABLE`：失败但允许按策略重试。
- `FAILED_FINAL`：失败且不再重试，例如参数非法、输入缺失或超过最大重试次数。
- `CANCELLED`：用户或系统取消，任务不应继续执行。

## 失权处理

长任务执行器必须定期检查自己是否仍持有执行权：

```text
owner_id == context.ownerId
fencing_token == context.fencingToken
status == RUNNING
```

检查失败时，执行线程必须：

- 停止 R 子进程或停止后续步骤。
- 清理当前 token 私有临时目录。
- 不写 `SUCCEEDED`。
- 不写用户可见失败。
- 不发 WebSocket 通知。
- 不 ACK 原 Redis Stream 消息。

旧 owner 失权不是用户可见失败，只是内部执行权转移。前端应继续看到任务运行中，直到当前有效 owner 或调度恢复器写入最终状态。

## 结果文件幂等

长任务输出应写入 token 私有临时目录，避免多个执行版本互相覆盖。

```text
/tmp/statistics/job-10001/token-1/result.csv
/tmp/statistics/job-10001/token-2/result.csv
/final/statistics/job-10001/result.csv
```

只有当前 `owner_id + fencing_token` 条件更新成功的执行者，才有资格发布最终结果。若最终文件已生成但 DB 尚未写 `SUCCEEDED` 时服务宕机，重试方应通过最终路径、checksum、结果元数据判断结果是否已完成，必要时补写 `SUCCEEDED` 并 ACK。

## 通知一致性

通知不能绕过 Job Ledger。执行线程发任何通知前，必须先成功写入对应状态、进度或版本。

正确顺序：

```text
生成进度
→ 条件更新 progress/status_version 成功
→ 发送 progressChanged 事件

生成结果
→ 条件更新 SUCCEEDED 成功
→ 发送 jobSucceeded 事件
→ XACK
```

条件更新失败表示线程已失权或任务已终止，此时不能通知前端。

通知事件建议携带：

```json
{
  "eventId": "uuid",
  "taskId": "job-10001",
  "businessType": "statistics-pca",
  "eventType": "PROGRESS_CHANGED",
  "status": "RUNNING",
  "progress": 60,
  "ownerId": "goto-task:node-b:boot-uuid-002",
  "fencingToken": 2,
  "statusVersion": 8,
  "occurredAt": "2026-06-27T10:08:00"
}
```

system 服务消费通知事件时，不应盲信事件中的状态内容。推荐把通知事件视为“某 taskId 发生变化”，再查询数据库最新快照：

```text
收到通知事件
→ 查询 Job Ledger / 业务任务表最新状态
→ event.fencingToken 小于 DB 当前 token 时丢弃
→ event.statusVersion 小于 DB 当前 version 时丢弃
→ 推送 DB 最新快照
```

前端也应缓存每个任务的最新 `statusVersion`，收到更旧版本的 WebSocket 消息时直接忽略，避免网络乱序造成进度回退或终态被覆盖。

## ACK 顺序

长任务使用 MANUAL ACK。ACK 必须发生在业务终态成功写入后。

```text
认领任务
→ 执行业务
→ 写 SUCCEEDED / FAILED_FINAL / CANCELLED
→ 发送当前版本通知事件
→ XACK
```

不能在提交线程池成功后 ACK，也不能在数据库事务提交前 ACK。否则线程异常、事务失败或服务宕机时，Redis Stream 不再重投，任务可能永久丢失。

## 示例：A 卡顿后 B 接管

```text
10:00 A 认领任务，owner_id=node-a，token=1，lease_until=10:05
10:02 A JVM 卡顿或数据库网络中断，续期失败
10:06 B 发现 lease 过期并接管，owner_id=node-b，token=2，lease_until=10:11
10:07 A 旧执行线程恢复，尝试写 SUCCEEDED
```

A 写成功时仍使用认领时保存的 token=1：

```sql
update stream_job_execution
set status = 'SUCCEEDED'
where id = :jobExecutionId
  and owner_id = 'node-a'
  and fencing_token = 1
  and status = 'RUNNING';
```

数据库当前已是 `owner_id=node-b`、`fencing_token=2`，所以 A 更新 0 行。A 必须停止、清理临时结果、不通知、不 ACK。只有 B 的 token=2 有资格提交结果和推动前端状态。

## 验收标准

- 重复投递同一消息时，已 `SUCCEEDED` 的任务不会再次执行业务。
- `RUNNING` 任务租约过期后，其他实例可以接管；租约未过期时不能接管。
- 旧 owner 使用旧 token 提交成功、失败、进度或通知时，均被拒绝。
- 服务宕机导致 Redis Stream 消息未 ACK 时，消息可通过 PEL idle 和 `XAUTOCLAIM` 恢复。
- 线程异常或续期停止后，任务不会永久卡在 `RUNNING`。
- 旧 owner 失权后不能发送用户可见通知。
- system 推送 WebSocket 前以数据库最新状态为准，旧 token 或旧 version 的通知被丢弃。
- 前端收到乱序 WebSocket 消息时，不会用旧 `statusVersion` 覆盖新状态。

## 后续计划边界

本设计只定义 Redis Stream 长任务稳定性、幂等状态机和通知一致性规则。具体实施计划需要进一步拆分 Starter 能力、task 消费器改造、system 通知消费改造、前端版本过滤、测试用例和 RabbitMQ 删除步骤。
