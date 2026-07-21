# LoadUp Framework - 项目规划与路线图

> **版本**: 0.0.2-SNAPSHOT  
> **更新日期**: 2026-05-18  
> **项目定位**: 企业级应用开发框架  
> **发展策略**: 巨石应用优先，保留分布式能力扩展  
> **核心理念**: 先把单体应用做到极致，再考虑分布式拆分

## 📋 目录

- [1. 项目现状分析](#1-项目现状分析)
- [2. 架构优势与不足](#2-架构优势与不足)
- [3. 发展路线图](#3-发展路线图)
- [4. 后续优化方向](#4-后续优化方向)
- [5. 通用业务模块规划](#5-通用业务模块规划)
- [6. 技术演进路线](#6-技术演进路线)

---

## 1. 项目现状分析

### 1.1 已完成的核心能力

#### ✅ 基础设施层 (Commons)

- **loadup-commons-api**: 通用接口定义
- **loadup-commons-dto**: 统一数据传输对象
- **loadup-commons-util**: 工具类库

#### ✅ 技术组件层 (Components) - 17个组件

**安全能力**:

- `loadup-components-authorization` - 轻量级方法级授权框架
- `loadup-components-captcha` - 验证码生成与验证
- `loadup-components-signature` - 数字签名和摘要计算

**数据能力**:

- `loadup-components-database` - 数据库访问和工具
- `loadup-components-cache` - 缓存抽象 (Caffeine/Redis)
- `loadup-components-dfs` - 分布式文件存储 (Local/S3/Database)
- `loadup-components-flyway` - 数据库版本管理

**业务能力**:

- `loadup-components-gotone` - 统一消息通知 (Email/SMS/Push/Webhook)
- `loadup-components-retrytask` - 分布式重试任务框架
- `loadup-components-globalunique` - 全局幂等性控制
- `loadup-components-scheduler` - 任务调度 (Quartz/XXL-Job/PowerJob)

**工程能力**:

- `loadup-components-extension` - 插件化扩展框架
- `loadup-components-pipeline` - 管道流水线处理框架
- `loadup-components-configcenter` - 配置中心绑定器 (Apollo/Nacos/Local)
- `loadup-components-tracer` - 分布式链路追踪 (OpenTelemetry 1.62.0，SPI 多导出器)
- `loadup-components-testcontainers` - 测试容器支持

#### ✅ 中间件层 (Middleware)

- **loadup-gateway**: 自研轻量级 API 网关
    - 基于 Spring MVC (非 WebFlux)
    - Action 责任链模式
    - 支持多种认证策略 (JWT/签名/自定义)
    - 集成链路追踪
    - 统一异常处理和响应封装

- **loadup-testify**: 测试框架

#### ✅ 业务模块层 (Modules)

- **loadup-modules-upms**: 用户权限管理系统
    - RBAC3 权限模型 (角色继承、职责分离、数据权限)
    - 组织架构管理 (无限层级部门树)
    - 多种登录方式 (用户名/邮箱/手机)
    - OAuth 2.0 第三方登录 (微信/QQ/GitHub/Google等9种渠道)
    - 操作审计和登录日志
- **loadup-modules-config**: 系统配置管理 + 数据字典管理
    - 系统参数 (key-value) 管理
    - 数据字典类型与字典项管理
- **loadup-modules-log**: 业务日志中心
    - 操作日志自动记录 (AOP)
    - 审计日志记录

### 1.2 技术栈选型

| 分类        | 技术               | 版本     | 评估             | 优先级 |
|-----------|------------------|--------|----------------|-----|
| **核心框架**  | Spring Boot      | 4.1.0  | ✅ 最新稳定版        | P0  |
| **编程语言**  | Java             | 21     | ✅ LTS版本，虚拟线程支持 | P0  |
| **ORM框架** | MyBatis-Flex     | 1.11.7 | ✅ 类型安全，性能优秀    | P0  |
| **数据库**   | MySQL            | 8.0+   | ✅ 主流选择         | P0  |
| **缓存**    | Redis (Redisson) | -      | ✅ 已集成          | P0  |
| **API网关** | 自研 Gateway       | -      | ✅ 轻量级，满足单体需求   | P0  |
| **链路追踪**  | OpenTelemetry    | 1.62.0 | ✅ 已集成          | P1  |
| **任务调度**  | Quartz/XXL-Job   | -      | ✅ 多种选择         | P1  |
| **消息队列**  | -                | -      | ⏸️ 暂缓，保留扩展能力   | P3  |
| **服务注册**  | -                | -      | ⏸️ 分布式阶段再考虑    | P3  |
| **配置中心**  | -                | -      | ⏸️ 分布式阶段再考虑    | P3  |
| **RPC框架** | Dubbo            | 3.2.8  | ⏸️ 已集成但暂不使用    | P3  |

### 1.3 代码质量保障

| 工具            | 配置                              | 状态                      |
|---------------|---------------------------------|-------------------------|
| **代码格式化**     | Spotless (Palantir Java Format) | ✅ 已配置，CI 强制检查           |
| **单元测试**      | JUnit 5 + Mockito + AssertJ     | ✅ 已配置                   |
| **集成测试**      | Testcontainers                  | ✅ 已配置                   |
| **代码覆盖率**     | JaCoCo                          | ✅ 已配置                   |
| **静态代码分析**    | SpotBugs + FindSecBugs          | ✅ 已配置，CI 强制检查           |
| **代码规范检查**    | PMD + Checkstyle                | ✅ 已配置，CI 强制检查           |
| **架构约束检查**    | ArchUnit                        | ✅ 已配置，覆盖分层依赖/循环依赖/命名规范  |
| **安全漏洞扫描**    | OWASP Dependency-Check          | ✅ 已配置，main/develop 分支触发 |
| **License检查** | License Maven Plugin            | ✅ 已配置                   |
| **依赖管理**      | Maven BOM                       | ✅ 已配置                   |
| **API文档**     | OpenAPI/Swagger                 | ✅ 已集成                   |

---

## 2. 架构优势与不足

### 2.1 核心优势 💪

1. **清晰的分层架构**
    - 严格的单向依赖规则
    - 模块职责明确，边界清晰
    - 符合 DDD 分层思想

2. **高度模块化**
    - 15个独立可复用组件
    - 插件化扩展机制
    - 组件间松耦合

3. **企业级最佳实践**
    - 统一异常处理
    - 统一响应格式
    - 完善的审计日志
    - 安全机制完备

4. **开发效率高**
    - 自动配置优先
    - 丰富的工具类
    - 完善的文档

5. **测试友好**
    - Testcontainers 集成测试
    - 高代码覆盖率要求
    - 分离单元测试和集成测试

### 2.2 当前不足与待改进 🔧

#### 核心能力层面 (高优先级)

1. **基础能力建设不完善** �
    - ✅ 配置管理模块 (loadup-modules-config 已创建)
    - ✅ 业务日志中心 (loadup-modules-log 已创建)
    - ❌ 缺少系统监控和告警 (Prometheus + Grafana)
    - ❌ 测试覆盖率不足 (当前约40%)
    - ❌ 性能基准测试缺失

2. **代码质量保障** 🟡
    - ✅ Spotless 代码格式化 (已集成，CI 强制检查)
    - ✅ SpotBugs + PMD + Checkstyle 静态分析 (已集成)
    - ✅ ArchUnit 架构约束检查 (已集成)
    - ✅ OWASP Dependency-Check 安全漏洞扫描 (已集成)
    - ❌ SonarQube 未集成
    - ❌ Mutation Testing (PIT) 未集成

3. **业务模块扩展** 🟡
    - ✅ loadup-modules-upms (用户权限管理)
    - ✅ loadup-modules-config (系统配置管理，已创建)
    - ✅ loadup-modules-log (业务日志中心，已创建)
    - ❌ 缺少 Notification 消息中心等其他通用模块
    - ❌ 缺少业务模块间协作示例

4. **性能优化空间大** 🟠
    - ❌ 缺少数据库连接池优化
    - ❌ 缺少查询性能监控
    - ❌ 缺少缓存策略优化
    - ❌ 批量操作未充分利用

5. **开发体验待提升** 🟡
    - ❌ 缺少代码生成器
    - ❌ 缺少开发者工具
    - ⚠️ API文档需要完善
    - ❌ 缺少最佳实践示例

#### 分布式能力层面 (低优先级，保留扩展)

6. **微服务能力** 🟢 (P3)
    - 暂不需要：服务注册与发现
    - 暂不需要：配置中心
    - 暂不需要：服务网格
    - 保留扩展：Dubbo已集成但暂不使用

7. **消息驱动能力** 🟢 (P3)
    - 暂不需要：消息队列
    - 暂不需要：事件驱动架构
    - 暂不需要：分布式事务
    - 可替代：使用本地事件机制

#### DevOps 能力层面 (中优先级)

8. **DevOps 能力** 🟡 (P2)
    - ❌ 缺少 CI/CD 配置
    - ❌ 缺少 Docker 镜像构建优化
    - ⚠️ 只有基本的 Dockerfile
    - ❌ 缺少自动化部署脚本

---

## 3. 发展路线图

### 3.1 短期目标 (1-3个月) - v0.1.0

**主题**: 基础能力建设 + 质量提升 + 核心业务模块

#### 阶段一：质量保障体系建设 (3周) 🔴 P0

**Week 1: 测试框架完善**

- [ ] 提升单元测试覆盖率到 80%
- [ ] 完善集成测试用例
- [ ] 引入 Mutation Testing (PIT)
- [ ] 配置 JaCoCo 覆盖率门禁

**Week 2: 代码质量工具集成**

- [ ] 集成 SonarQube 静态代码分析
- [ ] 集成 ArchUnit 架构约束检查
- [ ] 优化 Spotless 代码格式化规则
- [ ] 配置 OWASP Dependency Check

**Week 3: CI/CD 自动化**

- [ ] GitHub Actions CI 配置
- [ ] 自动化测试流水线
- [ ] 代码质量自动检查
- [ ] Docker 镜像自动构建

#### 阶段二：基础设施完善 (4周) 🔴 P0

**Week 4-5: 可观测性建设**

- [x] 完善 OpenTelemetry 链路追踪 (OTel 1.62.0 SPI 多导出器架构，全面重写，40 个测试)
- [x] 集成 Prometheus + Grafana 监控 (Micrometer + spring-boot-starter-actuator + micrometer-registry-prometheus)
- [x] 统一日志格式 (JSON) (logback-spring.xml，json profile 激活 JsonEncoder，MDC traceId/spanId 自动注入)
- [x] 实现应用健康检查端点 (/actuator/health, /actuator/prometheus, /actuator/metrics)

**Week 6: 配置管理模块**

- [x] 开发 Config 配置管理模块 (模块结构已创建)
- [ ] 系统参数管理
- [ ] 数据字典管理
- [ ] 配置热更新机制

**Week 7: 日志中心模块**

- [x] 开发 Log 日志中心模块 (模块结构已创建)
- [ ] 操作日志自动记录 (AOP)
- [ ] 审计日志记录
- [ ] 日志查询和导出

#### 阶段三：性能优化 (2周) 🟠 P1

**Week 8: 数据库性能优化**

- [ ] HikariCP 连接池调优
- [ ] 慢查询监控和优化
- [ ] 索引优化建议
- [ ] 批量操作优化

**Week 9: 缓存策略优化**

- [ ] 实现多级缓存 (Caffeine + Redis)
- [ ] 缓存一致性方案
- [ ] 缓存穿透/击穿防护
- [ ] 缓存预热机制

#### 阶段四：业务模块扩展 (3周) 🟠 P1

**Week 10: Notification 消息中心**

- [ ] 站内消息功能
- [ ] WebSocket 实时推送
- [ ] 消息订阅管理
- [ ] 公告管理

**Week 11-12: 安全加固**

- [ ] 实现 API 限流 (令牌桶算法)
- [ ] IP 黑白名单
- [ ] 防重放攻击
- [ ] 敏感数据脱敏
- [ ] SQL 注入防护

### 3.2 中期目标 (3-6个月) - v0.5.0

**主题**: 业务模块丰富 + 开发效率提升 + 平台化初探

#### 通用业务模块开发 (见第5章详细规划)

**核心业务模块** (8周):

- [ ] **CMS 内容管理模块** - 3周
    - 文章管理 (Markdown/富文本)
    - 分类标签体系
    - 媒体资源管理
    - 全文搜索 (Elasticsearch 可选)

- [ ] **Workflow 工作流模块** - 4周
    - 集成 Flowable/Camunda
    - 流程设计器
    - 任务管理
    - 流程监控

- [ ] **Report 报表中心** - 3周
    - 报表设计器
    - 图表报表 (ECharts)
    - Excel 导出
    - 定时报表

#### 开发者工具建设 (4周)

**Week 1-2: 代码生成器**

- [ ] CRUD 代码生成
- [ ] 基于模板的代码生成
- [ ] MyBatis-Flex 代码生成增强
- [ ] 前端代码生成 (Vue/React)

**Week 3: 开发者工具**

- [ ] SQL 审核工具
- [ ] API 测试工具
- [ ] 数据库文档生成
- [ ] ER 图生成

**Week 4: 文档完善**

- [ ] 完善组件使用文档
- [ ] 编写最佳实践指南
- [ ] 常见问题 FAQ
- [ ] 视频教程录制

#### 单体应用性能极致优化 (4周)

**Week 1: 数据库优化**

- [ ] 读写分离 (主从复制)
- [ ] 连接池深度优化
- [ ] 慢查询自动优化建议
- [ ] 定期索引优化

**Week 2: 缓存架构升级**

- [ ] 实现三级缓存架构
- [ ] 缓存预热自动化
- [ ] 热点数据自动识别
- [ ] 缓存监控大盘

**Week 3: 异步化改造**

- [ ] 全面启用虚拟线程
- [ ] 异步任务框架
- [ ] 事件驱动机制 (本地事件总线)
- [ ] 批量任务处理优化

**Week 4: 性能基准**

- [ ] JMH 性能基准测试
- [ ] 压力测试脚本
- [ ] 性能监控大盘
- [ ] 性能优化报告

### 3.3 长期目标 (6-12个月) - v1.0.0

**主题**: 平台化建设 + 生态成熟 + 商业化探索

#### 开发者平台建设 (8周)

**项目脚手架生成器**:

- [ ] 基于模板的项目初始化
- [ ] 可视化配置界面
- [ ] 组件选择和组合
- [ ] 一键生成项目结构

**在线开发工具**:

- [ ] 在线 SQL 编辑器
- [ ] API Mock 平台
- [ ] 数据库设计工具
- [ ] 接口文档管理

**开发者社区**:

- [ ] 技术博客平台
- [ ] 问答社区
- [ ] 案例分享
- [ ] 插件市场

#### 运维管理平台 (6周)

**监控大盘**:

- [ ] 应用性能监控
- [ ] 数据库监控
- [ ] 缓存监控
- [ ] 业务指标监控

**配置中心**:

- [ ] 统一配置管理界面
- [ ] 配置版本管理
- [ ] 配置灰度发布
- [ ] 配置审计

**日志平台**:

- [ ] 集中日志查询
- [ ] 日志分析
- [ ] 告警规则管理
- [ ] 日志导出

#### 低代码平台探索 (8周)

**可视化建模**:

- [ ] 数据模型设计器
- [ ] 页面设计器
- [ ] 流程设计器
- [ ] 规则引擎

**代码生成**:

- [ ] 前后端代码生成
- [ ] API 自动生成
- [ ] 文档自动生成
- [ ] 测试用例生成

#### 分布式能力预留 (可选，按需启用)

**微服务化准备** (仅保留扩展点):

- [ ] Dubbo RPC 深度集成文档
- [ ] 服务拆分最佳实践
- [ ] Nacos 集成方案 (文档)
- [ ] 服务网格方案调研

**消息驱动** (可选):

- [ ] RocketMQ 集成组件 (按需)
- [ ] 本地事件总线增强
- [ ] 事件溯源模式
- [ ] CQRS 架构模式

> **注意**: 分布式能力仅作为扩展选项保留，默认不启用，仅在业务确实需要时再考虑。

---

## 4. 后续优化方向

### 4.1 架构优化

#### 1. Gateway 增强 🚀

**当前问题**:

- 功能相对简单
- 不支持限流熔断
- 不支持动态路由

**优化方向**:

```
Phase 1: 核心功能增强
├─ 实现限流策略 (令牌桶/滑动窗口)
├─ 实现熔断降级 (Hystrix模式)
├─ 实现重试机制
└─ 实现动态路由配置

Phase 2: 高级功能
├─ 灰度发布支持
├─ AB测试路由
├─ 请求聚合 (BFF)
└─ 响应缓存

Phase 3: 可观测性
├─ 实时流量监控
├─ 错误率统计
├─ 响应时间分布
└─ 自定义告警
```

#### 2. 数据库层优化 💾

**读写分离**:

```java
// loadup-components-database 增强
@Configuration
public class DataSourceConfig {
    @Bean
    public DataSource routingDataSource() {
        // 动态路由数据源
        // 读操作 -> 从库
        // 写操作 -> 主库
    }
}
```

**分库分表**:

```
选择方案:
1. ShardingSphere (推荐)
2. MyBatis-Flex 原生支持
3. 手动分片

实现:
- 创建 loadup-components-sharding 组件
- 支持水平分表
- 支持垂直分库
- 透明化业务代码
```

**多租户支持**:

```
策略:
1. 独立数据库 (隔离性最好)
2. 共享数据库，独立Schema
3. 共享数据库，共享Schema，tenant_id区分

实现:
- TenantContext 租户上下文
- 自动添加 tenant_id 过滤
- 数据权限隔离
```

#### 3. 缓存策略优化 📦

**多级缓存**:

```
L1: Caffeine (本地缓存，ms级)
  ↓ Miss
L2: Redis (分布式缓存，10ms级)
  ↓ Miss
L3: Database (持久化，100ms级)

实现:
- 创建 MultiLevelCacheManager
- 缓存同步策略 (Canal/MQ)
- 缓存预热
- 缓存穿透/雪崩/击穿防护
```

**缓存一致性**:

```java
// 延迟双删
public void update(User user) {
    cache.delete(key);              // 删除缓存
    database.update(user);          // 更新数据库
    Thread.sleep(500);              // 延迟
    cache.delete(key);              // 再次删除
}

// 订阅 Binlog
canal.subscribe(event -> {
    if (event.isUpdate()) {
        cache.delete(buildKey(event));
    }
});
```

### 4.2 性能优化

#### 1. 异步化改造 ⚡

**虚拟线程应用**:

```yaml
spring:
  threads:
    virtual:
      enabled: true  # 启用虚拟线程

# 影响:
# - 每个请求一个虚拟线程
# - 阻塞操作不消耗系统线程
# - 吞吐量提升 10x+
```

**异步编程模式**:

```java
// CompletableFuture 异步链
CompletableFuture.supplyAsync(() -> queryUser())
    .thenApplyAsync(user -> enrichUserData(user))
    .thenAcceptAsync(user -> sendNotification(user))
    .exceptionally(ex -> handleError(ex));

// @Async 注解
@Async
public CompletableFuture<User> getUserAsync(String id) {
    return CompletableFuture.completedFuture(userService.getById(id));
}
```

#### 2. 批量操作优化 📊

**批量查询**:

```java
// 避免 N+1 问题
// Bad
for (Order order : orders) {
    User user = userService.getById(order.getUserId()); // N次查询
}

// Good
Set<String> userIds = orders.stream()
    .map(Order::getUserId)
    .collect(Collectors.toSet());
List<User> users = userService.listByIds(userIds);  // 1次查询
Map<String, User> userMap = users.stream()
    .collect(Collectors.toMap(User::getId, u -> u));
```

**批量写入**:

```java
// MyBatis-Flex 批量插入
userMapper.insertBatch(users, 1000);  // 每批1000条

// 优化: 使用 JDBC Batch
jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
    @Override
    public void setValues(PreparedStatement ps, int i) {
        // 设置参数
    }
    @Override
    public int getBatchSize() {
        return users.size();
    }
});
```

#### 3. SQL 优化 🔍

**慢查询监控**:

```yaml
# application.yml
spring:
  datasource:
    hikari:
      leak-detection-threshold: 60000  # 连接泄漏检测

mybatis-flex:
  configuration:
    log-impl: org.apache.ibatis.logging.slf4j.Slf4jImpl

# Logback 配置慢SQL
<logger name="io.github.loadup" level="DEBUG" />
```

**查询优化**:

```sql
-- 使用索引
CREATE INDEX idx_user_email ON user(email);
CREATE INDEX idx_user_status_created ON user(status, created_at);

-- 避免 SELECT *
SELECT id, username, email FROM user WHERE status = 1;

-- 分页优化 (深分页)
-- Bad: OFFSET 100000
SELECT * FROM user ORDER BY id LIMIT 100000, 20;

-- Good: 使用上次查询的最大ID
SELECT * FROM user WHERE id > 999980 ORDER BY id LIMIT 20;
```

### 4.3 代码质量提升

#### 1. 代码规范自动化 ✨

**增强 Spotless 配置**:

```xml
<plugin>
    <groupId>com.diffplug.spotless</groupId>
    <artifactId>spotless-maven-plugin</artifactId>
    <configuration>
        <java>
            <!-- Import 顺序 -->
            <importOrder>
                <order>java,javax,org,com,io,</order>
            </importOrder>
            <!-- 移除未使用导入 -->
            <removeUnusedImports/>
            <!-- 自定义格式化规则 -->
            <eclipse>
                <file>${project.basedir}/eclipse-formatter.xml</file>
            </eclipse>
        </java>
    </configuration>
</plugin>
```

**集成 ArchUnit**:

```java
// 架构测试
@AnalyzeClasses(packages = "io.github.loadup")
public class ArchitectureTest {
    
    @ArchTest
    static final ArchRule modules_should_not_depend_on_each_other =
        noClasses()
            .that().resideInAPackage("..modules.upms..")
            .should().dependOnClassesThat()
            .resideInAPackage("..modules.cms..");
    
    @ArchTest
    static final ArchRule controllers_should_be_in_adapter =
        classes()
            .that().haveSimpleNameEndingWith("Controller")
            .should().resideInAPackage("..adapter..");
}
```

#### 2. 单元测试增强 🧪

**测试覆盖率目标**:

```
- 核心业务逻辑: 90%+
- Service 层: 80%+
- Controller 层: 70%+
- Util 工具类: 95%+
```

**Mutation Testing (变异测试)**:

```xml
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <configuration>
        <targetClasses>
            <param>io.github.loadup.*</param>
        </targetClasses>
        <mutationThreshold>75</mutationThreshold>
    </configuration>
</plugin>
```

### 4.4 安全加固 🔒

#### 1. API 安全

**请求签名验证**:

```java
// 已有 signature 组件，增强使用
@PostMapping("/api/sensitive")
@RequireSignature(algorithm = "SHA256_WITH_RSA")
public Result operation(@RequestBody @Valid Request req) {
    // 业务逻辑
}
```

**防重放攻击**:

```java
@PostMapping("/api/payment")
@PreventReplay(window = 300)  // 5分钟窗口
public Result pay(@RequestBody PaymentRequest req) {
    // timestamp + nonce 验证
}
```

#### 2. 数据安全

**敏感数据加密**:

```java
// 字段级加密
@EncryptField(algorithm = "AES")
private String idCard;

@EncryptField(algorithm = "SM4")  // 国密
private String bankCard;
```

**数据脱敏**:

```java
@JsonSerialize(using = MaskSerializer.class)
@Mask(type = MaskType.PHONE)
private String phone;  // 138****8888

@Mask(type = MaskType.ID_CARD)
private String idCard;  // 110***********1234
```

---

## 5. 通用业务模块规划

### 5.1 优先级排序

| 优先级   | 模块                | 业务价值  | 技术复杂度 | 预计工期 |
|-------|-------------------|-------|-------|------|
| 🔴 P0 | Config 配置管理       | ⭐⭐⭐⭐⭐ | ⭐⭐    | 1周   |
| 🔴 P0 | Log 日志中心          | ⭐⭐⭐⭐⭐ | ⭐⭐    | 1周   |
| 🟠 P1 | Notification 消息中心 | ⭐⭐⭐⭐  | ⭐⭐⭐   | 2周   |
| 🟠 P1 | CMS 内容管理          | ⭐⭐⭐⭐  | ⭐⭐⭐   | 3周   |
| 🟡 P2 | Workflow 工作流      | ⭐⭐⭐⭐  | ⭐⭐⭐⭐⭐ | 4周   |
| 🟡 P2 | Report 报表中心       | ⭐⭐⭐   | ⭐⭐⭐⭐  | 3周   |
| 🟢 P3 | Search 搜索服务       | ⭐⭐⭐   | ⭐⭐⭐⭐  | 3周   |
| 🟢 P3 | Payment 支付中心      | ⭐⭐⭐   | ⭐⭐⭐⭐  | 4周   |
| 🟢 P3 | Order 订单中心        | ⭐⭐⭐   | ⭐⭐⭐⭐  | 3周   |

### 5.2 详细规划

#### 5.2.1 Config 配置管理模块 🔴

**功能范围**:

```
loadup-modules-config/
├─ 系统配置
│  ├─ 系统参数 (key-value)
│  ├─ 业务开关 (feature toggle)
│  └─ 运行时配置
├─ 数据字典
│  ├─ 字典类型管理
│  ├─ 字典项管理
│  └─ 级联字典支持
├─ 配置版本管理
│  ├─ 配置历史
│  ├─ 配置对比
│  └─ 配置回滚
└─ 配置分发
   ├─ 配置推送
   ├─ 配置订阅
   └─ 配置刷新
```

**核心实体**:

```java
// 系统配置
ConfigItem {
    String key;           // 配置键
    String value;         // 配置值
    String type;          // 数据类型 (STRING/INT/BOOL/JSON)
    String category;      // 分类
    String description;   // 说明
    Boolean editable;     // 是否可编辑
    Boolean encrypted;    // 是否加密
}

// 数据字典
DictType {
    String code;          // 字典编码
    String name;          // 字典名称
    Integer sort;         // 排序
    Boolean system;       // 是否系统字典
}

DictItem {
    String dictCode;      // 字典编码
    String label;         // 标签
    String value;         // 值
    Integer sort;         // 排序
    String parentValue;   // 父级值(级联)
}
```

**技术实现**:

```java
// 配置监听
@Component
public class ConfigChangeListener {
    @EventListener
    public void onConfigChange(ConfigChangeEvent event) {
        // 刷新本地缓存
        // 通知订阅者
    }
}

// 配置客户端
@Configuration
public class AppConfig {
    @ConfigValue("app.max-upload-size")
    private Long maxUploadSize;  // 自动刷新
}
```

#### 5.2.2 Log 日志中心模块 🔴

**功能范围**:

```
loadup-modules-log/
├─ 操作日志
│  ├─ 自动记录 (@OperationLog)
│  ├─ 手动记录
│  └─ 异步写入
├─ 访问日志
│  ├─ HTTP 请求日志
│  ├─ RPC 调用日志
│  └─ 性能统计
├─ 审计日志
│  ├─ 敏感操作记录
│  ├─ 数据变更记录
│  └─ 合规审计
└─ 日志查询
   ├─ 多条件检索
   ├─ 全文搜索 (ES)
   └─ 日志导出
```

**核心实体**:

```java
// 操作日志
OperationLog {
    String userId;        // 操作人
    String module;        // 模块
    String operation;     // 操作类型 (CREATE/UPDATE/DELETE)
    String description;   // 描述
    String method;        // 方法名
    String params;        // 请求参数
    String result;        // 返回结果
    Long duration;        // 耗时(ms)
    String ip;            // IP地址
    String userAgent;     // UA
    LocalDateTime time;   // 操作时间
}

// 审计日志
AuditLog {
    String userId;
    String dataType;      // 数据类型
    String dataId;        // 数据ID
    String action;        // 动作
    String before;        // 变更前(JSON)
    String after;         // 变更后(JSON)
    String reason;        // 变更原因
}
```

**技术实现**:

```java
// AOP 自动记录
@Aspect
public class OperationLogAspect {
    @Around("@annotation(OperationLog)")
    public Object around(ProceedingJoinPoint pjp, OperationLog log) {
        // 记录前置信息
        // 执行方法
        // 记录后置信息
        // 异步写入数据库
    }
}

// 审计拦截器
@Component
public class AuditInterceptor extends TableChangeInterceptor {
    @Override
    public void onUpdate(TableChangeEvent event) {
        // 记录数据变更
        auditLogService.save(buildAuditLog(event));
    }
}
```

#### 5.2.3 Notification 消息中心模块 🟠

**功能范围**:

```
loadup-modules-notification/
├─ 站内消息
│  ├─ 系统消息
│  ├─ 用户消息
│  └─ 消息分类
├─ 消息订阅
│  ├─ 订阅管理
│  ├─ 订阅规则
│  └─ 推送偏好
├─ 公告管理
│  ├─ 公告发布
│  ├─ 目标用户
│  └─ 公告有效期
└─ 消息推送
   ├─ 实时推送 (WebSocket)
   ├─ 批量推送
   └─ 定时推送
```

**核心实体**:

```java
// 消息
Message {
    String title;         // 标题
    String content;       // 内容
    String type;          // 类型 (SYSTEM/USER/NOTICE)
    String level;         // 级别 (INFO/WARNING/ERROR)
    String targetType;    // 目标类型 (USER/ROLE/DEPT/ALL)
    List<String> targets; // 目标列表
    LocalDateTime sentAt; // 发送时间
    LocalDateTime expireAt; // 过期时间
}

// 用户消息
UserMessage {
    String userId;
    String messageId;
    Boolean read;         // 是否已读
    LocalDateTime readAt; // 读取时间
}

// 消息订阅
MessageSubscription {
    String userId;
    String category;      // 订阅分类
    Boolean enabled;      // 是否启用
    Map<String, Object> config; // 推送配置
}
```

**技术实现**:

```java
// WebSocket 实时推送
@Component
public class MessagePushService {
    public void pushToUser(String userId, Message msg) {
        WebSocketSession session = sessionManager.getSession(userId);
        if (session != null && session.isOpen()) {
            session.sendMessage(new TextMessage(toJson(msg)));
        } else {
            // 存储离线消息
            messageRepository.save(msg);
        }
    }
}

// 批量推送
public void batchPush(Message msg, List<String> userIds) {
    // 使用虚拟线程并发推送
    userIds.parallelStream().forEach(userId -> {
        pushToUser(userId, msg);
    });
}
```

#### 5.2.4 CMS 内容管理模块 🟠

**功能范围**:

```
loadup-modules-cms/
├─ 文章管理
│  ├─ 文章编辑 (Markdown/富文本)
│  ├─ 文章分类
│  ├─ 标签管理
│  └─ 文章状态 (草稿/发布/下架)
├─ 栏目管理
│  ├─ 栏目树
│  ├─ 栏目模板
│  └─ 栏目权限
├─ 媒体管理
│  ├─ 图片上传
│  ├─ 视频上传
│  └─ 附件管理
└─ 内容审核
   ├─ 审核流程
   ├─ 敏感词过滤
   └─ 版权检测
```

**核心实体**:

```java
// 文章
Article {
    String title;         // 标题
    String summary;       // 摘要
    String content;       // 内容
    String contentType;   // 内容类型 (MARKDOWN/HTML)
    String categoryId;    // 分类ID
    List<String> tags;    // 标签
    String coverImage;    // 封面图
    String authorId;      // 作者ID
    Integer viewCount;    // 浏览量
    Integer likeCount;    // 点赞数
    String status;        // 状态 (DRAFT/PUBLISHED/ARCHIVED)
    LocalDateTime publishAt; // 发布时间
}

// 分类
Category {
    String name;
    String code;
    String parentId;      // 父分类
    Integer sort;
    String seoKeywords;   // SEO关键词
    String seoDescription;
}
```

**技术实现**:

```java
// 全文搜索 (集成 Elasticsearch)
@Service
public class ArticleSearchService {
    public Page<Article> search(String keyword, Pageable pageable) {
        return elasticsearchTemplate.search(
            Query.multiMatch(keyword, "title", "content", "tags"),
            Article.class,
            pageable
        );
    }
}

// 浏览量统计 (异步)
@Async
public void incrementViewCount(String articleId) {
    redisTemplate.opsForValue().increment("article:view:" + articleId);
    // 定时任务批量同步到数据库
}
```

#### 5.2.5 Workflow 工作流模块 🟡

**功能范围**:

```
loadup-modules-workflow/
├─ 流程定义
│  ├─ 流程设计器 (BPMN)
│  ├─ 流程模板
│  └─ 流程版本管理
├─ 流程实例
│  ├─ 流程发起
│  ├─ 任务处理
│  ├─ 流程流转
│  └─ 流程撤回
├─ 任务管理
│  ├─ 待办任务
│  ├─ 已办任务
│  ├─ 任务委托
│  └─ 任务转办
└─ 流程监控
   ├─ 流程跟踪
   ├─ 性能分析
   └─ 异常处理
```

**技术选型**:

```
方案一: Flowable (推荐)
- 成熟稳定
- BPMN 2.0标准
- 支持复杂流程

方案二: Camunda
- 轻量级
- 高性能
- 云原生

方案三: Activiti
- 老牌流程引擎
- 社区活跃
```

**核心实体**:

```java
// 流程定义
ProcessDefinition {
    String name;
    String key;
    String category;
    String bpmnXml;       // BPMN XML
    Integer version;
    Boolean enabled;
}

// 流程实例
ProcessInstance {
    String definitionId;
    String businessKey;   // 业务键
    String startUserId;
    String currentNode;
    String status;        // RUNNING/COMPLETED/SUSPENDED
    Map<String, Object> variables;
}

// 任务
Task {
    String instanceId;
    String nodeName;
    String assignee;      // 处理人
    LocalDateTime dueDate;
    String status;        // PENDING/COMPLETED
}
```

#### 5.2.6 其他规划模块

**Report 报表中心** 📊:

```
- 报表设计器
- SQL/存储过程报表
- 图表报表 (ECharts)
- Excel 导出
- 定时报表
- 报表订阅
```

**Search 搜索服务** 🔍:

```
- Elasticsearch 集成
- 全文检索
- 模糊搜索
- 聚合统计
- 搜索建议
- 热词统计
```

**Payment 支付中心** 💰:

```
- 支付宝/微信支付
- 银行卡支付
- 支付回调
- 退款处理
- 账单查询
- 对账功能
```

**Order 订单中心** 🛒:

```
- 订单创建
- 订单状态机
- 订单拆分
- 订单取消
- 售后处理
- 订单导出
```

---

## 6. 技术演进路线

### 6.1 架构演进（聚焦单体应用优化）

```
阶段一: 基础单体应用 (当前)
Application → Gateway → Modules (UPMS)
                     ↓
              MySQL + Redis

✅ 优点: 架构简单，易于开发和调试
⚠️ 不足: 性能有待优化，缺少监控

阶段二: 优化的单体应用 (3个月内)
Application → Gateway → Modules (UPMS + Config + Log + Notification)
                     ↓
        主从 MySQL (读写分离) + 多级缓存
                     ↓
          Prometheus + Grafana 监控

✅ 优点: 性能提升3-5倍，可观测性强
✅ 适用: 10万+ DAU，覆盖90%企业应用场景

阶段三: 极致性能单体 (6个月内)
Application → Gateway (限流熔断) → Modules (多个业务模块)
                     ↓
         MySQL 集群 (分库分表) + 三级缓存
                     ↓
    OpenTelemetry 全链路监控 + 自动告警

✅ 优点: 性能达到极致，支持100万+ DAU
✅ 适用: 99%企业应用场景

阶段四: 模块化单体 (可选，1年+)
Application → Gateway → 
              ├─ Module 1 (JAR)
              ├─ Module 2 (JAR)
              └─ Module 3 (JAR)
              
同一 JVM 进程内，模块间通过接口调用，保持单体架构优势

✅ 优点: 模块独立开发，保持单体部署简单性
✅ 适用: 大型团队协作

--------------------------------------------------
分布式架构 (可选扩展，仅在必要时考虑)

阶段五: 微服务架构 (按需，2年+)
仅在以下情况才考虑微服务拆分：
1. 单体应用性能瓶颈无法通过优化解决
2. 团队规模 > 50人
3. 业务复杂度确实需要隔离
4. 有专职运维团队支持

Application → Istio Ingress → Service Mesh
                           ↓
               Service1, Service2, Service3
                           ↓
              Nacos + K8s + Dubbo/gRPC

⚠️ 注意: 微服务带来复杂度，需谨慎评估
```

### 6.2 数据架构演进（渐进式优化）

```
阶段一: 单库单表 (当前)
MySQL 单实例
- 适用: 数据量 < 100万，并发 < 1000

✅ 优点: 简单可靠
⚠️ 瓶颈: 单点故障，性能受限

阶段二: 主从复制 + 读写分离 (1个月)
MySQL Master (写) → Slave1, Slave2 (读)
HikariCP 连接池 (写库/读库分离)

✅ 性能提升: 3-5倍
✅ 适用: 数据量 < 500万，并发 < 5000

阶段三: 垂直分库 (按需，3-6个月)
├─ 用户库 (user_db)
├─ 订单库 (order_db)
└─ 内容库 (cms_db)

✅ 适用: 单表数据 > 500万
✅ 好处: 资源隔离，故障隔离

阶段四: 水平分表 (按需，6-12个月)
user_table_00, user_table_01, ..., user_table_99
分片策略: 按 user_id % 100

✅ 适用: 单表数据 > 1000万
⚠️ 复杂度: 需要 ShardingSphere

--------------------------------------------------
多数据源架构 (可选扩展)

阶段五: 多类型数据库 (可选，1年+)
├─ MySQL (事务数据，OLTP)
├─ Redis (缓存 + Session)
├─ MongoDB (文档数据，日志)
├─ Elasticsearch (全文搜索)
└─ ClickHouse (OLAP分析，按需)

⚠️ 注意: 多数据源增加运维成本，按需引入
```

### 6.3 缓存架构演进

```
阶段一: Redis 单级缓存 (当前)
Application → Redis → MySQL

阶段二: 两级缓存 (1个月)
L1: Caffeine (本地缓存，ms级)
  ↓ Miss
L2: Redis (分布式缓存，10ms级)
  ↓ Miss
MySQL

✅ 性能提升: 5-10倍
✅ 缓存命中率: 95%+

阶段三: 三级缓存 (3个月)
L1: JVM Heap (热点数据，μs级)
  ↓ Miss
L2: Caffeine (本地缓存，ms级)
  ↓ Miss  
L3: Redis (分布式缓存，10ms级)
  ↓ Miss
MySQL

✅ 性能提升: 10-50倍
✅ 支持极高并发场景

阶段四: 智能缓存 (6个月)
- 自动识别热点数据
- 缓存预热自动化
- 缓存一致性保障
- 缓存穿透/击穿/雪崩防护
```

### 6.4 技术债务清理

| 问题       | 影响      | 优先级   | 解决方案                      | 计划时间        |
|----------|---------|-------|---------------------------|-------------|
| 测试覆盖率不足  | 质量风险高   | 🔴 P0 | 提升到80%，引入Mutation Testing | Q1 Week 1   |
| 缺少监控告警   | 运维风险高   | 🔴 P0 | 集成Prometheus+Grafana      | Q1 Week 4-5 |
| 静态代码分析缺失 | 代码质量无保障 | 🔴 P0 | 集成SonarQube               | Q1 Week 2   |
| 文档不完善    | 上手成本高   | 🟠 P1 | 补齐核心文档，录制视频教程             | Q2 Week 3   |
| 性能基准缺失   | 优化无依据   | 🟠 P1 | 建立性能基准测试                  | Q2 Week 4   |
| Dubbo 闲置 | 架构不清晰   | 🟢 P3 | 保留但不使用，文档说明               | Q3          |
| K8s 配置缺失 | 部署灵活性差  | 🟢 P3 | Docker优先，K8s可选            | Q4          |

**处理原则**:

- P0 (红色): 立即处理，阻塞发版
- P1 (橙色): 本季度内必须完成
- P2 (黄色): 半年内完成
- P3 (绿色): 一年内或按需处理

---

## 7. 行动计划（按优先级）

### 7.1 Q1 2026 (1-3月) - 基础打牢

**Week 1: 质量门禁建设** 🔴

- [x] 配置 JaCoCo 覆盖率检查
- [ ] 集成 Mutation Testing (PIT)
- [ ] 编写单元测试提升覆盖率到 60%
- [ ] GitHub Actions CI 基础配置

**Week 2: 代码质量工具** 🔴

- [ ] 集成 SonarQube
- [ ] 集成 ArchUnit 架构约束
- [ ] 优化 Spotless 规则
- [ ] OWASP Dependency Check

**Week 3: 自动化构建** 🔴

- [ ] 完善 GitHub Actions
- [ ] Docker 镜像构建优化
- [ ] 自动化测试流水线
- [ ] 发布流程自动化

**Week 4-5: 可观测性** 🔴

- [x] 完善链路追踪 (OTel 1.62.0 SPI 架构，40 个测试全通过)
- [x] Prometheus + Grafana 集成 (Micrometer + /actuator/prometheus)
- [x] 统一日志格式 (JSON) (logback-spring.xml + `json` profile)
- [x] 健康检查端点 (/actuator/health, show-details: when-authorized)

**Week 6: Config 模块** 🔴

- [x] 模块结构创建 (loadup-modules-config)
- [ ] 系统参数管理
- [ ] 数据字典管理
- [ ] 配置热更新
- [ ] 单元测试

**Week 7: Log 模块** 🔴

- [x] 模块结构创建 (loadup-modules-log)
- [ ] 操作日志 AOP
- [ ] 审计日志
- [ ] 日志查询
- [ ] 单元测试

**Week 8: 数据库优化** 🟠

- [ ] HikariCP 调优
- [ ] 慢查询监控
- [ ] 索引优化
- [ ] 批量操作优化

**Week 9: 缓存优化** 🟠

- [ ] 两级缓存实现
- [ ] 缓存一致性
- [ ] 缓存防护
- [ ] 性能测试

**Week 10: Notification 模块** 🟠

- [ ] 站内消息
- [ ] WebSocket 推送
- [ ] 消息订阅
- [ ] 单元测试

**Week 11-12: 安全加固** 🟠

- [ ] API 限流
- [ ] IP 黑白名单
- [ ] 防重放攻击
- [ ] 数据脱敏

**Week 12: 测试覆盖率冲刺** 🔴

- [ ] 单元测试覆盖率达到 80%
- [ ] 集成测试完善
- [ ] 性能基准测试
- [ ] 测试报告生成

### 7.2 Q2 2026 (4-6月) - 业务扩展

**业务模块**:

- [ ] CMS 内容管理 (3周)
- [ ] Workflow 工作流 (4周)
- [ ] Report 报表中心 (3周)

**开发者工具**:

- [ ] 代码生成器 (2周)
- [ ] SQL 审核工具 (1周)
- [ ] 文档生成 (1周)

**性能优化**:

- [ ] 读写分离 (1周)
- [ ] 三级缓存 (1周)
- [ ] 虚拟线程全面应用 (1周)
- [ ] 性能基准报告 (1周)

### 7.3 Q3-Q4 2026 (7-12月) - 平台化

**平台建设**:

- [ ] 开发者平台 (8周)
- [ ] 运维管理平台 (6周)
- [ ] 低代码平台 (8周)

**生态建设**:

- [ ] 插件市场
- [ ] 最佳实践库
- [ ] 开发者社区
- [ ] 视频教程

**可选扩展**:

- [ ] 微服务拆分方案（仅文档）
- [ ] 消息队列集成（按需）
- [ ] 分布式部署（按需）

---

## 8. 成功指标

### 8.1 质量指标（最高优先级）

| 指标              | 当前  | Q1目标 | Q2目标 | 年度目标 |
|-----------------|-----|------|------|------|
| **单元测试覆盖率**     | 40% | 80%  | 85%  | 90%  |
| **Mutation覆盖率** | 0%  | 60%  | 70%  | 75%  |
| **SonarQube评分** | -   | B    | A    | A+   |
| **技术债务**        | 未统计 | <8h  | <4h  | <2h  |
| **代码重复率**       | 未统计 | <5%  | <3%  | <2%  |

### 8.2 性能指标（高优先级）

| 指标              | 当前 | Q1目标     | Q2目标     | 年度目标      |
|-----------------|----|----------|----------|-----------|
| **API P95响应时间** | -  | <200ms   | <100ms   | <50ms     |
| **API P99响应时间** | -  | <500ms   | <300ms   | <100ms    |
| **数据库连接池利用率**   | -  | <70%     | <60%     | <50%      |
| **缓存命中率**       | -  | >90%     | >95%     | >98%      |
| **并发支持**        | -  | 1000 QPS | 5000 QPS | 10000 QPS |

### 8.3 稳定性指标（高优先级）

| 指标                | 当前 | Q1目标 | Q2目标   | 年度目标   |
|-------------------|----|------|--------|--------|
| **应用可用性**         | -  | 99%  | 99.9%  | 99.95% |
| **错误率**           | -  | <1%  | <0.5%  | <0.1%  |
| **MTTR (平均恢复时间)** | -  | <1h  | <30min | <15min |
| **监控覆盖率**         | 0% | 80%  | 95%    | 100%   |

### 8.4 开发效率指标（中优先级）

| 指标          | 当前  | Q1目标  | Q2目标  | 年度目标  |
|-------------|-----|-------|-------|-------|
| **构建时间**    | -   | <5min | <3min | <2min |
| **部署频率**    | -   | 周     | 日     | 日多次   |
| **代码生成覆盖率** | 0%  | 30%   | 50%   | 70%   |
| **文档完成度**   | 60% | 90%   | 95%   | 100%  |

### 8.5 业务指标（中优先级）

| 指标         | 当前 | Q1目标 | Q2目标 | 年度目标 |
|------------|----|------|------|------|
| **组件数量**   | 17 | 19   | 21   | 24   |
| **业务模块**   | 3  | 5    | 8    | 12   |
| **最佳实践案例** | 0  | 5    | 15   | 30   |
| **社区贡献者**  | 1  | 3    | 5    | 10   |

### 8.6 可选扩展指标（低优先级）

> 仅在启用分布式能力时关注

| 指标       | 说明           |
|----------|--------------|
| 服务数量     | 仅在微服务化时统计    |
| 服务间调用延迟  | 仅在启用RPC时统计   |
| 分布式事务成功率 | 仅在使用Seata时统计 |
| 服务注册成功率  | 仅在使用Nacos时统计 |

---

## 9. 架构决策说明

### 9.1 为什么选择"单体应用优先"策略？

#### 技术层面

**1. 单体应用的优势**

- ✅ **部署简单**: 一个 JAR 包搞定，无需复杂的服务编排
- ✅ **调试方便**: 本地 IDE 直接启动，断点调试无障碍
- ✅ **性能更好**: 进程内方法调用，无网络开销
- ✅ **事务简单**: 本地事务即可，无需分布式事务
- ✅ **运维成本低**: 一个进程监控，日志集中在一起

**2. 微服务的代价**

- ❌ **复杂度高**: 需要服务注册、配置中心、网关、链路追踪等一整套基础设施
- ❌ **运维成本高**: 多个服务部署、监控、日志收集
- ❌ **分布式事务难**: CAP 定理的取舍，最终一致性的复杂度
- ❌ **网络延迟**: 服务间调用增加网络延迟
- ❌ **调试困难**: 问题排查需要跨多个服务

#### 业务层面

**3. 实际业务需求分析**

```
99% 的企业应用：
- DAU < 100万
- 并发 < 10000 QPS
- 团队 < 50人
- 业务复杂度：中等

结论：单体应用完全够用！
```

**4. 过早微服务化的风险**

- 🔴 **过度设计**: 为了微服务而微服务，增加不必要的复杂度
- 🔴 **效率降低**: 开发效率、调试效率、部署效率都会下降
- 🔴 **团队负担**: 小团队难以驾驭微服务的复杂度
- 🔴 **资源浪费**: 多个服务占用更多内存和CPU

### 9.2 什么时候才需要微服务？

#### 真正需要微服务的场景

只有满足以下**至少3个条件**，才考虑微服务化：

1. **性能瓶颈** 🔴
    - 单体应用优化到极致仍无法满足性能要求
    - 例如：DAU > 100万，并发 > 10000 QPS

2. **团队规模** 🔴
    - 团队规模 > 50人
    - 多个团队并行开发同一个系统
    - 组织结构需要康威定律支持

3. **业务隔离** 🟠
    - 不同业务模块的发布周期差异很大
    - 某些模块需要独立扩缩容
    - 业务域边界清晰且稳定

4. **技术异构** 🟡
    - 不同模块需要不同的技术栈
    - 例如：核心业务用 Java，推荐系统用 Python

5. **运维团队** 🟡
    - 有专职的运维团队（至少5人）
    - 具备 K8s、监控、日志等全套基础设施运维能力

### 9.3 单体应用的性能极限

**优化后的单体应用可以支持**:

| 指标     | 优化前   | 优化后    | 说明             |
|--------|-------|--------|----------------|
| 并发 QPS | 1000  | 10000+ | 通过缓存、异步、批量优化   |
| 响应时间   | 500ms | 50ms   | 多级缓存 + 数据库优化   |
| DAU    | 10万   | 100万+  | 读写分离 + 缓存优化    |
| 数据量    | 100万  | 1亿+    | 分库分表 (仍然是单体部署) |

**真实案例**:

- **Stack Overflow**: 单体应用支持 1300万+ DAU
- **Shopify**: 早期单体应用支持数万商家
- **GitHub**: 2018年前一直是单体应用

### 9.4 我们的渐进式架构路径

```
第一阶段 (0-6个月):
├─ 完善单体应用
├─ 提升代码质量
├─ 建立监控体系
└─ 性能优化

第二阶段 (6-12个月):
├─ 模块化单体 (Modular Monolith)
├─ 模块边界清晰化
├─ 模块独立测试
└─ 为拆分做准备

第三阶段 (12-24个月):
├─ 评估是否真正需要微服务
├─ 如需要：渐进式拆分
│   ├─ 边缘服务先拆 (如报表)
│   ├─ 核心服务后拆 (如用户)
│   └─ 保持数据库单体 (延迟拆分)
└─ 如不需要：持续优化单体

原则：
✅ 能用单体解决的，绝不拆分
✅ 架构随业务演进，而非提前设计
✅ 保持简单，直到复杂度不可避免
```

### 9.5 保留分布式能力的理由

虽然当前优先单体，但保留分布式能力是为了：

1. **技术预研**: 了解分布式技术，为未来做准备
2. **可选扩展**: 某些大客户可能需要分布式部署
3. **人才储备**: 团队成员学习分布式架构
4. **灵活切换**: 业务突然爆发时，可以快速切换

**实现方式**:

- Dubbo: 集成但默认不启用，保留 RPC 调用能力
- Nacos: 提供集成文档，需要时一键启用
- MQ: 组件化设计，需要时引入依赖即可
- K8s: Dockerfile 和 Helm Charts 准备好，需要时部署

### 9.6 给团队的建议

**对开发者**:

- 先把单体应用代码写好，遵循 SOLID 原则
- 模块边界清晰，低耦合高内聚
- 充分的单元测试和集成测试
- 性能优化优先于架构拆分

**对架构师**:

- 不要为了炫技而过度设计
- 架构演进要跟随业务发展
- 保持 KISS 原则 (Keep It Simple, Stupid)
- 监控指标驱动架构决策

**对管理者**:

- 理解微服务不是银弹
- 单体应用不等于技术落后
- 性能问题优先考虑优化而非拆分
- 团队能力是架构选择的关键因素

---

## 附录

### A. 参考资源

- [Spring Boot 官方文档](https://spring.io/projects/spring-boot)
- [MyBatis-Flex 文档](https://mybatis-flex.com/)
- [Nacos 文档](https://nacos.io/)
- [RocketMQ 文档](https://rocketmq.apache.org/)
- [OpenTelemetry 文档](https://opentelemetry.io/)

### B. 相关文档

- [架构设计文档](./architecture.md)
- [组件文档](./components.md)
- [模块文档](./modules.md)
- [Gateway 文档](./gateway.md)

### C. 变更历史

| 版本    | 日期         | 变更内容         |
|-------|------------|--------------|
| 0.1.0 | 2026-02-28 | 初始版本，完整规划路线图 |

---

**© 2026 LoadUp Framework Team**

