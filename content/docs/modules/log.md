# Log 模块（loadup-modules-log）

操作日志、审计日志与错误日志管理模块，提供统一的用户行为记录、变更审计、异常捕获与统计分析能力。所有写入均通过异步线程池执行，不阻塞主业务链路。

---

## 模块结构

```
loadup-modules-log/
├── loadup-modules-log-client/          # 对外 DTO + Query
├── loadup-modules-log-domain/          # 领域模型（POJO）+ Gateway 接口 + 枚举
├── loadup-modules-log-infrastructure/  # DO + GatewayImpl + 异步写入器
├── loadup-modules-log-app/             # @Service 业务编排 + AutoConfiguration
└── loadup-modules-log-test/            # 集成测试（Testcontainers MySQL）
```

---

## 功能概览

| 功能   | 说明                                 |
|------|------------------------------------|
| 操作日志 | 记录用户的 CRUD、登录、导出等行为，含请求参数、耗时、成功/失败 |
| 审计日志 | 记录敏感数据变更的前后快照（JSON diff），满足合规审计要求  |
| 错误日志 | 捕获业务异常、系统异常、三方调用异常，含完整堆栈           |
| 统计分析 | 按模块/操作类型/日期分组统计，提供成功率、平均耗时等指标      |
| 异步写入 | 通过 `LogAsyncWriter` 异步落库，零侵入主业务事务  |
| 分页查询 | 所有日志类型均支持多维度条件分页查询，单页上限 200 条      |

---

## 领域模型

### OperationLog（操作日志）

| 字段               | 类型              | 说明                                                            |
|------------------|-----------------|---------------------------------------------------------------|
| `id`             | `String`        | 主键（UUID 无连字符）                                                 |
| `userId`         | `String`        | 操作用户 ID                                                       |
| `username`       | `String`        | 操作用户名                                                         |
| `module`         | `String`        | 所属模块（如 `CONFIG`、`UPMS`）                                       |
| `operationType`  | `String`        | 操作类型枚举：`CREATE/UPDATE/DELETE/QUERY/EXPORT/LOGIN/LOGOUT/OTHER` |
| `description`    | `String`        | 操作描述                                                          |
| `method`         | `String`        | 调用方法名                                                         |
| `requestParams`  | `String`        | 请求参数（脱敏后 JSON）                                                |
| `responseResult` | `String`        | 返回结果摘要                                                        |
| `duration`       | `Long`          | 执行耗时（ms）                                                      |
| `success`        | `Boolean`       | 是否成功                                                          |
| `errorMessage`   | `String`        | 失败原因                                                          |
| `ip`             | `String`        | 客户端 IP                                                        |
| `userAgent`      | `String`        | User-Agent                                                    |
| `operationTime`  | `LocalDateTime` | 操作时间                                                          |

### AuditLog（审计日志）

| 字段              | 类型              | 说明                                          |
|-----------------|-----------------|---------------------------------------------|
| `id`            | `String`        | 主键                                          |
| `userId`        | `String`        | 操作用户 ID                                     |
| `username`      | `String`        | 操作用户名                                       |
| `dataType`      | `String`        | 数据类型（如 `USER`、`ROLE`、`CONFIG`）              |
| `dataId`        | `String`        | 被操作数据的 ID                                   |
| `action`        | `String`        | 操作动作（如 `CREATE`、`UPDATE`、`DELETE`、`ASSIGN`） |
| `beforeData`    | `String`        | 变更前数据（JSON）                                 |
| `afterData`     | `String`        | 变更后数据（JSON）                                 |
| `diffData`      | `String`        | 差异数据（JSON）                                  |
| `reason`        | `String`        | 变更原因                                        |
| `ip`            | `String`        | 客户端 IP                                      |
| `operationTime` | `LocalDateTime` | 操作时间                                        |

### ErrorLog（错误日志）

| 字段              | 类型              | 说明                                     |
|-----------------|-----------------|----------------------------------------|
| `id`            | `String`        | 主键                                     |
| `userId`        | `String`        | 触发用户 ID（可为空）                           |
| `errorType`     | `String`        | 错误类型：`BUSINESS / SYSTEM / THIRD_PARTY` |
| `errorCode`     | `String`        | 应用错误码（可为空）                             |
| `errorMessage`  | `String`        | 简短错误描述                                 |
| `stackTrace`    | `String`        | 完整堆栈信息                                 |
| `requestUrl`    | `String`        | 触发请求 URL                               |
| `requestMethod` | `String`        | HTTP 方法                                |
| `requestParams` | `String`        | 请求参数                                   |
| `ip`            | `String`        | 客户端 IP                                 |
| `errorTime`     | `LocalDateTime` | 错误发生时间                                 |

---

## 枚举

### OperationType

```
CREATE, UPDATE, DELETE, QUERY, EXPORT, LOGIN, LOGOUT, OTHER
```

---

## Service API

### OperationLogService

| 方法                                                     | 说明                                  |
|--------------------------------------------------------|-------------------------------------|
| `listByCondition(query)`                               | 多条件分页查询操作日志（最大 200 条/页）             |
| `countByCondition(query)`                              | 统计符合条件的操作日志数量                       |
| `record(userId, username, module, operationType, ...)` | 手动异步记录一条操作日志                        |
| `statistics(query)`                                    | 统计分析：总量、成功/失败数、成功率、平均耗时、按模块/类型/日期分组 |

### AuditLogService

| 方法                                                                              | 说明                      |
|---------------------------------------------------------------------------------|-------------------------|
| `listByCondition(query)`                                                        | 多条件分页查询审计日志（最大 200 条/页） |
| `countByCondition(query)`                                                       | 统计符合条件的审计日志数量           |
| `record(userId, username, dataType, dataId, action, before, after, reason, ip)` | 手动异步记录一条审计日志            |

### ErrorLogService

| 方法                                                                                              | 说明                      |
|-------------------------------------------------------------------------------------------------|-------------------------|
| `listByCondition(query)`                                                                        | 多条件分页查询错误日志（最大 200 条/页） |
| `countByCondition(query)`                                                                       | 统计符合条件的错误日志数量           |
| `record(userId, errorType, errorCode, errorMessage, stackTrace, requestUrl, requestMethod, ip)` | 手动异步记录一条错误日志            |

### 统计返回结构（LogStatisticsDTO）

| 字段             | 说明        |
|----------------|-----------|
| `total`        | 总操作数      |
| `successCount` | 成功数       |
| `failureCount` | 失败数       |
| `successRate`  | 成功率（%）    |
| `avgDuration`  | 平均耗时（ms）  |
| `maxDuration`  | 最大耗时（ms）  |
| `byModule`     | 按模块分组统计   |
| `byType`       | 按操作类型分组统计 |
| `byDate`       | 按日期分组统计   |

---

## 异步写入

所有日志写入通过 `LogAsyncWriter` 执行，使用 Spring `@Async` 异步线程池，不占用主业务线程，也不参与主业务事务。

```yaml
# application.yml 异步线程池配置
spring:
  task:
    execution:
      pool:
        core-size: 4
        max-size: 16
        queue-capacity: 1000
        keep-alive: 60s
      thread-name-prefix: log-async-
```

> ⚠️ 异步写入失败不会影响主业务返回，但错误会记录在 Spring 日志中。生产环境建议监控线程池队列积压情况。

---

## Gateway 路由配置

在 `loadup-application/src/main/resources/application.yml` 中添加以下路由：

```yaml
loadup:
  gateway:
    routes:
      # ── 操作日志 ──
      - path: /api/v1/log/operation/list
        method: POST
        target: "bean://operationLogService:listByCondition"
        securityCode: "default"
      - path: /api/v1/log/operation/count
        method: POST
        target: "bean://operationLogService:countByCondition"
        securityCode: "default"
      - path: /api/v1/log/operation/record
        method: POST
        target: "bean://operationLogService:record"
        securityCode: "default"
      - path: /api/v1/log/operation/statistics
        method: POST
        target: "bean://operationLogService:statistics"
        securityCode: "default"
      # ── 审计日志 ──
      - path: /api/v1/log/audit/list
        method: POST
        target: "bean://auditLogService:listByCondition"
        securityCode: "default"
      - path: /api/v1/log/audit/count
        method: POST
        target: "bean://auditLogService:countByCondition"
        securityCode: "default"
      - path: /api/v1/log/audit/record
        method: POST
        target: "bean://auditLogService:record"
        securityCode: "default"
      # ── 错误日志 ──
      - path: /api/v1/log/error/list
        method: POST
        target: "bean://errorLogService:listByCondition"
        securityCode: "default"
      - path: /api/v1/log/error/count
        method: POST
        target: "bean://errorLogService:countByCondition"
        securityCode: "default"
      - path: /api/v1/log/error/record
        method: POST
        target: "bean://errorLogService:record"
        securityCode: "default"
```

---

## 数据库表结构

### operation_log

```sql
CREATE TABLE IF NOT EXISTS operation_log (
    id             VARCHAR(64)  NOT NULL                                          COMMENT 'ID',
    tenant_id      VARCHAR(64)                                                    COMMENT '租户ID',
    user_id        VARCHAR(64)                                                    COMMENT '操作用户ID',
    username       VARCHAR(100)                                                   COMMENT '操作用户名',
    module         VARCHAR(50)  NOT NULL DEFAULT ''                               COMMENT '模块',
    operation_type VARCHAR(20)  NOT NULL                                          COMMENT '操作类型: CREATE/UPDATE/DELETE/QUERY/EXPORT/LOGIN/LOGOUT',
    description    VARCHAR(500)                                                   COMMENT '操作描述',
    method         VARCHAR(500)                                                   COMMENT '方法名',
    request_params TEXT                                                           COMMENT '请求参数',
    response_result TEXT                                                          COMMENT '返回结果',
    duration       BIGINT                                                         COMMENT '执行时长(ms)',
    success        TINYINT      NOT NULL DEFAULT 1                                COMMENT '是否成功',
    error_message  TEXT                                                           COMMENT '错误信息',
    ip             VARCHAR(128)                                                   COMMENT 'IP地址',
    user_agent     VARCHAR(500)                                                   COMMENT 'User-Agent',
    operation_time DATETIME     NOT NULL                 COMMENT '操作时间',
    created_at     DATETIME     NOT NULL                 COMMENT '创建时间',
    updated_at     DATETIME              NOT NULL         COMMENT '更新时间',
    deleted        TINYINT      NOT NULL DEFAULT 0                                COMMENT '删除标记',
    PRIMARY KEY (id),
    KEY idx_tenant_id      (tenant_id),
    KEY idx_user_id        (user_id),
    KEY idx_module         (module),
    KEY idx_operation_type (operation_type),
    KEY idx_operation_time (operation_time),
    KEY idx_success        (success)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='操作日志表';
```

### audit_log

```sql
CREATE TABLE IF NOT EXISTS audit_log (
    id             VARCHAR(64)  NOT NULL                                          COMMENT 'ID',
    tenant_id      VARCHAR(64)                                                    COMMENT '租户ID',
    user_id        VARCHAR(64)  NOT NULL DEFAULT ''                               COMMENT '操作用户ID',
    username       VARCHAR(100) NOT NULL DEFAULT ''                               COMMENT '操作用户名',
    data_type      VARCHAR(50)  NOT NULL                                          COMMENT '数据类型: USER/ROLE/PERMISSION/CONFIG..',
    data_id        VARCHAR(64)                                                    COMMENT '数据ID',
    action         VARCHAR(20)  NOT NULL                                          COMMENT '操作: CREATE/UPDATE/DELETE/ASSIGN..',
    before_data    TEXT                                                           COMMENT '变更前数据(JSON)',
    after_data     TEXT                                                           COMMENT '变更后数据(JSON)',
    diff_data      TEXT                                                           COMMENT '差异数据(JSON)',
    reason         VARCHAR(500)                                                   COMMENT '变更原因',
    ip             VARCHAR(128)                                                   COMMENT 'IP地址',
    operation_time DATETIME     NOT NULL                 COMMENT '操作时间',
    created_at     DATETIME     NOT NULL                 COMMENT '创建时间',
    updated_at     DATETIME              NOT NULL         COMMENT '更新时间',
    deleted        TINYINT      NOT NULL DEFAULT 0                                COMMENT '删除标记',
    PRIMARY KEY (id),
    KEY idx_tenant_id      (tenant_id),
    KEY idx_user_id        (user_id),
    KEY idx_data_type      (data_type),
    KEY idx_data_id        (data_id),
    KEY idx_action         (action),
    KEY idx_operation_time (operation_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='审计日志表';
```

### error_log

```sql
CREATE TABLE IF NOT EXISTS error_log (
    id             VARCHAR(64)   NOT NULL                                         COMMENT 'ID',
    tenant_id      VARCHAR(64)                                                    COMMENT '租户ID',
    user_id        VARCHAR(64)                                                    COMMENT '用户ID',
    error_type     VARCHAR(20)   NOT NULL                                         COMMENT '错误类型: BUSINESS/SYSTEM/THIRD_PARTY',
    error_code     VARCHAR(64)                                                    COMMENT '错误码',
    error_message  VARCHAR(1000) NOT NULL                                         COMMENT '错误信息',
    stack_trace    TEXT                                                           COMMENT '堆栈信息',
    request_url    VARCHAR(500)                                                   COMMENT '请求URL',
    request_method VARCHAR(10)                                                    COMMENT '请求方法',
    request_params TEXT                                                           COMMENT '请求参数',
    ip             VARCHAR(128)                                                   COMMENT 'IP地址',
    error_time     DATETIME      NOT NULL                COMMENT '错误时间',
    created_at     DATETIME      NOT NULL                COMMENT '创建时间',
    updated_at     DATETIME               NOT NULL        COMMENT '更新时间',
    deleted        TINYINT       NOT NULL DEFAULT 0                               COMMENT '删除标记',
    PRIMARY KEY (id),
    KEY idx_tenant_id   (tenant_id),
    KEY idx_user_id     (user_id),
    KEY idx_error_type  (error_type),
    KEY idx_error_code  (error_code),
    KEY idx_error_time  (error_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='错误日志表';
```

---

## 安全规范

- `requestParams` / `responseResult` 字段写入前必须经过脱敏处理（手机号、身份证、密码等）
- 禁止将 Token、密码字段写入任何日志字段
- `errorMessage` 不得包含敏感业务数据，`stackTrace` 仅供内部查询，不对外暴露

---

## 测试

测试模块：`loadup-modules-log-test`，使用 Testcontainers 启动真实 MySQL，禁止 `@MockBean` 替代数据库。

```bash
# 运行所有集成测试（需要 Docker）
mvn test -pl modules/loadup-modules-log/loadup-modules-log-test
```

### 测试类说明

| 测试类                        | 场景                            |
|----------------------------|-------------------------------|
| `OperationLogServiceIT`    | 操作日志记录、分页查询、多条件过滤             |
| `OperationLogStatisticsIT` | 统计分析：成功率、耗时分布、按模块/类型/日期分组     |
| `AuditLogServiceIT`        | 审计日志记录、分页查询、before/after 数据验证 |
| `ErrorLogServiceIT`        | 错误日志记录、按类型/错误码查询              |

### 测试资源文件

`src/test/resources/` 目录下包含以下文件：

| 文件                     | 说明                                                     |
|------------------------|--------------------------------------------------------|
| `application.yml`      | 激活 `test` profile（固定内容：`spring.profiles.active: test`） |
| `application-test.yml` | 本地开发配置（Testcontainers reuse=true，print-sql=true）       |
| `application-ci.yml`   | CI 流水线配置（Testcontainers reuse=false，print-sql=false）   |
| `schema.sql`           | 测试数据库 Schema（与生产 schema 完全一致）                          |

