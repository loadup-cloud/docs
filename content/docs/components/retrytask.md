# LoadUp Components :: Retry Task

[![Java](https://img.shields.io/badge/Java-21-blue.svg)](https://openjdk.org/projects/jdk/21/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1.0-green.svg)](https://spring.io/projects/spring-boot)

Retry Task 是一个基于 Spring Boot 的**分布式重试任务框架**，用于处理异步任务失败后的自动重试逻辑，支持灵活的重试策略、优先级管理和失败通知。

## 核心特性

- ✅ **分布式任务重试** - 支持分布式环境下的任务重试
- ✅ **灵活的重试策略** - 固定间隔、指数退避、自定义策略
- ✅ **优先级管理** - 支持高优先级和低优先级任务
- ✅ **业务类型隔离** - 基于 BizType 的任务类型隔离
- ✅ **失败通知** - 支持自定义通知器（日志、邮件、钉钉等）
- ✅ **幂等性保证** - 基于 bizType + bizId 的唯一约束
- ✅ **自动清理** - 支持任务删除和重置
- ✅ **易于扩展** - 插件化架构，易于扩展策略和处理器

## 架构设计

### 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                  Retry Task Framework                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐         ┌──────────────┐                 │
│  │   Facade     │◄────────│   Starter    │ (自动配置)       │
│  │  (对外接口)   │         └──────────────┘                 │
│  └──────┬───────┘                                           │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────────────────────────────────┐              │
│  │          Core (核心引擎)                   │              │
│  │  ┌────────────┐  ┌─────────────────┐   │              │
│  │  │ Executor   │  │ Processor       │   │              │
│  │  │(异步执行池) │  │ Registry        │   │              │
│  │  └────────────┘  └─────────────────┘   │              │
│  └────────┬─────────────────────────────────┘              │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────────────────────────────────────────┐  │
│  │      Strategy (重试策略)   Notify (通知)            │  │
│  │  ┌───────────┐  ┌───────────┐  ┌──────────┐       │  │
│  │  │  Fixed    │  │Exponential│  │  Notifier│       │  │
│  │  └───────────┘  └───────────┘  └──────────┘       │  │
│  └────────────────────┬────────────────────────────────┘  │
│                       │                                    │
│                       ▼                                    │
│  ┌─────────────────────────────────────────────────────┐  │
│  │            Infra (基础设施)                           │  │
│  │  ┌────────────┐  ┌──────────────┐                  │  │
│  │  │ JDBC       │  │  Optimistic  │                  │  │
│  │  │ Template   │  │  Locking     │                  │  │
│  │  └────────────┘  └──────────────┘                  │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 核心机制

#### 1. 分布式抢占 (Optimistic Locking)

为了解决多节点并发重复执行的问题，组件采用了基于 DB 状态的乐观锁机制：

- 任务执行前，通过 `UPDATE retry_task SET status='RUNNING' WHERE id=? AND status='PENDING'` 尝试抢锁。
- 只有抢锁成功的节点才会执行任务逻辑。
- 此机制不依赖 Redis 等外部组件，保持了架构的轻量级。

#### 2. 异步执行池

调度器只负责"拉取"和"分发"任务，实际执行提交给独立的 `Executor` 线程池。

- 这避免了单任务执行缓慢阻塞整个调度线程的问题，大幅提升了吞吐量。

#### 3. 防饥饿调度

调度器会遍历配置的所有 `biz-types`，针对每个业务类型独立拉取任务，防止某一类业务任务积压导致其他业务任务无法执行（饥饿问题）。

#### 4. 僵尸任务自愈

自动检测处于 `RUNNING` 状态超过阈值的任务（如节点宕机导致），并将其重置为 `PENDING`，确保任务不丢失。

### 核心流程

```
1. 任务注册
   ├─ 业务代码调用 RetryTaskFacade.register()
   ├─ 验证参数并生成唯一标识 (bizType + bizId)
   ├─ 检查是否需要立即执行 (executeImmediately)
   │   ├─ 是: 提交到线程池执行 (可选择同步等待结果)
   │   └─ 否: 仅持久化，等待调度
   └─ 持久化到数据库 (PENDING)

2. 任务调度 (DistributedScheduler)
   ├─ 遍历所有 BizType
   ├─ 拉取到期任务 (PENDING)
   ├─ 提交到 Executor 线程池
   └─ 异步执行

3. 任务执行 (Executor)
   ├─ 尝试抢占任务 (乐观锁: PENDING -> RUNNING)
   │   ├─ 失败: 跳过 (已被其他节点抢占)
   │   └─ 成功: 执行 Processor 逻辑
   └─ 处理结果
       ├─ 成功: 标记删除或归档
       └─ 失败:
           ├─ 重试次数 < max: 计算下次时间，状态重置为 PENDING
           └─ 重试次数 >= max: 标记 FAILED，发送通知
```

## 快速开始

### 1. 添加依赖

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-retrytask-starter</artifactId>
</dependency>
```

### 2. 配置数据库

执行 `schema.sql` 创建 `retry_task` 表。

### 3. 配置文件

```yaml
loadup:
  retrytask:
    biz-types:
      ORDER_PAYMENT:          # 业务类型
        strategy: exponential  # 重试策略：fixed/exponential
        max-retry-count: 5     # 最大重试次数
        notifier: log          # 通知器：log/email/dingtalk
        priority: H            # 优先级：H(高)/L(低)
      
      USER_REGISTER:
        strategy: fixed
        max-retry-count: 3
        notifier: log
        priority: L
```

### 4. 实现 Processor

```java
@Component
public class OrderPaymentProcessor implements RetryTaskProcessor {
    
    @Override
    public String getBizType() {
        return "ORDER_PAYMENT";
    }
    
    @Override
    public boolean process(RetryTask task) {
        // 处理订单支付重试逻辑
        String bizId = task.getBizId();
        
        try {
            // 调用支付接口
            paymentService.processPayment(bizId);
            return true; // 成功
        } catch (Exception e) {
            log.error("支付失败: {}", bizId, e);
            return false; // 失败，会自动重试
        }
    }
}
```

### 5. 注册任务

```java
@Autowired
private RetryTaskFacade retryTaskFacade;

// 注册重试任务
RetryTaskRegisterRequest request = new RetryTaskRegisterRequest();
request.setBizType("ORDER_PAYMENT");
request.setBizId("ORDER_123456");

Long taskId = retryTaskFacade.register(request);
```

## 配置说明

### 全局配置

```yaml
loadup:
  retrytask:
    # 数据库类型 (mysql/pgsql/oracle)，默认 mysql
    db-type: mysql
    
    # 表前缀，如 "loadup_"，默认空
    table-prefix: ""
    
    # 线程池配置 (Spring Boot 标准配置)
    executor:
      core-pool-size: 10
      max-pool-size: 50
      queue-capacity: 1000
```

### 业务类型配置 (BizType)

| 配置项                   | 类型      | 默认值   | 说明                                           |
|-----------------------|---------|-------|----------------------------------------------|
| `strategy`            | String  | fixed | 重试策略 (fixed, exponential, random, etc.)      |
| `max-retry-count`     | Integer | 10    | 最大重试次数                                       |
| `notifier`            | String  | log   | 通知器类型                                        |
| `priority`            | String  | L     | 优先级（H/L）                                     |
| `execute-immediately` | Boolean | false | 注册后是否立即提交执行 (跳过首次等待)                         |
| `wait-result`         | Boolean | false | 是否同步阻塞等待执行结果 (仅 execute-immediately=true 有效) |

### 调度器配置

```yaml
loadup:
  retrytask:
    scheduler:
      enabled: true            # 开启调度器
      scan-cron: "0 * * * * ?" # 任务扫描频率 (Cron)
      zombie-check-cron: "0 */5 * * * * ?" # 僵尸任务检查频率 (Cron)
```

## 使用场景

### 场景 1: 订单支付重试

```java
// 1. 实现 Processor
@Component
public class OrderPaymentProcessor implements RetryTaskProcessor {
    
    @Autowired
    private PaymentService paymentService;
    
    @Override
    public String getBizType() {
        return "ORDER_PAYMENT";
    }
    
    @Override
    public boolean process(RetryTask task) {
        return paymentService.retryPayment(task.getBizId());
    }
}

// 2. 注册任务
public void createOrder(Order order) {
    // 创建订单
    orderService.create(order);
    
    // 发起支付
    boolean success = paymentService.pay(order.getId());
    
    if (!success) {
        // 支付失败，注册重试任务
        RetryTaskRegisterRequest request = new RetryTaskRegisterRequest();
        request.setBizType("ORDER_PAYMENT");
        request.setBizId(order.getId());
        retryTaskFacade.register(request);
    }
}
```

### 场景 2: 消息推送重试

```java
@Component
public class MessagePushProcessor implements RetryTaskProcessor {
    
    @Override
    public String getBizType() {
        return "MESSAGE_PUSH";
    }
    
    @Override
    public boolean process(RetryTask task) {
        // 重试推送消息
        String userId = task.getBizId();
        return messagePushService.push(userId);
    }
}
```

### 场景 3: 手动管理任务

```java
// 删除任务
retryTaskFacade.delete("ORDER_PAYMENT", "ORDER_123456");

// 重置任务（重新开始重试）
retryTaskFacade.reset("ORDER_PAYMENT", "ORDER_123456");
```

## 重试策略

### 1. 固定间隔策略 (FixedRetryStrategy)

```yaml
strategy: fixed
```

- 每次重试间隔固定时长
- 适用于外部服务临时故障的场景

**计算公式**:

```
下次重试时间 = 当前时间 + 固定间隔（默认 60 秒）
```

### 2. 指数退避策略 (ExponentialBackoffRetryStrategy)

```yaml
strategy: exponential
```

- 重试间隔呈指数增长
- 适用于需要逐渐增加重试间隔的场景

**计算公式**:

```
下次重试时间 = 当前时间 + (2^重试次数) * 基础间隔
```

**示例**:

- 第 1 次重试: 2 秒后
- 第 2 次重试: 4 秒后
- 第 3 次重试: 8 秒后
- 第 4 次重试: 16 秒后

### 3. 自定义策略

实现 `RetryStrategy` 接口：

```java
@Component
public class CustomRetryStrategy implements RetryStrategy {
    
    @Override
    public String getType() {
        return "custom";
    }
    
    @Override
    public LocalDateTime nextRetryTime(RetryTask task) {
        // 自定义计算逻辑
        int retryCount = task.getRetryCount();
        int delaySeconds = calculateDelay(retryCount);
        return LocalDateTime.now().plusSeconds(delaySeconds);
    }
    
    private int calculateDelay(int retryCount) {
        // 自定义延迟计算
        return retryCount * 30;
    }
}
```

## 扩展开发

### 1. 自定义通知器

```java
@Component
public class DingtalkNotifier implements RetryTaskNotifier {
    
    @Override
    public String getType() {
        return "dingtalk";
    }
    
    @Override
    public void notifyFailure(RetryTask task) {
        // 发送钉钉通知
        String message = String.format(
            "重试任务失败: bizType=%s, bizId=%s, 重试次数=%d",
            task.getBizType(),
            task.getBizId(),
            task.getRetryCount()
        );
        dingtalkService.send(message);
    }
}
```

## 性能优化

### 1. 索引优化

```sql
-- 复合索引：状态 + 优先级 + 时间
KEY `idx_status_priority_time` (`status`, `priority`, `next_retry_time`)
```

### 2. 批量处理

```java
// 一次扫描多个任务
List<RetryTask> tasks = retryTaskManagement.scanPendingTasks(100);
```

### 3. 线程池配置

```yaml
loadup:
  retrytask:
    executor:
      core-pool-size: 10      # 核心线程数
      max-pool-size: 50       # 最大线程数
      queue-capacity: 1000    # 队列容量
      keep-alive-seconds: 60  # 空闲线程存活时间
```

## 数据模型

### retry_task 表结构

| 字段                    | 类型           | 说明     |
|-----------------------|--------------|--------|
| `id`                  | BIGINT       | 主键     |
| `biz_type`            | VARCHAR(255) | 业务类型   |
| `biz_id`              | VARCHAR(255) | 业务ID   |
| `retry_count`         | INT          | 已重试次数  |
| `max_retry_count`     | INT          | 最大重试次数 |
| `next_retry_time`     | DATETIME     | 下次重试时间 |
| `status`              | VARCHAR(255) | 状态     |
| `priority`            | CHAR(1)      | 优先级    |
| `last_failure_reason` | TEXT         | 最后失败原因 |
| `create_time`         | DATETIME     | 创建时间   |
| `update_time`         | DATETIME     | 更新时间   |

**约束**:

- 唯一键: `uk_biz_type_biz_id` (bizType + bizId)
- 复合索引: `idx_status_priority_time` (status + priority + nextRetryTime)

## 常见问题

### Q: 任务重复执行怎么办？

**A:**

1. **DB 乐观锁 (默认)**：组件内置了基于 `RUNNING` 状态的乐观锁机制。多节点同时拉取到同一任务时，只有一个节点能成功更新状态并获得锁，从而避免重复执行。
2. **唯一约束**：数据库层面的 `uk_biz_type_biz_id` 保证了同一业务 ID 不会注册多个活跃任务。

### Q: 如何监控任务执行情况？

**A:**

1. 当前：查看日志和数据库记录
2. 未来：集成 Micrometer 暴露指标

### Q: 支持哪些数据库？

**A:**

- MySQL
- PostgreSQL
- Oracle

## 许可证

本组件基于 Apache 2.0 许可证开源。

**© 2026 LoadUp Framework. All rights reserved.**

