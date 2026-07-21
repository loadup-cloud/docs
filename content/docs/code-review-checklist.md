# 代码审查检查单

> 适用于 PR 审查和 AI 辅助代码审查。逐节检查，发现问题在代码中标注行号和改进建议。

---

## 一、模块分层与依赖方向

### 依赖单向规则

- [ ] `commons` ← `components` ← `modules` ← `application`（严格单向，不得反向）
- [ ] `middleware/loadup-gateway` 只依赖 `commons` / `components`，未被 `modules` / `application` 依赖
- [ ] 无 modules 之间的横向依赖（`loadup-modules-xxx` 不引用 `loadup-modules-yyy`）
- [ ] 新增依赖已在 `loadup-dependencies/pom.xml` 的 `<dependencyManagement>` 中声明
- [ ] 子模块 `<parent>` 指向根 `loadup-parent`，不指向模块自身聚合 pom

### 层内职责

- [ ] `domain` 层：无 `@Table`、无 `@Service`、无任何 Spring / ORM 注解，只有 POJO + Gateway 接口 + 枚举
- [ ] `infrastructure` 层：DO 继承 `BaseDO`，不重复定义 `id` / `createdAt` / `updatedAt` / `tenantId` / `deleted`
- [ ] `infrastructure` 层：Mapper 只继承 `BaseMapper<XxxDO>`，不添加额外 SQL 方法
- [ ] `app` 层：Service 不直接操作 Mapper，通过 Gateway 抽象访问数据层
- [ ] `client` 层：无业务逻辑，无 DB 注解，只有 DTO / Command / Query

---

## 二、代码规范

### 注入方式

- [ ] 无 `@Autowired` 字段注入，全部使用构造器注入（`@RequiredArgsConstructor` + `final` 字段）

### 命名规范

- [ ] 数据库映射类命名为 `XxxDO`，继承 `BaseDO`
- [ ] 对外 DTO 命名为 `XxxDTO`
- [ ] 写操作入参命名为 `XxxCreateCommand` / `XxxUpdateCommand`
- [ ] 查询入参命名为 `XxxQuery`
- [ ] Gateway 接口命名为 `XxxGateway`，实现命名为 `XxxGatewayImpl`
- [ ] Service 直接加 `@Service`，不再分 interface + impl

### 文件头

- [ ] Java 文件头无 `/*- #%L ... #L% */` License 块（CI 自动插入，手动写会重复）

### API 暴露

- [ ] 无 `@RestController` / `@Controller`，API 路由在 `application.yml` 的 `loadup.gateway.routes` 中声明
- [ ] Gateway 路由 `target` 格式为 `bean://serviceName:method`

---

## 三、数据库与持久层

### 表结构

- [ ] 每张新表含全部 5 个标准字段：`id`、`tenant_id`、`created_at`、`updated_at`、`deleted`
- [ ] 主键类型为 `VARCHAR(64)` + 业务层 UUID 赋值，未使用 `BIGINT AUTO_INCREMENT`
- [ ] 无 `BOOLEAN` / `BOOL` 类型，逻辑列统一使用 `TINYINT`（0/1）
- [ ] Flyway 迁移脚本命名符合 `V{n}__{描述}.sql` 格式
- [ ] 测试 `schema.sql` 与生产 Flyway 脚本字段保持一致

### 查询安全

- [ ] 无字符串拼接 SQL，全部使用 MyBatis-Flex `QueryWrapper`
- [ ] 无 `SELECT *`，字段明确列出
- [ ] 批量写入使用 `insertBatch()` / `updateBatch()`，单批 ≤ 1000 条
- [ ] 无 N+1 查询，已使用 `in(ids)` 批量查询

### 对象转换

- [ ] DO ↔ domain model 转换使用 MapStruct，未使用手写 setter 链

---

## 四、安全

- [ ] 密码字段加 `@JsonIgnore`，DTO 层不返回
- [ ] 日志中无密码 / Token / 手机号 / 身份证明文
- [ ] 敏感信息（手机号、邮箱）已通过 commons 工具脱敏后输出
- [ ] 写操作已添加 `@RequirePermission("xxx:yyy")` 注解
- [ ] 无硬编码的密钥、AccessKey、密码（应从配置读取）

---

## 五、测试

- [ ] 集成测试类名以 `IT` 结尾（`XxxServiceIT.java`）
- [ ] 单元测试类名以 `Test` 结尾（`XxxServiceTest.java`）
- [ ] 集成测试使用 `@EnableTestContainers(ContainerType.MYSQL)`，未使用 `@MockBean` 替代 DB
- [ ] 测试方法命名遵循 `methodName_shouldResult_whenCondition` 格式
- [ ] 测试模块 `pom.xml` 的 `<parent>` 指向根 `loadup-parent`（不是模块自身 pom）
- [ ] `src/test/resources/` 下包含三个 yml 文件：`application.yml`、`application-test.yml`、`application-ci.yml`
- [ ] 核心 Service 覆盖率 ≥ 80%

---

## 六、性能

- [ ] 读多写少数据使用 Caffeine 本地缓存，写操作后主动 evict
- [ ] 分页查询单页上限 ≤ 200 条
- [ ] 大结果集使用分页，未一次性 `selectAll()`

---

## 七、pom.xml

- [ ] 同项目内模块互相引用时未写 `<version>`（由 BOM 统一管理）
- [ ] 三方依赖已在 `loadup-dependencies/pom.xml` 声明版本
- [ ] 新业务模块已在 `loadup-dependencies/pom.xml` 的 `<dependencyManagement>` 中添加条目
- [ ] `loadup-modules-xxx-test` 模块已设置 `<skip.deploy>true</skip.deploy>`（测试模块不发布）

---

## 快速判断：高风险改动

以下改动需要更严格的 review：

| 改动类型                    | 检查重点                                      |
|-------------------------|-------------------------------------------|
| 新增 Flyway 迁移脚本          | 版本号唯一、向后兼容、生产 schema 与测试 schema 同步        |
| 修改 GatewayImpl          | QueryWrapper 正确性、软删除过滤是否生效                |
| 新增 Gateway 路由           | securityCode 是否正确、路由不与现有路径冲突              |
| 引入新的三方依赖                | 是否已在 loadup-dependencies 声明、许可证兼容 GPL-3.0 |
| 修改 BaseDO 或公共 Converter | 影响所有模块，需全量测试                              |
| 修改 AutoConfiguration    | 组件扫描范围变化，检查 Bean 冲突                       |
