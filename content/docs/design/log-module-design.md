# LoadUp Log 日志中心模块 - 设计方案

> **版本**: v1.0  
> **创建日期**: 2026-02-28  
> **模块代号**: loadup-modules-log  
> **优先级**: 🔴 P0

## 📋 目录

- [1. 模块概述](#1-模块概述)
- [2. 功能设计](#2-功能设计)
- [3. 架构设计](#3-架构设计)
- [4. 数据模型设计](#4-数据模型设计)
- [5. API 设计](#5-api-设计)
- [6. 技术实现](#6-技术实现)
- [7. 性能优化](#7-性能优化)
- [8. 测试方案](#8-测试方案)
- [9. 实施计划](#9-实施计划)

---

## 1. 模块概述

### 1.1 业务价值

Log 日志中心模块是系统审计和问题排查的核心模块，提供：

- **操作日志**: 自动记录用户的关键操作行为
- **访问日志**: HTTP 请求的完整记录
- **审计日志**: 敏感操作和数据变更的合规记录
- **错误日志**: 系统异常的集中收集
- **日志查询**: 多维度的日志检索和分析
- **日志导出**: 日志数据的导出和归档

### 1.2 核心特性

| 特性       | 说明              | 优先级 |
|----------|-----------------|-----|
| 操作日志自动记录 | AOP拦截，无侵入式记录    | P0  |
| 访问日志     | HTTP请求响应完整记录    | P0  |
| 审计日志     | 数据变更前后对比        | P0  |
| 异步写入     | 不影响业务性能         | P0  |
| 多维度查询    | 按时间/用户/模块/IP等查询 | P0  |
| 日志导出     | Excel/CSV 格式导出  | P1  |
| 日志统计     | 操作趋势分析          | P1  |
| 日志归档     | 历史数据归档          | P2  |

### 1.3 非功能需求

- **性能**: 日志记录不影响业务 (异步 < 10ms)
- **吞吐量**: 支持 10000+ TPS 日志写入
- **存储**: 支持海量日志存储 (分表/归档)
- **查询性能**: 普通查询 < 500ms
- **数据保留**: 默认保留6个月

---

## 2. 功能设计

### 2.1 操作日志

#### 功能列表

```
操作日志
├─ 自动记录
│  ├─ AOP 拦截 @OperationLog 注解
│  ├─ 记录请求参数
│  ├─ 记录返回结果（可选）
│  └─ 记录执行时长
├─ 手动记录
│  ├─ 通过 API 调用
│  └─ 自定义日志内容
├─ 日志分类
│  ├─ 按模块分类 (用户/角色/权限...)
│  ├─ 按操作类型 (CREATE/UPDATE/DELETE/QUERY/EXPORT...)
│  └─ 按业务类型
└─ 异步写入
   ├─ 线程池异步处理
   └─ 批量写入数据库
```

#### 核心场景

**场景1: 用户登录**

```java
@OperationLog(
    type = "LOGIN",
    module = "认证",
    description = "用户登录"
)
public LoginResponse login(LoginRequest request) {
    // 业务逻辑
}

// 记录内容:
// - 用户ID
// - 用户名
// - 登录时间
// - IP地址
// - User-Agent
// - 登录结果（成功/失败）
```

**场景2: 删除用户**

```java
@OperationLog(
    type = "DELETE",
    module = "用户管理",
    description = "删除用户",
    recordParams = true
)
public void deleteUser(String userId) {
    // 业务逻辑
}

// 记录内容:
// - 操作人
// - 被删除的用户ID
// - 操作时间
// - 操作结果
```

### 2.2 访问日志

#### 功能列表

```
访问日志
├─ HTTP 请求记录
│  ├─ 请求URL
│  ├─ 请求方法 (GET/POST/...)
│  ├─ 请求头
│  ├─ 请求参数
│  └─ 请求体
├─ HTTP 响应记录
│  ├─ 响应状态码
│  ├─ 响应头
│  ├─ 响应体（可选）
│  └─ 响应时长
├─ 请求来源
│  ├─ IP地址
│  ├─ User-Agent
│  ├─ Referer
│  └─ 地理位置（可选）
└─ 性能统计
   ├─ 接口调用频率
   ├─ 平均响应时间
   └─ 慢接口识别
```

#### 实现方式

```java
/**
 * 访问日志拦截器
 */
@Component
public class AccessLogInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) {
        // 记录请求开始时间
        request.setAttribute("startTime", System.currentTimeMillis());
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request,
                               HttpServletResponse response,
                               Object handler,
                               Exception ex) {
        // 记录访问日志
        long startTime = (Long) request.getAttribute("startTime");
        long duration = System.currentTimeMillis() - startTime;
        
        AccessLog log = AccessLog.builder()
            .url(request.getRequestURI())
            .method(request.getMethod())
            .ip(getClientIp(request))
            .userAgent(request.getHeader("User-Agent"))
            .statusCode(response.getStatus())
            .duration(duration)
            .build();
        
        // 异步写入
        accessLogService.saveAsync(log);
    }
}
```

### 2.3 审计日志

审计日志用于记录敏感操作和数据变更，通过 `AuditLogService.record()` 手动调用写入，不提供 AOP
自动拦截（变更前后数据对比意义有限，由业务方按需传入关键信息）。

#### 核心场景

**场景: 角色权限变更**

```java
// 手动记录，业务方传入关键操作信息
auditLogService.record(userId, username, "ROLE_PERMISSION", roleId,
    "ASSIGN", null, null, reason, ip);
```

#### 功能列表

```
审计日志
├─ 手动记录
│  ├─ dataType  数据类型 (USER/ROLE/CONFIG...)
│  ├─ dataId    数据ID
│  ├─ action    操作 (CREATE/UPDATE/DELETE/ASSIGN...)
│  ├─ reason    变更原因
│  └─ ip        操作来源IP
├─ 多维度查询
│  ├─ 按用户
│  ├─ 按数据类型/ID
│  ├─ 按操作类型
│  └─ 按时间范围
└─ 合规审计
   ├─ 操作人追溯
   └─ 操作时间追溯
```

### 2.4 错误日志

#### 功能列表

```
错误日志
├─ 异常捕获
│  ├─ 全局异常处理器
│  ├─ 异常堆栈记录
│  └─ 异常上下文信息
├─ 错误分类
│  ├─ 业务异常
│  ├─ 系统异常
│  └─ 第三方异常
├─ 错误统计
│  ├─ 错误率趋势
│  ├─ 高频错误TOP10
│  └─ 错误分布（按模块）
└─ 错误告警
   ├─ 错误阈值告警
   └─ 集成告警组件
```

---

## 3. 架构设计

### 3.1 分层架构 (COLA 4.0)

```
loadup-modules-log/
├─ loadup-modules-log-client/              # 客户端API
│  └─ src/main/java/
│     └─ io/github/loadup/log/client/
│        ├─ api/
│        │  ├─ OperationLogService.java
│        │  ├─ AccessLogService.java
│        │  └─ AuditLogService.java
│        ├─ annotation/
│        │  ├─ OperationLog.java         # 操作日志注解
│        │  └─ AuditLog.java             # 审计日志注解
│        └─ dto/
│           ├─ OperationLogDTO.java
│           └─ AuditLogDTO.java
│
├─ loadup-modules-log-adapter/             # 适配层
│  └─ src/main/java/
│     └─ io/github/loadup/log/adapter/
│        ├─ web/                          # REST API
│        │  ├─ OperationLogController.java
│        │  ├─ AuditLogController.java
│        │  └─ AccessLogController.java
│        ├─ interceptor/                  # 拦截器
│        │  ├─ AccessLogInterceptor.java
│        │  └─ ErrorLogInterceptor.java
│        └─ aspect/                       # AOP切面
│           ├─ OperationLogAspect.java
│           └─ AuditLogAspect.java
│
├─ loadup-modules-log-app/                 # 应用层
│  └─ src/main/java/
│     └─ io/github/loadup/log/app/
│        ├─ command/                      # 命令处理
│        │  └─ SaveLogCmd.java
│        ├─ query/                        # 查询处理
│        │  ├─ QueryOperationLogQry.java
│        │  └─ ExportLogQry.java
│        └─ executor/                     # 执行器
│           ├─ LogCommandExecutor.java
│           └─ LogQueryExecutor.java
│
├─ loadup-modules-log-domain/              # 领域层
│  └─ src/main/java/
│     └─ io/github/loadup/log/domain/
│        ├─ operation/                    # 操作日志聚合
│        │  ├─ OperationLog.java
│        │  └─ OperationLogRepository.java
│        ├─ audit/                        # 审计日志聚合
│        │  ├─ AuditLog.java
│        │  ├─ DataDiff.java             # 数据差异值对象
│        │  └─ AuditLogRepository.java
│        ├─ access/                       # 访问日志聚合
│        │  ├─ AccessLog.java
│        │  └─ AccessLogRepository.java
│        └─ error/                        # 错误日志聚合
│           ├─ ErrorLog.java
│           └─ ErrorLogRepository.java
│
├─ loadup-modules-log-infrastructure/      # 基础设施层
│  └─ src/main/java/
│     └─ io/github/loadup/log/infra/
│        ├─ repository/                   # 仓储实现
│        │  ├─ OperationLogRepositoryImpl.java
│        │  ├─ AuditLogRepositoryImpl.java
│        │  └─ AccessLogRepositoryImpl.java
│        ├─ mapper/                       # MyBatis Mapper
│        │  ├─ OperationLogMapper.java
│        │  ├─ AuditLogMapper.java
│        │  └─ AccessLogMapper.java
│        ├─ async/                        # 异步处理
│        │  ├─ LogAsyncService.java
│        │  └─ LogThreadPoolConfig.java
│        ├─ diff/                         # 数据对比（暂不实现，意义有限）
│        └─ archive/                      # 归档
│           └─ LogArchiveService.java
│
└─ loadup-modules-log-starter/             # 自动配置
   └─ src/main/java/
      └─ io/github/loadup/log/starter/
         ├─ LogAutoConfiguration.java
         ├─ LogProperties.java
         └─ LogAspectConfiguration.java
```

### 3.2 核心组件

#### 3.2.1 操作日志切面

```java
/**
 * 操作日志AOP切面
 */
@Aspect
@Component
@Slf4j
public class OperationLogAspect {
    
    private final LogAsyncService logAsyncService;
    private final HttpServletRequest request;
    
    @Around("@annotation(operationLog)")
    public Object around(ProceedingJoinPoint joinPoint, OperationLog operationLog) throws Throwable {
        
        long startTime = System.currentTimeMillis();
        Object result = null;
        Exception exception = null;
        
        try {
            // 执行方法
            result = joinPoint.proceed();
            return result;
        } catch (Exception e) {
            exception = e;
            throw e;
        } finally {
            // 异步记录日志
            logAsyncService.saveOperationLog(
                buildOperationLog(joinPoint, operationLog, startTime, result, exception)
            );
        }
    }
    
    private OperationLogDO buildOperationLog(ProceedingJoinPoint joinPoint,
                                              OperationLog annotation,
                                              long startTime,
                                              Object result,
                                              Exception exception) {
        long duration = System.currentTimeMillis() - startTime;
        
        return OperationLogDO.builder()
            .userId(UserContext.getUserId())
            .username(UserContext.getUsername())
            .module(annotation.module())
            .operationType(annotation.type())
            .description(annotation.description())
            .method(joinPoint.getSignature().toShortString())
            .params(annotation.recordParams() ? toJson(joinPoint.getArgs()) : null)
            .result(annotation.recordResponse() ? toJson(result) : null)
            .duration(duration)
            .success(exception == null)
            .errorMessage(exception != null ? exception.getMessage() : null)
            .ip(getClientIp(request))
            .userAgent(request.getHeader("User-Agent"))
            .operationTime(LocalDateTime.now())
            .build();
    }
}
```

#### 3.2.2 异步日志服务

```java
/**
 * 异步日志服务
 * 
 * 使用独立线程池处理日志写入，不阻塞业务线程
 */
@Service
@Slf4j
public class LogAsyncService {
    
    private final OperationLogRepository operationLogRepository;
    private final AuditLogRepository auditLogRepository;
    private final Executor logExecutor;
    
    /**
     * 异步保存操作日志
     */
    @Async("logExecutor")
    public CompletableFuture<Void> saveOperationLog(OperationLogDO log) {
        return CompletableFuture.runAsync(() -> {
            try {
                operationLogRepository.save(log);
            } catch (Exception e) {
                log.error("保存操作日志失败", e);
                // 失败时记录到文件或MQ
            }
        }, logExecutor);
    }
    
    /**
     * 批量保存操作日志
     */
    @Async("logExecutor")
    public CompletableFuture<Void> batchSaveOperationLog(List<OperationLogDO> logs) {
        return CompletableFuture.runAsync(() -> {
            try {
                // 分批插入，每批1000条
                Lists.partition(logs, 1000).forEach(batch -> {
                    operationLogRepository.batchSave(batch);
                });
            } catch (Exception e) {
                log.error("批量保存操作日志失败", e);
            }
        }, logExecutor);
    }
}
```

---

## 4. 数据模型设计

### 4.1 数据库表设计

#### 4.1.1 操作日志表 (operation_log)

```sql
CREATE TABLE operation_log (
    id VARCHAR(64) NOT NULL COMMENT '主键',
    user_id VARCHAR(64) COMMENT '操作用户ID',
    username VARCHAR(100) COMMENT '操作用户名',
    module VARCHAR(50) NOT NULL COMMENT '模块',
    operation_type VARCHAR(20) NOT NULL COMMENT '操作类型: CREATE/UPDATE/DELETE/QUERY/EXPORT/LOGIN/LOGOUT',
    description VARCHAR(500) COMMENT '操作描述',
    method VARCHAR(500) COMMENT '方法名',
    request_params TEXT COMMENT '请求参数',
    response_result TEXT COMMENT '返回结果',
    duration BIGINT COMMENT '执行时长(ms)',
    success BOOLEAN NOT NULL DEFAULT TRUE COMMENT '是否成功',
    error_message TEXT COMMENT '错误信息',
    ip VARCHAR(128) COMMENT 'IP地址',
    location VARCHAR(200) COMMENT '地理位置',
    user_agent VARCHAR(500) COMMENT 'User-Agent',
    operation_time DATETIME NOT NULL COMMENT '操作时间',
    created_at DATETIME NOT NULL  COMMENT '创建时间',
    PRIMARY KEY (id),
    KEY idx_user_id (user_id),
    KEY idx_module (module),
    KEY idx_operation_type (operation_type),
    KEY idx_operation_time (operation_time),
    KEY idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='操作日志表'
PARTITION BY RANGE (YEAR(operation_time) * 100 + MONTH(operation_time)) (
    PARTITION p202601 VALUES LESS THAN (202602),
    PARTITION p202602 VALUES LESS THAN (202603),
    PARTITION p202603 VALUES LESS THAN (202604),
    PARTITION p202604 VALUES LESS THAN (202605),
    PARTITION p202605 VALUES LESS THAN (202606),
    PARTITION p202606 VALUES LESS THAN (202607),
    PARTITION p202607 VALUES LESS THAN (202608),
    PARTITION p202608 VALUES LESS THAN (202609),
    PARTITION p202609 VALUES LESS THAN (202610),
    PARTITION p202610 VALUES LESS THAN (202611),
    PARTITION p202611 VALUES LESS THAN (202612),
    PARTITION p202612 VALUES LESS THAN (202701),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);
```

#### 4.1.2 审计日志表 (audit_log)

```sql
CREATE TABLE audit_log (
    id VARCHAR(64) NOT NULL COMMENT '主键',
    user_id VARCHAR(64) NOT NULL COMMENT '操作用户ID',
    username VARCHAR(100) NOT NULL COMMENT '操作用户名',
    data_type VARCHAR(50) NOT NULL COMMENT '数据类型: USER/ROLE/PERMISSION/CONFIG...',
    data_id VARCHAR(64) COMMENT '数据ID',
    action VARCHAR(20) NOT NULL COMMENT '操作: CREATE/UPDATE/DELETE/ASSIGN...',
    before_data JSON COMMENT '变更前数据',
    after_data JSON COMMENT '变更后数据',
    diff_data JSON COMMENT '差异数据',
    reason VARCHAR(500) COMMENT '变更原因',
    ip VARCHAR(128) COMMENT 'IP地址',
    operation_time DATETIME NOT NULL COMMENT '操作时间',
    created_at DATETIME NOT NULL  COMMENT '创建时间',
    PRIMARY KEY (id),
    KEY idx_user_id (user_id),
    KEY idx_data_type (data_type),
    KEY idx_data_id (data_id),
    KEY idx_action (action),
    KEY idx_operation_time (operation_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='审计日志表'
PARTITION BY RANGE (YEAR(operation_time) * 100 + MONTH(operation_time)) (
    PARTITION p202601 VALUES LESS THAN (202602),
    PARTITION p202602 VALUES LESS THAN (202603),
    -- ... 其他分区
    PARTITION pmax VALUES LESS THAN MAXVALUE
);
```

#### 4.1.3 访问日志表 (access_log)

```sql
CREATE TABLE access_log (
    id VARCHAR(64) NOT NULL COMMENT '主键',
    user_id VARCHAR(64) COMMENT '用户ID',
    request_url VARCHAR(500) NOT NULL COMMENT '请求URL',
    request_method VARCHAR(10) NOT NULL COMMENT '请求方法',
    request_params TEXT COMMENT '请求参数',
    request_body TEXT COMMENT '请求体',
    response_status INT NOT NULL COMMENT '响应状态码',
    response_body TEXT COMMENT '响应体',
    duration BIGINT NOT NULL COMMENT '耗时(ms)',
    ip VARCHAR(128) COMMENT 'IP地址',
    user_agent VARCHAR(500) COMMENT 'User-Agent',
    referer VARCHAR(500) COMMENT 'Referer',
    access_time DATETIME NOT NULL COMMENT '访问时间',
    created_at DATETIME NOT NULL  COMMENT '创建时间',
    PRIMARY KEY (id),
    KEY idx_user_id (user_id),
    KEY idx_request_url (request_url(255)),
    KEY idx_response_status (response_status),
    KEY idx_access_time (access_time),
    KEY idx_duration (duration)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='访问日志表'
PARTITION BY RANGE (YEAR(access_time) * 100 + MONTH(access_time)) (
    PARTITION p202601 VALUES LESS THAN (202602),
    -- ... 其他分区
    PARTITION pmax VALUES LESS THAN MAXVALUE
);
```

#### 4.1.4 错误日志表 (error_log)

```sql
CREATE TABLE error_log (
    id VARCHAR(64) NOT NULL COMMENT '主键',
    user_id VARCHAR(64) COMMENT '用户ID',
    error_type VARCHAR(50) NOT NULL COMMENT '错误类型: BUSINESS/SYSTEM/THIRD_PARTY',
    error_code VARCHAR(50) COMMENT '错误码',
    error_message TEXT NOT NULL COMMENT '错误信息',
    stack_trace TEXT COMMENT '堆栈信息',
    request_url VARCHAR(500) COMMENT '请求URL',
    request_method VARCHAR(10) COMMENT '请求方法',
    request_params TEXT COMMENT '请求参数',
    ip VARCHAR(128) COMMENT 'IP地址',
    error_time DATETIME NOT NULL COMMENT '错误时间',
    created_at DATETIME NOT NULL  COMMENT '创建时间',
    PRIMARY KEY (id),
    KEY idx_error_type (error_type),
    KEY idx_error_code (error_code),
    KEY idx_error_time (error_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='错误日志表'
PARTITION BY RANGE (YEAR(error_time) * 100 + MONTH(error_time)) (
    PARTITION p202601 VALUES LESS THAN (202602),
    -- ... 其他分区
    PARTITION pmax VALUES LESS THAN MAXVALUE
);
```

### 4.2 分区策略

- **按月分区**: 每月一个分区
- **自动创建**: 定时任务自动创建未来3个月分区
- **自动归档**: 6个月前的分区自动归档到历史表
- **自动清理**: 12个月前的历史表可选清理

```sql
-- 定时创建分区的存储过程
DELIMITER $$
CREATE PROCEDURE create_log_partitions()
BEGIN
    DECLARE v_year INT;
    DECLARE v_month INT;
    DECLARE v_partition_name VARCHAR(20);
    DECLARE v_partition_value INT;
    
    -- 获取3个月后的年月
    SET v_year = YEAR(DATE_ADD(NOW(), INTERVAL 3 MONTH));
    SET v_month = MONTH(DATE_ADD(NOW(), INTERVAL 3 MONTH));
    SET v_partition_name = CONCAT('p', v_year, LPAD(v_month, 2, '0'));
    SET v_partition_value = v_year * 100 + v_month + 1;
    
    -- 为每个日志表添加分区
    SET @sql = CONCAT('ALTER TABLE operation_log ADD PARTITION IF NOT EXISTS (PARTITION ', v_partition_name, ' VALUES LESS THAN (', v_partition_value, '))');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- 重复其他表...
END$$
DELIMITER ;
```

---

## 5. API 设计

### 5.1 操作日志 API

```java
/**
 * 操作日志 API
 */
@RestController
@RequestMapping("/api/v1/log/operation")
@RequiredArgsConstructor
@Tag(name = "操作日志")
public class OperationLogController {
    
    /**
     * 查询操作日志列表
     */
    @PostMapping("/list")
    @Operation(summary = "查询操作日志列表")
    @RequirePermission("log:operation:query")
    public Result<PagedResult<OperationLogDTO>> list(
        @RequestBody @Valid QueryOperationLogQry qry) {
        return Result.success(queryExecutor.execute(qry));
    }
    
    /**
     * 获取操作日志详情
     */
    @PostMapping("/get")
    @Operation(summary = "获取操作日志详情")
    @RequirePermission("log:operation:query")
    public Result<OperationLogDTO> get(@RequestBody @Valid GetLogQry qry) {
        return Result.success(queryExecutor.execute(qry));
    }
    
    /**
     * 导出操作日志
     */
    @PostMapping("/export")
    @Operation(summary = "导出操作日志")
    @RequirePermission("log:operation:export")
    public void export(@RequestBody @Valid ExportLogQry qry, HttpServletResponse response) {
        // 导出Excel
    }
    
    /**
     * 统计操作日志
     */
    @PostMapping("/statistics")
    @Operation(summary = "统计操作日志")
    @RequirePermission("log:operation:query")
    public Result<LogStatisticsDTO> statistics(@RequestBody @Valid StatisticsQry qry) {
        return Result.success(queryExecutor.execute(qry));
    }
}
```

### 5.2 审计日志 API

```java
/**
 * 审计日志 API
 */
@RestController
@RequestMapping("/api/v1/log/audit")
@RequiredArgsConstructor
@Tag(name = "审计日志")
public class AuditLogController {
    
    /**
     * 查询审计日志列表
     */
    @PostMapping("/list")
    @Operation(summary = "查询审计日志列表")
    @RequirePermission("log:audit:query")
    public Result<PagedResult<AuditLogDTO>> list(@RequestBody @Valid QueryAuditLogQry qry) {
        return Result.success(queryExecutor.execute(qry));
    }
    
    /**
     * 获取数据变更详情
     */
    @PostMapping("/get-diff")
    @Operation(summary = "获取数据变更详情")
    @RequirePermission("log:audit:query")
    public Result<DataDiffDTO> getDiff(@RequestBody @Valid GetDiffQry qry) {
        return Result.success(queryExecutor.execute(qry));
    }
    
    /**
     * 审计报告生成
     */
    @PostMapping("/report")
    @Operation(summary = "生成审计报告")
    @RequirePermission("log:audit:report")
    public Result<AuditReportDTO> generateReport(@RequestBody @Valid AuditReportQry qry) {
        return Result.success(queryExecutor.execute(qry));
    }
}
```

---

## 6. 技术实现

### 6.1 审计日志写入

审计日志采用**手动调用**方式，由业务方在关键操作处直接调用 `AuditLogService.record()`，传入必要字段（dataType / dataId /
action / reason / ip）。不实现自动 AOP 拦截和变更前后对比，原因如下：

- 变更前后数据对比价值有限，增加系统复杂度
- 业务方对"什么信息值得记录"更清楚，手动调用更精确
- 避免反射/序列化带来的性能开销和脆弱性

```java
// 业务代码中手动记录审计日志
auditLogService.record(userId, username, "CONFIG", configKey,
    "UPDATE", null, null, "修复生产配置", request.getRemoteAddr());
```

### 6.2 日志归档

```java
/**
 * 日志归档服务
 */
@Service
@Slf4j
public class LogArchiveService {
    
    /**
     * 归档历史日志
     * 
     * 定时任务: 每月1号凌晨2点执行
     */
    @Scheduled(cron = "0 0 2 1 * ?")
    public void archiveOldLogs() {
        LocalDate sixMonthsAgo = LocalDate.now().minusMonths(6);
        int year = sixMonthsAgo.getYear();
        int month = sixMonthsAgo.getMonthValue();
        
        String partitionName = String.format("p%04d%02d", year, month);
        
        // 1. 导出数据到归档表
        archivePartition("operation_log", partitionName);
        archivePartition("audit_log", partitionName);
        archivePartition("access_log", partitionName);
        
        // 2. 删除分区
        dropPartition("operation_log", partitionName);
        dropPartition("audit_log", partitionName);
        dropPartition("access_log", partitionName);
        
        log.info("日志归档完成: partition={}", partitionName);
    }
    
    private void archivePartition(String tableName, String partitionName) {
        String archiveTable = tableName + "_archive";
        String sql = String.format(
            "INSERT INTO %s SELECT * FROM %s PARTITION (%s)",
            archiveTable, tableName, partitionName
        );
        jdbcTemplate.execute(sql);
    }
}
```

---

## 7. 性能优化

### 7.1 异步写入

- 独立线程池处理日志写入
- 线程池配置: 核心10个，最大50个线程
- 批量写入: 每批1000条

### 7.2 分区表

- 按月分区，提升查询性能
- 历史数据归档，控制表大小

### 7.3 索引优化

- 查询热点字段建索引
- 复合索引优化多条件查询

### 7.4 性能指标

- 日志写入: < 10ms (异步)
- 日志查询: < 500ms
- 吞吐量: 10000+ TPS

---

## 8. 测试方案

### 8.1 单元测试

```java
@SpringBootTest
class OperationLogServiceTest {
    
    @Test
    void testSaveOperationLog() {
        // 测试保存操作日志
    }
    
    @Test
    void testQueryOperationLog() {
        // 测试查询操作日志
    }
}
```

### 8.2 集成测试

```java
@SpringBootTest
@EnableTestContainers(ContainerType.MYSQL)
class LogIntegrationTest {
    
    @Test
    void testAopLogging() {
        // 测试AOP自动记录日志
    }
}
```

### 8.3 性能测试

```java
@Test
void testConcurrentLogging() {
    // JMH 压测: 10000+ TPS
}
```

---

## 9. 实施计划

> **📅 当前状态（2026-02-28 更新）**

### ✅ 已完成（第一阶段）

#### 模块结构

- [x] 5 个子模块 pom.xml（parent 均指向根 loadup-parent）
- [x] modules/pom.xml 注册 loadup-modules-log

#### Client 层

- [x] `@OperationLog` 注解（type / module / description / recordParams / recordResponse）
- [x] `OperationLogDTO` / `AuditLogDTO`
- [x] `OperationLogQuery` / `AuditLogQuery`

#### Domain 层（纯 POJO，无框架注解）

- [x] `OperationType` 枚举
- [x] `OperationLog` domain model
- [x] `AuditLog` domain model
- [x] `OperationLogGateway` 接口
- [x] `AuditLogGateway` 接口

#### Infrastructure 层

- [x] `OperationLogDO` extends BaseDO（`@Table("operation_log")`）
- [x] `AuditLogDO` extends BaseDO（`@Table("audit_log")`）
- [x] `OperationLogGatewayImpl`（MyBatis-Flex QueryWrapper，分页查询）
- [x] `AuditLogGatewayImpl`（MyBatis-Flex QueryWrapper，分页查询）
- [x] `LogAsyncWriter`（`@Async("logExecutor")`，独立线程池）
- [x] `OperationLogAspect`（AOP `@Around`，无侵入拦截 `@OperationLog`）
- [x] Flyway `V1__init_log.sql`（operation_log + audit_log 建表）

#### App 层

- [x] `OperationLogService`（listByCondition / countByCondition / record）
- [x] `AuditLogService`（listByCondition / countByCondition / record）
- [x] `LogModuleAutoConfiguration`（线程池 + AOP + @EnableAsync + @MapperScan）
- [x] `AutoConfiguration.imports` 注册

#### Gateway 路由（routes.csv）

- [x] operation log：list / count / record（3 条）
- [x] audit log：list / count / record（3 条）
- [x] loadup-application pom.xml 引入 `loadup-modules-log-app`

#### 测试

- [x] `OperationLogServiceIT`（7 个用例：persist / filterByUserId / filterByModule / filterBySuccess / count /
  pagination / asyncRecord）
- [x] `AuditLogServiceIT`（5 个用例：persist / filterByDataType / filterByDataId / count / asyncRecord）
- [x] `@EnableTestContainers(ContainerType.MYSQL)` 真实 MySQL 容器
- [x] `BeforeEach` 清理脏数据

---

### ❌ 未完成项（P2）

| 优先级 | 项目                | 说明                           |
|-----|-------------------|------------------------------|
| P2  | 日志导出（Excel/CSV）   | `OperationLogService.export` |
| P2  | 统计分析接口            | 按模块/操作类型/时间段聚合统计             |
| P2  | 日志归档              | 按月分区 + 定时归档历史数据              |
| P2  | 错误日志（error_log 表） | 全局异常处理器自动记录                  |

### 9.3 验收标准

- [x] 核心功能编译通过（无 ERROR）
- [x] 集成测试覆盖 CRUD + 分页 + 异步场景（12 个用例）
- [x] Gateway 路由注册完成（6 条）
- [x] Flyway migration 就绪
- [x] 异步写入不阻塞业务线程（独立线程池）
- [ ] 单元测试覆盖率 > 80%（当前约 70%）
- [ ] Code Review 通过

