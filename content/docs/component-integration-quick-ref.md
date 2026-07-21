# 组件接入速查表

> 新建业务模块时，按需从下表复制依赖声明和配置片段。所有版本由 `loadup-dependencies` BOM 统一管理，`<version>` 留空。

---

## 1. 数据库（loadup-components-database）

**必选，所有模块均需接入。**

### pom.xml（infrastructure 子模块）

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-database</artifactId>
</dependency>
```

### application.yml

```yaml
loadup:
  database:
    id-generator:
      enabled: true
      strategy: random   # random | uuid-v4 | uuid-v7 | snowflake
      length: 20
    multi-tenant:
      enabled: false     # 多租户场景改为 true
      column-name: tenant_id
    logical-delete:
      enabled: true
      column-name: deleted
      deleted-value: "1"
      normal-value: "0"

spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/{dbname}?useUnicode=true&characterEncoding=utf8mb4&useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: your-password
    hikari:
      pool-name: LoadupPool
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000

mybatis-flex:
  configuration:
    map-underscore-to-camel-case: true
    cache-enabled: false
  global-config:
    print-sql: false   # 生产 false，测试 true
```

### 关键用法

```java
// DO 必须继承 BaseDO（id / tenantId / createdAt / updatedAt / deleted 已在其中）
@Data @NoArgsConstructor @AllArgsConstructor
@EqualsAndHashCode(callSuper = true)
@Table("your_table")
public class YourEntityDO extends BaseDO { ... }

// AutoConfiguration 中扫描 Mapper
@AutoConfiguration
@MapperScan("io.github.loadup.modules.{mod}.infrastructure.mapper")
@ComponentScan("io.github.loadup.modules.{mod}")
public class YourModuleAutoConfiguration {}
```

---

## 2. 本地缓存（loadup-components-cache-binder-caffeine）

### pom.xml（app 子模块）

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-cache-binder-caffeine</artifactId>
</dependency>
```

### application.yml

```yaml
loadup:
  cache:
    default-binder: caffeine
    enabled-binders: [caffeine]
    binders:
      caffeine:
        spec: initialCapacity=100,maximumSize=1000,expireAfterWrite=300s
```

---

## 3. 分布式缓存（loadup-components-cache-binder-redis）

### pom.xml（app 子模块）

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-cache-binder-redis</artifactId>
</dependency>
```

### application.yml

```yaml
loadup:
  cache:
    default-binder: redis
    enabled-binders: [redis]
    binders:
      redis:
        host: ${spring.data.redis.host:localhost}
        port: ${spring.data.redis.port:6379}
        password: ${spring.data.redis.password:}
        database: ${spring.data.redis.database:0}

spring:
  data:
    redis:
      host: localhost
      port: 6379
```

---

## 4. 集成测试容器（loadup-components-testcontainers）

**只放 test 子模块的 pom.xml。**

### pom.xml（test 子模块）

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-testify-spring-boot-starter</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
```

### 测试类注解

```java
@SpringBootTest
@EnableTestContainers(ContainerType.MYSQL)   // 自动启动 MySQL 容器
class YourServiceIT {
    @Autowired
    YourService service;
}
```

### 三个必需 yml（test/src/test/resources/）

```yaml
# application.yml（固定，不变）
spring:
  profiles:
    active: test
```

```yaml
# application-test.yml（本地开发）
testcontainers:
  reuse:
    enable: true
spring:
  sql:
    init:
      mode: always
      schema-locations: classpath:schema.sql
```

```yaml
# application-ci.yml（CI 环境）
testcontainers:
  reuse:
    enable: false
```

---

## 5. 数据库迁移（loadup-components-flyway）

### pom.xml（infrastructure 子模块）

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-flyway</artifactId>
</dependency>
```

### 迁移脚本命名

```
src/main/resources/db/migration/
├── V1__Create_initial_schema.sql
├── V2__Add_index_on_xxx.sql
└── V3__Add_column_xxx.sql
```

---

## 6. 方法级权限（loadup-components-authorization）

### pom.xml（app 子模块）

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-authorization</artifactId>
</dependency>
```

### 用法

```java
@RequirePermission("config:item:create")
public String create(ConfigItemCreateCommand cmd) { ... }
```

---

## 7. 分布式文件存储（loadup-components-dfs）

### pom.xml（app 子模块）

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-dfs-api</artifactId>
</dependency>
<!-- 选择一个 binder -->
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-dfs-binder-local</artifactId>
</dependency>
```

### application.yml

```yaml
loadup:
  dfs:
    default-provider: local
    max-file-size: 104857600   # 100MB
    providers:
      local:
        enabled: true
        base-path: /var/dfs/files
```

---

## 8. 统一消息通知（loadup-components-gotone）

### pom.xml（app 子模块）

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-gotone</artifactId>
</dependency>
```

### application.yml（最小配置，仅邮件）

```yaml
spring:
  mail:
    host: smtp.example.com
    port: 587
    username: noreply@example.com
    password: your-password

loadup:
  gotone:
    template:
      cache-enabled: true
      cache-ttl: 3600
```

---

## 9. 任务调度（loadup-components-scheduler）

### pom.xml（app 子模块）

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-scheduler</artifactId>
</dependency>
```

### application.yml

```yaml
# 开发/测试用
loadup:
  scheduler:
    type: simplejob

# 生产（Quartz + 持久化）
loadup:
  scheduler:
    type: quartz
spring:
  quartz:
    job-store-type: jdbc
```

---

## 10. 全局幂等（loadup-components-globalunique）

### pom.xml（infrastructure 子模块）

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-globalunique</artifactId>
</dependency>
```

> 依赖唯一约束表，幂等键方案参见 `components/loadup-components-globalunique/README.md`。

---

## 常见误区

| 错误                               | 正确                                           |
|----------------------------------|----------------------------------------------|
| `@Autowired` 注入                  | `@RequiredArgsConstructor` + `final` 字段      |
| 集成测试用 `@MockBean` 替代 DB          | `@EnableTestContainers(ContainerType.MYSQL)` |
| domain 层 DO 加 `@Table`           | DO 只放 `infrastructure.dataobject`            |
| 子模块写 `<version>`                 | 版本由 BOM 统一管理，不写                              |
| 同时引入 caffeine + redis binder 无配置 | `loadup.cache.default-binder` 指定默认 binder    |
