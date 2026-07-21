# LoadUp Components :: Global Unique

## 概述

`loadup-components-globalunique` 是一个基于数据库唯一键约束的全局幂等性控制组件。通过数据库的唯一键机制，实现分布式环境下的幂等性保证，避免重复操作。

## 核心特性

- ✅ **极简设计**: 基于数据库唯一键约束，零状态管理
- ✅ **事务一致性**: 与业务逻辑在同一事务中，保证数据一致性
- ✅ **多数据库支持**: 支持 MySQL、PostgreSQL、Oracle
- ✅ **灵活配置**: 支持表名前缀配置
- ✅ **零运维成本**: 无需定时任务、无需缓存、无需状态维护
- ✅ **高性能**: 仅依赖数据库索引，无额外查询

## 快速开始

### 1. 引入依赖

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-globalunique</artifactId>
</dependency>
```

### 2. 配置

```yaml
loadup:
  components:
    globalunique:
      enabled: true              # 是否启用，默认 true
      db-type: MYSQL             # 数据库类型: MYSQL, POSTGRESQL, ORACLE
      table-prefix: ""           # 表名前缀，默认为空
      table-name: global_unique  # 表名，默认 global_unique
```

### 3. 执行数据库迁移

根据数据库类型，执行对应的 SQL 脚本：

- MySQL: `V20260225000001__create_global_unique_mysql.sql`
- PostgreSQL: `V20260225000001__create_global_unique_postgresql.sql`
- Oracle: `V20260225000001__create_global_unique_oracle.sql`

或者使用 Flyway 自动迁移（推荐）：

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
```

### 4. 使用示例

#### 基本用法

```java
@Service
@RequiredArgsConstructor
public class OrderServiceImpl implements OrderService {
    
    private final GlobalUniqueService globalUniqueService;
    private final OrderMapper orderMapper;
    
    @Override
    @Transactional(rollbackFor = Exception.class)
    public OrderResponse createOrder(OrderRequest request) {
        
        // 1. 生成唯一键（业务方自定义规则）
        String uniqueKey = String.format("ORDER_CREATE:%s:%s", 
            request.getUserId(), request.getOrderNo());
        
        // 2. 幂等检查（在业务事务内）
        if (!globalUniqueService.insertAndCheck(uniqueKey, "ORDER")) {
            // 幂等拦截：查询并返回已存在的订单
            log.info("订单创建幂等拦截: orderNo={}", request.getOrderNo());
            return getExistingOrder(request.getOrderNo());
        }
        
        // 3. 执行业务逻辑
        OrderEntity order = buildOrder(request);
        orderMapper.insert(order);
        
        return convertToResponse(order);
    }
}
```

#### 带业务ID和请求数据

```java
// 带业务ID
boolean result = globalUniqueService.insertAndCheck(uniqueKey, bizType, bizId);

// 带请求数据快照（用于问题排查）
String requestDataJson = "{\"userId\":\"123\",\"amount\":100.00}";
boolean result = globalUniqueService.insertAndCheck(uniqueKey, bizType, bizId, requestDataJson);
```

## 工作原理

### 1. 核心机制

```
客户端请求
    ↓
业务逻辑开始事务
    ↓
GlobalUniqueService.insertAndCheck(uniqueKey, bizType)
    ↓
尝试插入记录到 global_unique 表
    ↓
    ├─→ 成功: 返回 true，继续执行业务逻辑
    └─→ 唯一键冲突: 捕获 DuplicateKeyException，返回 false（幂等拦截）
```

### 2. 事务行为

- **业务提交**: `global_unique` 记录持久化 → 永久幂等
- **业务回滚**: `global_unique` 记录也回滚 → 下次请求可重试

### 3. 唯一键生成规则

业务方需要自定义唯一键规则，建议格式：

```
业务类型:业务维度1:业务维度2:...
```

示例：

```java
// 订单创建
String uniqueKey = "ORDER_CREATE:" + userId + ":" + orderNo;

// 支付
String uniqueKey = "PAYMENT:" + userId + ":" + paymentNo + ":" + timestamp;

// 退款
String uniqueKey = "REFUND:" + orderId + ":" + refundNo;
```

## 数据库表结构

### MySQL

```sql
CREATE TABLE global_unique (
    id VARCHAR(64) NOT NULL COMMENT '主键',
    unique_key VARCHAR(255) NOT NULL COMMENT '唯一键(业务方自定义)',
    biz_type VARCHAR(50) NOT NULL COMMENT '业务类型',
    biz_id VARCHAR(100) DEFAULT NULL COMMENT '业务ID(可选)',
    request_data TEXT DEFAULT NULL COMMENT '请求数据快照(可选,用于问题排查)',
    created_at DATETIME NOT NULL  COMMENT '创建时间',
    updated_at DATETIME NOT NULL  ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (id),
    UNIQUE KEY uk_unique_key (unique_key),
    KEY idx_biz_type (biz_type),
    KEY idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## API 文档

### GlobalUniqueService

```java
public interface GlobalUniqueService {
    
    /**
     * 插入并检查唯一性（核心方法）
     * @param uniqueKey 唯一键（业务方自行拼接）
     * @param bizType 业务类型
     * @return true=首次插入成功, false=已存在(幂等拦截)
     */
    boolean insertAndCheck(String uniqueKey, String bizType);
    
    /**
     * 插入并检查（带业务ID）
     */
    boolean insertAndCheck(String uniqueKey, String bizType, String bizId);
    
    /**
     * 插入并检查（带请求数据快照）
     */
    boolean insertAndCheck(String uniqueKey, String bizType, String bizId, String requestData);
}
```

## 最佳实践

### 1. 唯一键设计

- ✅ **包含业务类型**: 便于区分不同业务
- ✅ **包含业务维度**: 如 userId、orderNo 等
- ✅ **可读性**: 便于问题排查
- ❌ **避免过长**: 控制在 255 字符以内

### 2. 事务边界

```java
// ✅ 正确：幂等检查在业务事务内
@Transactional
public void bizMethod() {
    if (globalUniqueService.insertAndCheck(...)) {
        // 业务逻辑
    }
}

// ❌ 错误：幂等检查在事务外
public void bizMethod() {
    if (globalUniqueService.insertAndCheck(...)) {
        // 业务逻辑在另一个事务中
        doBusinessInNewTransaction();
    }
}
```

### 3. 失败重试

- **业务失败回滚**: 幂等记录也回滚，可以重试
- **业务成功提交**: 幂等记录持久化，不可重试

### 4. 数据维护

- **数据持续增长**: 可按需手动归档（如按月分表）
- **历史数据查询**: 通过 `created_at` 索引高效查询

## 性能考虑

### 优势

- ✅ **单次数据库操作**: 只需一次 INSERT
- ✅ **索引命中**: 利用唯一键索引
- ✅ **无额外查询**: 不需要 SELECT 后 INSERT

### 潜在瓶颈

- ⚠️ **表数据增长**: 长期运行会积累大量数据
    - 解决方案: 定期归档或分表
- ⚠️ **数据库负载**: 高并发场景下数据库压力
    - 解决方案: 数据库优化、读写分离

## 常见问题

### Q1: 为什么不使用 Redis？

**A**:

- 数据库唯一键保证强一致性
- 与业务数据在同一事务中，避免数据不一致
- 无需额外运维 Redis

### Q2: 表数据会一直增长吗？

**A**:

- 是的，记录永久保留（无过期机制）
- 可根据业务需求手动归档历史数据
- 建议监控数据增长速度，按需优化

### Q3: 如何处理并发插入？

**A**:

- 数据库唯一键自动处理并发冲突
- 其中一个成功，其他抛出 `DuplicateKeyException`
- 组件内部捕获异常并返回 `false`

### Q4: 支持分布式部署吗？

**A**:

- 完全支持分布式部署
- 依赖数据库唯一键保证全局唯一性

## 架构决策

### 为什么不使用状态字段？

- **简化设计**: 无需维护 `PROCESSING/SUCCESS/FAILED` 状态
- **避免状态机**: 减少复杂度
- **依赖唯一键**: 插入成功即表示幂等检查通过

### 为什么不使用过期时间？

- **永久保留**: 避免误删导致重复操作
- **审计需求**: 保留完整操作记录
- **按需归档**: 业务自主决定归档策略

### 为什么不使用缓存？

- **强一致性**: 数据库保证数据一致性
- **事务绑定**: 与业务数据在同一事务
- **零运维**: 无需额外缓存组件

## 后续优化方向

1. **数据归档**: 提供自动归档功能（可选）
2. **监控指标**: 提供 Micrometer 指标暴露
3. **性能优化**: 提供批量检查接口
4. **缓存层**: 可选的 Redis 缓存支持（高性能场景）

## 许可证

本组件基于 Apache 2.0 许可证开源。

