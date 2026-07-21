# LoadUp 故障排查手册

本手册覆盖 LoadUp 开发、测试、部署过程中最常见的故障场景，提供快速诊断思路和解决方案。

> 网关（Gateway）专题参见 [troubleshooting/gateway-faq.md](troubleshooting/gateway-faq.md)  
> 401 认证问题参见 [troubleshooting/FIX_401_ERROR.md](troubleshooting/FIX_401_ERROR.md)

---

## 目录

1. [编译失败](#1-编译失败)
2. [MapStruct / MyBatis-Flex APT 生成问题](#2-mapstruct--mybatis-flex-apt-生成问题)
3. [Testcontainers 连接失败](#3-testcontainers-连接失败)
4. [Flyway 迁移失败](#4-flyway-迁移失败)
5. [Bean 注册与冲突](#5-bean-注册与冲突)
6. [Gateway 路由不生效](#6-gateway-路由不生效)
7. [缓存问题](#7-缓存问题)
8. [多租户与软删除异常](#8-多租户与软删除异常)
9. [Spotless 格式化失败](#9-spotless-格式化失败)
10. [依赖冲突](#10-依赖冲突)

---

## 1. 编译失败

### 1.1 `cannot find symbol` — 找不到 Tables.XXX_DO

**症状：**

```
error: cannot find symbol
symbol:   variable Tables.CONFIG_ITEM_DO
```

**原因：** MyBatis-Flex APT 尚未执行生成 `Tables` 类，或生成目录不在编译路径上。

**解决：**

```bash
# 强制重新触发 APT 编译
mvn clean generate-sources -pl modules/loadup-modules-config/loadup-modules-config-infrastructure

# 检查生成结果
find modules/loadup-modules-config/loadup-modules-config-infrastructure/target \
    -name "Tables.java" -o -name "*TableDef.java"
```

IDEA 中需要：Mark Directory as → Generated Sources Root，路径为 `target/generated-sources/annotations`。

---

### 1.2 `cannot find symbol` — MapStruct 生成的 Converter 找不到

**症状：**

```
error: cannot find symbol
symbol:   class ConfigItemConverterImpl
```

**原因：** `mapstruct-processor` 未在编译期执行，或与 Lombok 处理器顺序冲突。

**解决：** 确认 `pom.xml` 中 `mapstruct-processor` 在 Lombok 之后声明：

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <scope>provided</scope>
</dependency>
```

然后执行：

```bash
mvn clean compile -pl modules/loadup-modules-config/loadup-modules-config-infrastructure
```

---

### 1.3 循环依赖编译报错

**症状：** Maven 报 `[ERROR] The projects in the reactor contain a cyclic reference`

**原因：** 某子模块的 `<parent>` 错误地指向了本模块的聚合 pom（如 `loadup-modules-config/pom.xml`），而不是根
`loadup-parent`。

**解决：** 检查所有子模块的 `<parent>` 配置：

```bash
# 找出所有非 loadup-parent 作为 parent 的子模块
grep -r "artifactId>" modules/*/loadup-modules-* \
    --include="pom.xml" -l | xargs grep -l "<parent>"
```

子模块（client/domain/infrastructure/app/test）的 `<parent>` 必须是：

```xml
<parent>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-parent</artifactId>
    <version>0.0.2-SNAPSHOT</version>
    <relativePath>../../../pom.xml</relativePath>
</parent>
```

---

### 1.4 `package does not exist` — 同项目模块未找到

**症状：**

```
error: package io.github.loadup.modules.config.domain does not exist
```

**原因：** `loadup-modules-config-domain` 尚未进入本地 Maven 仓库，需要先 install。

**解决：**

```bash
# 安装依赖模块（含所有上游）
mvn clean install -DskipTests -pl modules/loadup-modules-config/loadup-modules-config-domain -am
```

---

## 2. MapStruct / MyBatis-Flex APT 生成问题

### 2.1 Tables 类生成为空（无字段）

**症状：** `Tables.java` 存在但内容为空，或缺少 `CONFIG_ITEM_DO` 等常量。

**检查项：**

1. `mybatis-flex.config` 文件是否在项目根目录（`loadup-parent/mybatis-flex.config`）
2. 内容是否包含 `processor.tables-generate-enable=true`

```properties
# loadup-parent/mybatis-flex.config（正确内容）
processor.tables-generate-enable=true
processor.entity-generate-enable=false
processor.allInTables.enable=true
processor.tables-class-name=Tables
processor.mapper.generateEnable=true
processor.mapper.annotation=true
```

> ⚠️ 不要在各子模块重复放置 `mybatis-flex.config`，APT 会自动冒泡到根目录查找。  
> ⚠️ 不要通过 Maven `<compilerArg>-Aprocessor.xxx</compilerArg>` 传入，含连字符的 key 会导致编译报错。

---

### 2.2 MapStruct 转换字段缺失

**症状：** `Converter.toModel()` 的某些字段没有映射，值为 `null`。

**检查项：**

- 字段名是否在 DO 和 domain model 中一致（MapStruct 按名称自动映射）
- 如字段名不一致，需加 `@Mapping(source = "doField", target = "modelField")`
- `BaseDO` 中的字段（`id`/`createdAt` 等）是否被 `@EqualsAndHashCode(callSuper = true)` 继承

```java
// 正确示例：继承 BaseDO 的字段会被 MapStruct 自动映射
@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
public interface ConfigItemConverter {
    ConfigItem toModel(ConfigItemDO entity);
    ConfigItemDO toEntity(ConfigItem model);
}
```

---

## 3. Testcontainers 连接失败

### 3.1 Docker 未启动

**症状：**

```
Could not find a valid Docker environment
```

**解决：**

```bash
# macOS
open -a Docker
# 或
docker ps   # 验证 Docker 可用
```

---

### 3.2 容器启动超时

**症状：**

```
org.testcontainers.containers.ContainerLaunchException: Container startup failed
```

**检查项：**

```bash
# 查看容器日志
docker ps -a
docker logs <container-id>

# 清理僵尸容器
docker container prune
```

本地开发加速：启用容器重用（已在 `application-test.yml` 中配置）：

```yaml
testcontainers:
  reuse:
    enable: true
```

---

### 3.3 schema.sql 未加载 / 表不存在

**症状：**

```
SQLSyntaxErrorException: Table 'testdb.config_item' doesn't exist
```

**检查项：**

1. `application-test.yml` 中是否配置了 `spring.sql.init.schema-locations`：

```yaml
spring:
  sql:
    init:
      mode: always
      schema-locations: classpath:schema.sql
      continue-on-error: false
```

2. `src/test/resources/schema.sql` 是否存在且与 `modules/loadup-modules-xxx/schema.sql` 内容一致
3. `schema.sql` 中是否使用了 `CREATE TABLE IF NOT EXISTS`（避免已存在时报错）

---

### 3.4 `@EnableTestContainers` 找不到

**症状：**

```
java.lang.annotation.AnnotationFormatError: Invalid default: @EnableTestContainers
```

**原因：** `loadup-components-testcontainers` 未在测试模块中声明。

**解决：** 在 `*-test/pom.xml` 中添加：

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-testify-spring-boot-starter</artifactId>
    <scope>test</scope>
</dependency>
```

---

## 4. Flyway 迁移失败

### 4.1 checksum 不匹配

**症状：**

```
FlywayException: Validate failed: Migration checksum mismatch for migration version 1
```

**原因：** 已应用的迁移脚本内容被修改。

**规则：** 已提交并应用的 Flyway 脚本**严禁修改**。需要变更则新建版本：

```bash
# 正确做法：新建 V2 脚本
touch modules/loadup-modules-config/loadup-modules-config-infrastructure/src/main/resources/db/migration/V2__add_column_xxx.sql
```

---

### 4.2 版本号冲突

**症状：**

```
FlywayException: Found more than one migration with version 1
```

**原因：** 同一模块或跨模块存在重复版本号。

**命名规范：**

- 格式：`V{n}__{描述}.sql`（双下划线）
- 例：`V1__init_config.sql`、`V2__add_config_item_tags.sql`
- 版本号在同一数据库实例内必须全局唯一（多模块共用一个 DB 时尤需注意）

---

### 4.3 本地测试 Flyway 与 schema.sql 冲突

**症状：** 集成测试启动时报 `Table already exists`

**原因：** `spring.sql.init` 和 Flyway 同时执行，重复建表。

**解决：** 测试环境只启用 `spring.sql.init`（用 schema.sql），禁用 Flyway：

```yaml
# application-test.yml
spring:
  flyway:
    enabled: false
  sql:
    init:
      mode: always
      schema-locations: classpath:schema.sql
```

生产/staging 环境只启用 Flyway，禁用 `spring.sql.init`。

---

## 5. Bean 注册与冲突

### 5.1 重复 Bean 定义

**症状：**

```
BeanDefinitionOverrideException: Invalid bean definition with name 'xxxService'
```

**原因：** 两个 `@ComponentScan` 扫描了相同包路径，或两个 `@Service` Bean 类名相同。

**检查项：**

```bash
# 查找相同类名
find modules -name "*Service.java" | xargs basename -a | sort | uniq -d
```

每个模块的 `AutoConfiguration` 中 `@ComponentScan` 应精确到本模块包：

```java
@ComponentScan(basePackages = "io.github.loadup.modules.config")
```

---

### 5.2 `@Autowired` 找不到 Bean

**症状：**

```
NoSuchBeanDefinitionException: No qualifying bean of type 'XxxGateway'
```

**原因：**

1. `AutoConfiguration` 未注册（`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 缺失）
2. `@MapperScan` 路径错误，Mapper Bean 未注册
3. `GatewayImpl` 缺少 `@Repository` 注解

**检查：**

```bash
# 查看 AutoConfiguration 注册文件
find modules -path "*/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports"

# 确认 GatewayImpl 注解
grep -r "@Repository" modules --include="*GatewayImpl.java"
```

---

### 5.3 Cache Bean 缺失（CacheService 未注入）

**原因：** 未引入任何 cache-binder（caffeine 或 redis），只有 `cache-api` 接口无法实例化。

**解决：** 在测试模块或生产应用中引入实现：

```xml
<!-- 本地缓存（测试 / 单体部署） -->
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-cache-binder-caffeine</artifactId>
</dependency>
```

---

## 6. Gateway 路由不生效

详见 [troubleshooting/gateway-faq.md](troubleshooting/gateway-faq.md)，以下是最常见的快速检查：

### 6.1 路由配置检查

```yaml
# application.yml 路由必须满足：
# 1. path 以 / 开头
# 2. target 格式正确：bean://beanName:methodName
# 3. Bean 名称默认为首字母小写的类名（configItemService）
loadup:
  gateway:
    routes:
      - path: /api/v1/config/list
        method: POST
        target: "bean://configItemService:listAll"
        securityCode: "default"
```

### 6.2 Bean 名称确认

```bash
# 查看 Spring Boot 启动日志，找注册的 Bean 名称
grep "configItemService\|Mapped.*ConfigItemService" logs/application.log
```

### 6.3 securityCode 说明

| securityCode | 含义             |
|--------------|----------------|
| `OFF`        | 关闭所有安全校验（公开接口） |
| `default`    | JWT Token 验证   |
| `hmac`       | HMAC 签名验签      |

---

## 7. 缓存问题

### 7.1 缓存未命中 / 每次都查 DB

**检查项：**

1. `@Cacheable` 是否在 Spring Bean 方法上（不能在 private 方法或 final 类上）
2. 同类内部调用不会触发 Spring AOP 缓存代理

**正确使用（在 `LocalCache` 类中）：**

```java
@Component
public class ConfigLocalCache {

    private final Cache<String, ConfigItem> cache = Caffeine.newBuilder()
        .expireAfterWrite(Duration.ofMinutes(10))
        .maximumSize(1000)
        .build();

    public ConfigItem getOrLoad(String key, Supplier<ConfigItem> loader) {
        return cache.get(key, k -> loader.get());
    }

    public void evict(String key) {
        cache.invalidate(key);
    }
}
```

### 7.2 写操作后缓存未失效

**规则：** 写操作（create/update/delete）完成后必须主动 `evict`：

```java
@Transactional(rollbackFor = Exception.class)
public void update(ConfigItemUpdateCommand cmd) {
    // ... 业务逻辑
    gateway.update(model);
    localCache.evict(cmd.getConfigKey());  // 必须清空对应缓存
}
```

---

## 8. 多租户与软删除异常

### 8.1 查询到其他租户数据

**原因：** `tenant_id` 过滤器未生效，或使用了 `mapper.selectAll()` 绕过了过滤器。

**检查：** `loadup-components-database` 中的 `TenantInterceptor` 是否被加载，`BaseDO` 上的 `@Table` 是否配置了多租户。

### 8.2 软删除数据仍被查询到

**症状：** `deleted=1` 的记录出现在查询结果中。

**原因：** 使用了原始 JDBC 或跨越了 MyBatis-Flex 的逻辑删除过滤器。

**正确用法：** 所有查询必须通过 `XxxDOMapper`（继承 `BaseMapper<XxxDO>`），不要直接执行 SQL。

### 8.3 软删除后唯一键冲突

**症状：** 删除后重新创建相同数据时，唯一键约束报错。

**原因：** `deleted=1` 的行仍占用唯一键索引。

**解决方案：** 唯一键设计时包含 `deleted` 字段（但会引入复杂性），或设计成逻辑唯一键用 `uuid` 区分，参见 schema 设计规范。

---

## 9. Spotless 格式化失败

### 9.1 格式检查失败（CI 报错）

**症状：**

```
[ERROR] The following files had format violations:
```

**解决：** 在推送前本地修复：

```bash
# 自动修复所有文件
mvn spotless:apply

# 单模块修复
mvn spotless:apply -pl modules/loadup-modules-config/loadup-modules-config-app
```

### 9.2 Spotless 与 IDEA 格式化冲突

Spotless 使用 **Palantir Java Format**，IDEA 默认格式化风格不同。

**解决：** 安装 IDEA 插件 [Palantir Java Format](https://plugins.jetbrains.com/plugin/13180-palantir-java-format)，或只依赖
`mvn spotless:apply` 修复，不使用 IDEA Format Code。

---

## 10. 依赖冲突

### 10.1 ClassNotFoundException / NoSuchMethodError

**症状：** 运行时找不到类或方法，但编译成功。

**原因：** Maven 依赖树中存在同一库的多个版本，错误版本被引入。

**诊断：**

```bash
# 查找冲突的依赖
mvn dependency:tree -Dverbose -pl loadup-application | grep "omitted\|conflict"

# 检查某个特定库的版本
mvn dependency:tree -pl loadup-application -Dincludes=com.fasterxml.jackson.core:jackson-databind
```

**解决：** 在 `loadup-dependencies/pom.xml` 的 `<dependencyManagement>` 中固定版本，不要在子模块中直接写 `<version>`。

---

### 10.2 子模块版本未受 BOM 管理

**症状：** 构建时警告 `Dependency 'xxx' is not managed`。

**解决：** 在 `loadup-dependencies/pom.xml` 中添加：

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-modules-xxx-client</artifactId>
    <version>${loadup.framework.version}</version>
</dependency>
```

子模块 `pom.xml` 中去掉 `<version>` 标签。

---

## 附：快速诊断决策树

```
构建/运行失败
├── 编译报错
│   ├── cannot find symbol → APT 未生成？ → 见 1.1, 1.2
│   ├── package does not exist → 上游模块未 install → 见 1.4
│   └── cyclic reference → parent 指向错误 → 见 1.3
├── 测试失败
│   ├── Docker 未启动 → 见 3.1
│   ├── 表不存在 → schema.sql 未加载 → 见 3.3
│   └── Bean 找不到 → AutoConfiguration 未注册 → 见 5.2
├── 运行时异常
│   ├── Gateway 路由 404 → 见 6.1
│   ├── 缓存未命中 → 见 7.1
│   └── ClassNotFoundException → 依赖冲突 → 见 10.1
└── CI 失败
    ├── Spotless → 见 9.1
    ├── Flyway checksum → 见 4.1
    └── ArchUnit → 横向依赖 → 见 module-dependency-map.md
```
