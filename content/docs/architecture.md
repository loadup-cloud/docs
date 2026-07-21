# LoadUp Framework - 架构设计文档

## 1. 架构概述

LoadUp Framework 是一个基于 Spring Boot 4.1.0 的企业级微服务开发框架，采用 **Monorepo（单仓库）多模块架构**
，提供可复用的基础组件和最佳实践，帮助团队快速构建高质量的企业应用。

### 1.1 设计理念

- **模块化设计**: 清晰的模块边界，单一职责原则
- **依赖倒置**: 高层模块不依赖低层模块，都依赖抽象
- **可扩展性**: 插件化架构，支持业务定制扩展
- **开箱即用**: 自动配置，最小化配置原则
- **企业级**: 高性能、高可用、可观测、安全

### 1.2 技术栈

| 技术领域   | 技术选型                             | 版本     |
|--------|----------------------------------|--------|
| 编程语言   | Java                             | 21     |
| 应用框架   | Spring Boot                      | 4.1.0  |
| 持久层框架  | MyBatis-Flex                     | 1.11.7 |
| 数据库    | MySQL                            | 8.0+   |
| 缓存     | Redis (Redisson)                 | -      |
| 消息队列   | (待集成)                            | -      |
| 服务治理   | Dubbo                            | 3.3.6  |
| 链路追踪   | OpenTelemetry                    | 1.62.0 |
| 任务调度   | Quartz/XXL-Job/PowerJob          | -      |
| 认证授权   | JWT                              | -      |
| API 文档 | knife4j + OpenAPI (Swagger)      | 4.5.0  |
| 测试框架   | JUnit 5, Mockito, Testcontainers | -      |
| 构建工具   | Maven                            | 3.6+   |

## 2. 模块架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    LoadUp Framework                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐          ┌─────────────────┐          │
│  │  Application    │          │    Gateway      │          │
│  │  (启动器/主应用)   │◄────────►│   (API 网关)    │          │
│  └────────┬────────┘          └────────┬────────┘          │
│           │                            │                    │
│           ▼                            ▼                    │
│  ┌──────────────────────────────────────────────┐          │
│  │            Modules (业务模块)                  │          │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │          │
│  │  │   UPMS   │  │   ...    │  │   ...    │  │          │
│  │  └──────────┘  └──────────┘  └──────────┘  │          │
│  └────────────────────┬──────────────────────┘          │
│                       │                                    │
│                       ▼                                    │
│  ┌─────────────────────────────────────────────────────┐  │
│  │         Components (技术组件层)                        │  │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐      │  │
│  │  │Database│ │ Cache  │ │Scheduler│ │ Tracer │ ...  │  │
│  │  └────────┘ └────────┘ └────────┘ └────────┘      │  │
│  └────────────────────┬────────────────────────────────┘  │
│                       │                                    │
│                       ▼                                    │
│  ┌─────────────────────────────────────────────────────┐  │
│  │          Commons (通用基础层)                         │  │
│  │  ┌────────┐ ┌────────┐ ┌────────┐                  │  │
│  │  │  API   │ │  DTO   │ │  Util  │                  │  │
│  │  └────────┘ └────────┘ └────────┘                  │  │
│  └─────────────────────────────────────────────────────┘  │
│                       │                                    │
│                       ▼                                    │
│  ┌─────────────────────────────────────────────────────┐  │
│  │       Dependencies (依赖管理 BOM)                     │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐  │
│  │         Testify (测试框架 - 独立)                     │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 模块依赖关系

**严格单向依赖规则**：

```
dependencies (基础)
    ↑
commons (通用层)
    ↑
components (组件层)
    ↑
modules (业务模块层)
    ↑
application (应用层)

gateway (独立) ← → commons, components
testify (独立测试框架)
```

**依赖原则**：

1. ✅ 下层模块可以被上层模块依赖
2. ❌ 禁止反向依赖（上层不能被下层依赖）
3. ❌ 禁止跨层依赖（必须逐层依赖）
4. ❌ 严禁循环依赖

### 2.3 核心模块说明

#### 2.3.1 loadup-dependencies

**职责**: 统一依赖版本管理（BOM）

**内容**:

- 所有第三方依赖版本定义
- LoadUp 框架内部模块版本管理
- Maven 插件配置（Spotless、Compiler、Test 等）

**特点**:

- 采用 `<dependencyManagement>` 统一管理版本
- 子模块通过 `<scope>import</scope>` 引入
- 确保整个框架版本一致性

#### 2.3.2 loadup-commons

**职责**: 提供通用的基础工具类和类型定义

**子模块**:

- `loadup-commons-api`: 通用接口、异常、枚举定义
- `loadup-commons-dto`: 数据传输对象、Result 封装、分页对象
- `loadup-commons-util`: 工具类（日期、字符串、集合、JSON 等）

**特点**:

- 零业务逻辑
- 高度可复用
- 最小依赖（避免引入过多第三方库）

#### 2.3.3 loadup-components

**职责**: 提供可复用的技术组件，屏蔽底层技术细节

**子模块**:

| 组件               | 功能      | 说明                              |
|------------------|---------|---------------------------------|
| `database`       | 数据库访问   | MyBatis-Flex 集成、审计、多租户、逻辑删除     |
| `cache`          | 缓存抽象    | 支持 Redis/Caffeine，统一缓存接口        |
| `captcha`        | 验证码     | 图形验证码生成与验证                      |
| `scheduler`      | 任务调度    | 支持 Quartz/XXL-Job/PowerJob 多种实现 |
| `tracer`         | 链路追踪    | 基于 OpenTelemetry 的分布式追踪         |
| `dfs`            | 分布式文件存储 | S3 兼容的文件存储抽象                    |
| `liquibase`      | 数据库版本管理 | Liquibase 集成                    |
| `extension`      | 扩展点机制   | 类似 SPI 的扩展点框架                   |
| `gotone`         | 多租户支持   | 租户隔离与上下文管理                      |
| `testcontainers` | 集成测试支持  | Testcontainers 集成               |
| `web`            | Web 增强  | 全局异常处理、统一响应等                    |

**设计原则**:

- **抽象优先**: 定义统一接口，支持多种实现
- **自动配置**: 利用 Spring Boot Auto-Configuration
- **零侵入**: 不强制业务代码依赖特定实现
- **开关控制**: 通过配置启用/禁用功能

#### 2.3.4 loadup-modules

**职责**: 业务功能模块

**子模块**:

- `loadup-modules-upms`: 用户权限管理系统（User Permission Management System）
    - 用户管理
    - 角色管理
    - 权限管理
    - 菜单管理
    - 部门管理
    - 数据权限

**架构风格**: COLA 4.0 分层架构

- `adapter`: 适配器层（REST API、DTO 转换）
- `application`: 应用层（Command/Query Handler）
- `domain`: 领域层（实体、值对象、Repository 接口）
- `infrastructure`: 基础设施层（Repository 实现、外部服务调用）

#### 2.3.5 loadup-gateway

**职责**: API 网关

**子模块**:

- `loadup-gateway-facade`: API 定义
- `loadup-gateway-core`: 核心实现
- `loadup-gateway-starter`: Spring Boot Starter
- `plugins/`: 插件实现
    - `proxy-http-plugin`: HTTP 代理
    - `proxy-rpc-plugin`: RPC 代理（Dubbo）
    - `proxy-springbean-plugin`: Spring Bean 代理
    - `repository-file-plugin`: 文件配置存储
    - `repository-database-plugin`: 数据库配置存储

**特点**:

- 插件化架构
- 支持多种代理类型
- 动态路由配置

#### 2.3.6 loadup-testify

**职责**: 声明式自动化测试框架

**特点**:

- YAML 驱动的测试定义
- 动态 Mock（基于 AOP）
- 数据驱动测试（TestNG DataProvider）
- 复杂断言支持（JSONAssert、JsonPath）

**子模块**:

- `loadup-testify-core`: 核心引擎
- `loadup-testify-assert-engine`: 断言引擎
- `loadup-testify-mock-engine`: Mock 引擎
- `loadup-testify-data-engine`: 数据引擎
- `loadup-testify-infra-container`: 基础设施容器
- `loadup-testify-spring-boot-starter`: Spring Boot 集成

#### 2.3.7 loadup-application

**职责**: 应用启动器

**内容**:

- Spring Boot 主应用类
- 配置文件（application.yml）
- 聚合所有业务模块

## 3. 分层架构

### 3.1 整体分层

```
┌─────────────────────────────────────────┐
│         表现层 (Presentation)             │  Controller, REST API
├─────────────────────────────────────────┤
│          应用层 (Application)             │  Service, 业务编排
├─────────────────────────────────────────┤
│           领域层 (Domain)                 │  Entity, VO, Repository Interface
├─────────────────────────────────────────┤
│        基础设施层 (Infrastructure)         │  Mapper, Cache, External API
├─────────────────────────────────────────┤
│         数据层 (Data)                     │  MySQL, Redis
└─────────────────────────────────────────┘
```

### 3.2 UPMS 模块分层（COLA 4.0）

```
┌─────────────────────────────────────────┐
│  Adapter Layer (upms-adapter)            │
│  - Controller (REST API)                 │
│  - DTO Converter                         │
│  - Request/Response DTO                  │
└────────────────┬────────────────────────┘
                 │ depends on
┌────────────────▼────────────────────────┐
│  Application Layer (upms-application)    │
│  - Command Handler                       │
│  - Query Handler                         │
│  - Service (业务编排)                     │
└────────────────┬────────────────────────┘
                 │ depends on
┌────────────────▼────────────────────────┐
│  Domain Layer (upms-domain)              │
│  - Entity (充血模型)                      │
│  - Value Object                          │
│  - Domain Service                        │
│  - Repository Interface                  │
└────────────────▲────────────────────────┘
                 │ implemented by
┌────────────────┴────────────────────────┐
│  Infrastructure Layer (upms-infrastructure)│
│  - Repository Impl (MyBatis-Flex Mapper)│
│  - Cache Impl                            │
│  - External API Client                   │
│  - Config                                │
└─────────────────────────────────────────┘
```

## 4. 核心设计模式

### 4.1 策略模式 (Strategy Pattern)

**应用场景**:

- ID 生成策略（Random/UUID v4/UUID v7/Snowflake）
- 缓存实现策略（Redis/Caffeine）
- 调度器实现策略（Quartz/XXL-Job/PowerJob）

**实现方式**:

```java
// 定义策略接口
public interface IdGenerator {
    String generate();
}

// 多种实现
@Component("randomIdGenerator")
public class RandomIdGenerator implements IdGenerator { ... }

@Component("snowflakeIdGenerator")
public class SnowflakeIdGenerator implements IdGenerator { ... }

// 策略选择
@Autowired
@Qualifier("${loadup.database.id-generator.strategy}")
private IdGenerator idGenerator;
```

### 4.2 模板方法模式 (Template Method Pattern)

**应用场景**:

- BaseMapper 提供通用 CRUD 方法
- 测试基类提供测试流程模板

### 4.3 观察者模式 (Observer Pattern)

**应用场景**:

- 审计回调（BeforeSaveCallback）
- 事件驱动（Spring Event）

### 4.4 代理模式 (Proxy Pattern)

**应用场景**:

- Gateway 路由代理
- AOP 拦截（事务、缓存、日志）
- Testify Mock 拦截

### 4.5 工厂模式 (Factory Pattern)

**应用场景**:

- Spring Bean 工厂
- 插件工厂（Gateway Plugin Factory）

## 5. 数据库设计规范

### 5.1 表命名规范

- **系统表前缀**: `sys_`（如 `sys_config`）
- **业务表前缀**: `t_`（如 `t_user`, `t_order`）
- **关联表**: `{table1}_{table2}`（如 `user_role`）
- **命名风格**: snake_case（小写 + 下划线）

### 5.2 字段规范

**基础审计字段**（所有表必须包含）:

```sql
id              VARCHAR(64)  NOT NULL PRIMARY KEY COMMENT '主键ID',
created_by      VARCHAR(64)  COMMENT '创建人',
created_time    DATETIME     COMMENT '创建时间',
updated_by      VARCHAR(64)  COMMENT '更新人',
updated_time    DATETIME     COMMENT '更新时间'
```

**多租户字段**（需要租户隔离的表）:

```sql
tenant_id       VARCHAR(64)  NOT NULL COMMENT '租户ID'
```

**逻辑删除字段**（需要软删除的表）:

```sql
deleted         TINYINT(1)   DEFAULT 0 COMMENT '是否删除 0-否 1-是'
```

### 5.3 索引规范

- **主键**: 使用字符串 ID（由 ID 生成器生成）
- **唯一索引**: `uk_{column_name}`
- **普通索引**: `idx_{column_name}`
- **联合索引**: `idx_{table}_{column1}_{column2}`
- **多租户索引**: 必须在 `tenant_id` 上建索引

## 6. API 设计规范

### 6.1 RESTful API 规范

**URL 设计**:

```
GET    /api/{module}/{resource}           # 列表查询
GET    /api/{module}/{resource}/{id}      # 单个查询
POST   /api/{module}/{resource}           # 创建
PUT    /api/{module}/{resource}/{id}      # 更新
DELETE /api/{module}/{resource}/{id}      # 删除
```

**统一响应格式**:

```json
{
  "code": "0",
  "message": "success",
  "data": { ... },
  "timestamp": "2025-02-04T10:00:00Z"
}
```

### 6.2 分页查询

**请求参数**:

```java
public class PageQuery {
    private Integer pageNum = 1;   // 页码（从 1 开始）
    private Integer pageSize = 10; // 每页大小
    private String sortBy;         // 排序字段
    private String sortOrder;      // 排序方向（asc/desc）
}
```

**响应格式**:

```json
{
  "code": "0",
  "message": "success",
  "data": {
    "records": [ ... ],
    "total": 100,
    "pageNum": 1,
    "pageSize": 10,
    "pages": 10
  }
}
```

## 7. 安全设计

### 7.1 认证与授权

- **认证方式**: JWT Token
- **Token 存储**: Header: `Authorization: Bearer {token}`
- **权限控制**: 基于 RBAC（角色 + 权限）
- **数据权限**: 支持多级数据范围（全部/部门/本人）

### 7.2 敏感信息保护

- 密码字段必须标注 `@JsonIgnore`
- 敏感信息脱敏（手机号、身份证、邮箱）
- 日志中禁止打印 Token 和密码

### 7.3 SQL 注入防护

- 严禁字符串拼接 SQL
- 使用 MyBatis-Flex QueryWrapper 或参数化查询
- 对用户输入进行严格校验

## 8. 性能优化策略

### 8.1 缓存策略

**多级缓存**:

```
Request → 本地缓存 (Caffeine) → 分布式缓存 (Redis) → 数据库
```

**缓存模式**:

- Cache-Aside（旁路缓存）
- Read-Through（读穿透）
- Write-Through（写穿透）

### 8.2 数据库优化

- **连接池**: HikariCP（默认）
- **批量操作**: `insertBatch` / `updateBatch`
- **分页查询**: MyBatis-Flex Page 对象
- **慢查询监控**: p6spy 日志监控

### 8.3 并发控制

- 乐观锁（version 字段）
- 分布式锁（Redis/Redisson）
- 限流（Resilience4j RateLimiter）

## 9. 可观测性

### 9.1 日志

- **框架**: SLF4J + Logback
- **级别**: ERROR < WARN < INFO < DEBUG < TRACE
- **格式**: JSON 格式（方便日志采集）
- **MDC**: 请求 ID、用户 ID、租户 ID

### 9.2 链路追踪

- **框架**: OpenTelemetry
- **集成**: Jaeger/Zipkin
- **采样率**: 可配置

### 9.3 指标监控

- Spring Boot Actuator
- Micrometer + Prometheus
- 自定义业务指标

## 10. 测试策略

### 10.1 测试金字塔

```
       ┌──────────────┐
       │  End-to-End  │  (少量, Testify 集成测试)
       └──────────────┘
      ┌────────────────┐
      │  Integration   │  (适量, Testcontainers)
      └────────────────┘
    ┌──────────────────┐
    │   Unit Tests     │  (大量, JUnit + Mockito)
    └──────────────────┘
```

### 10.2 测试工具

- **单元测试**: JUnit 5 + Mockito + AssertJ
- **集成测试**: Testcontainers (MySQL, Redis)
- **接口测试**: Testify (YAML 驱动)
- **覆盖率**: JaCoCo

## 11. 部署架构

### 11.1 容器化部署

```
┌────────────────────────────────────────┐
│          Kubernetes Cluster             │
│  ┌──────────┐  ┌──────────┐           │
│  │  App Pod │  │  App Pod │  ...      │
│  └────┬─────┘  └────┬─────┘           │
│       │             │                  │
│  ┌────▼─────────────▼──────┐          │
│  │      Service (LB)        │          │
│  └────────────┬─────────────┘          │
└───────────────┼────────────────────────┘
                │
         ┌──────▼──────┐
         │   Ingress   │
         └─────────────┘
```

### 11.2 配置管理

- **配置中心**: Nacos / Spring Cloud Config
- **环境隔离**: dev / test / staging / prod
- **敏感配置**: 加密存储

## 12. 未来规划

### 12.1 短期目标（3-6 个月）

- [ ] 完善 Gateway 功能（限流、熔断、降级）
- [ ] 集成消息队列（Kafka/RabbitMQ）
- [ ] 增强监控告警能力
- [ ] 完善文档和示例

### 12.2 中期目标（6-12 个月）

- [ ] 支持多数据库（PostgreSQL、Oracle）
- [ ] 分布式事务支持（Seata）
- [ ] 服务网格集成（Istio）
- [ ] 低代码平台

### 12.3 长期愿景

- 打造完整的企业级开发平台
- 提供可视化配置工具
- 支持多语言客户端（Go、Python、Node.js）
- 构建活跃的开源社区

## 13. 参考资源

- [Spring Boot 官方文档](https://spring.io/projects/spring-boot)
- [MyBatis-Flex 文档](https://mybatis-flex.com/)
- [COLA 架构](https://github.com/alibaba/COLA)
- [阿里巴巴 Java 开发手册](https://github.com/alibaba/p3c)
- [领域驱动设计（DDD）](https://www.domainlanguage.com/ddd/)

---

**© 2025 LoadUp Framework. All rights reserved.**
