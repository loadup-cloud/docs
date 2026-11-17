---
title: "Task 持久化任务"
weight: 31
---

# LoadUp Task

单一持久化任务模块。事务内提交工作，提交后立即或按计划执行；消息投递、业务截止时间和恢复轮询共享一张队列表与一套执行引擎。

## 接入

引入 `io.github.loadup-cloud:loadup-components-task`，将 `db/schema/task.sql` 纳入应用的 Flyway 初始迁移。新建部署使用单一固定 schema。业务写入与任务提交必须使用同一 DataSource、同一 Spring JDBC 事务管理器。

```java
@Transactional
public void accept() {
    orders.save(order);
    tasks.submit(TaskSubmission.immediate("order.notify", 1, order.id(), frozenPayload));
    tasks.submit(TaskSubmission.scheduled("order.expire", order.id(), order.dueAt()));
}
```

`TaskSubmission` 的完整字段是 taskType、payloadVersion、bizId、payload、executeAt、dedupKey。executeAt 为空表示尽快执行。payload 可以为空，最大 1 MiB，不应包含凭证。scheduled 工厂显式使用类型和业务 ID 组合的去重键；需要多代任务时把 generation 编入业务 ID。immediate 工厂不去重。直接构造可自定义去重键。

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

worker-enabled=false 仍允许登记和管理任务，独立执行节点可调用 TaskDispatcher.execute()。`loadup.task.task-type` 可限定本执行节点领取的任务类型，用于隔离慢任务与关键投递的执行容量；未设置时领取所有类型。Task 自带统一轮询，不再依赖 Scheduler binder；Scheduler 可继续用于其他扫描器。所有后台访问使用应用固定 schema，不恢复请求身份。

## 管理

submit 返回任务 ID；find(id) 查询完整任务，find(type,bizId) 返回最新登记项，仅供业务定位。cancel(id) 支持 PENDING/RUNNING/FAILED；reschedule(id,time) 支持 PENDING/FAILED/CANCELLED；retryFailed(id,operator,reason) 只接受 FAILED，保留任务 ID、payload 和原计划时间并记录人工操作。

重试默认采用有限指数退避，最长 24 小时；`loadup.task.types.<taskType>` 可覆盖 max-attempts、lease、retry-delay、exponential，exponential=false 使用固定间隔；任务登记时快照最大次数、基础间隔和租约。executeAt 保存业务计划，nextExecuteAt 保存下次尝试时间。租约判断、领取、恢复和重试时间统一使用数据库时钟。日期沿用项目系统本地时区契约，应用与数据库会话保持一致。

## 能力矩阵

| 能力 | 契约 |
|---|---|
| 事务一致性 | 登记强制加入同数据源的可写业务事务 |
| 去重 | 可选 dedupKey，重复内容或明确计划时间不同则冲突 |
| 立即/定时执行 | 同一提交接口；到期后领取，不保证精确触发时刻 |
| 多实例与崩溃恢复 | SKIP LOCKED + claim token + 有效租约确认 |
| 失败恢复 | 有限指数退避、人工重试审计、取消与改期 |
| 消费端去重 | IdempotentEventConsumer 的标识与本地数据库工作同事务 |
| 执行保证 | 至少一次，无顺序或恰好一次保证 |
| 续租、归档 | 尚未提供；Handler 必须幂等，租约覆盖正常处理时间 |

查询结果包含任务 payload，应用负责授权和响应裁剪；日志及 toString 隐藏 payload。组件不提供管理 Controller。成功任务保持终态；再次发起工作创建新任务。去重记录与 Inbox 的保留期必须覆盖允许的业务重放窗口。

本次重构未运行构建、测试或部署验收。
