---
title: "Task 持久化任务"
weight: 31
---

# LoadUp Task

单一持久化任务模块。事务内提交工作，提交后立即或按计划执行；消息投递、业务截止时间和恢复轮询共享一张队列表与一套执行引擎。

## 接入

引入 `io.github.loadup-cloud:loadup-components-task`，将所选数据库的 `db/task/pg/schema.sql` 或 `db/task/mysql/schema.sql` 纳入应用的初始迁移。业务写入与任务提交必须使用同一 DataSource、同一 Spring JDBC 事务管理器。

```java
@Transactional
public void accept() {
    orders.save(order);
    tasks.submit(TaskSubmission.immediate("order.notify", deliveryId, frozenPayload));
    tasks.submit(TaskSubmission.scheduled("order.expire", order.id(), order.dueAt()));
}
```

`TaskSubmission` 的完整字段是 taskType、bizId、payload、executeAt。executeAt 为空表示尽快执行。payload 可以为空，最大 1 MiB，不应包含凭证。业务需要版本时由业务载荷或任务类型表达。队列统一按 `(taskType, bizId)` 去重。bizId 标识本次工作。多代任务可使用业务 ID 加代数，事件投递可使用投递 ID。同一业务的不同工作应使用不同的 bizId。

```java
@Component
public class ExpireHandler implements TaskHandler {
    public String taskType() { return "order.expire"; }
    public void handle(TaskExecution task) {
        orders.closeIfUnpaidAndExpired(task.bizId());
    }
}
```

Handler 在领取事务提交后执行；自行调用业务事务并检查业务状态，外部调用设置超时。正常返回表示成功，异常触发技术重试。业务仍未完成时登记下一代任务，不把下一轮业务轮询视为本次技术失败。

## 配置

```yaml
loadup:
  task:
    enabled: true
    db-type: pg
    table-prefix: loadup_
    worker-enabled: true
    poll-interval: PT1S
    batch-size: 20
    max-attempts: 8
    lease: PT2M
    retry-delay: PT5S
    types:
      "[order.expire]":
        max-attempts: 5
        lease: PT5M
        retry-delay: PT1M
        exponential: false
```

`db-type` 选择 `pg` 或 `mysql`，默认 `pg`。MySQL 使用 8.0.17 及以上版本和 InnoDB。运行 SQL 按方言从 `db/task/<db-type>/queries.sql` 加载。配置只选择 SQL，数据库连接仍由应用提供。

`table-prefix` 默认是 `loadup_`，应用于 `task` 和 `task_operation`。前缀可留空，非空时以小写字母开头，只含小写字母、数字和下划线，最多 49 个字符。初始化脚本使用默认前缀。自定义时，先在应用的迁移中替换表及索引的 `loadup_` 前缀，再将同一值配置到 `loadup.task.table-prefix`。已有表的前缀变更由应用迁移处理。

worker-enabled=false 仍允许登记和管理任务，独立执行节点可调用 TaskDispatcher.execute()。`loadup.task.task-type` 可限定本执行节点领取的任务类型，用于隔离慢任务与关键投递的执行容量；未设置时领取所有类型。Task 自带统一轮询，不再依赖 Scheduler binder；Scheduler 可继续用于其他扫描器。所有后台访问使用应用固定 schema，不恢复请求身份。

## 管理

submit 返回任务 ID；find(id) 查询完整任务，find(type,bizId) 返回最新登记项，仅供业务定位。cancel(id) 支持 PENDING/RUNNING/FAILED；reschedule(id,time) 支持 PENDING/FAILED/CANCELLED；retryFailed(id,operator,reason) 只接受 FAILED，保留任务 ID、payload 和原计划时间并记录人工操作。

`types` 可省略或配置为空。默认最多尝试 8 次（含首次执行），租约 2 分钟，重试从 5 秒开始按指数退避，最长 24 小时。按类型可覆盖 `max-attempts`、`lease`、`retry-delay`、`exponential`，部分覆盖继承全局默认值。`exponential: false` 使用固定间隔。

提交时把初始计划写入 `next_execute_at`，同时保存 `max_attempts`。领取时按当前类型配置计算 `lease_until`。失败且还有次数时，按当前类型配置更新 `next_execute_at`；次数耗尽时进入 FAILED。状态变更更新 `updated_at`，成功确认直接删除任务行。租约与退避以数据库时钟为准。业务截止时间由业务记录保存。应用与数据库会话应使用一致的时区。

## 能力矩阵

| 能力 | 契约 |
|---|---|
| 事务一致性 | 登记强制加入同数据源的可写业务事务 |
| 去重 | 同一任务类型与业务 ID 返回同一任务 ID，载荷不同则冲突。重复提交保留当前执行时间，需要改期时调用 reschedule |
| 立即/定时执行 | 同一提交接口；到期后领取，不保证精确触发时刻 |
| 多实例与崩溃恢复 | SKIP LOCKED + claim token + 有效租约确认 |
| 失败恢复 | 有限指数退避、人工重试审计、取消与改期 |
| 完成清理 | 有效领取者成功确认后直接删除任务行 |
| 执行保证 | 至少一次，无顺序或恰好一次保证 |
| 续租、归档 | 尚未提供；Handler 必须幂等，租约覆盖正常处理时间 |

查询结果包含任务 payload，应用负责授权和响应裁剪；日志及 toString 隐藏 payload。组件不提供管理 Controller。成功确认后直接删除任务，查询不再返回该行，提交去重键也随之释放。需要长效幂等的业务通过自己的唯一约束或已提交结果识别重复处理。处理成功至确认删除之间如果发生崩溃，任务仍可能重新执行。失败、取消的任务保留，供人工重试或改期；人工重试日志仍保留。

本次重构未运行构建、测试或部署验收。
