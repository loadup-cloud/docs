# LoadUp 测试指南

本文档说明 LoadUp 项目的测试策略、分层测试方法、常用模式和执行命令。

> 测试框架依赖：JUnit 5 + `loadup-testify-spring-boot-starter` + `loadup-components-testcontainers`

---

## 一、测试分层策略

| 层次      | 类型                        | 启动容器     | 类名后缀             | 主要工具                     |
|---------|---------------------------|----------|------------------|--------------------------|
| 纯逻辑单元测试 | 无 DB 交互的业务逻辑              | 否        | `XxxServiceTest` | Mockito                  |
| 集成测试    | 真实数据库的 Service/Gateway 测试 | 是（MySQL） | `XxxServiceIT`   | Testcontainers + AssertJ |
| 架构约束测试  | 依赖方向、包隔离检查                | 否        | `ArchTest`       | ArchUnit                 |

> ⚠️ **禁止在集成测试中用 `@MockBean` 替代真实数据库**，必须使用 `@EnableTestContainers(ContainerType.MYSQL)`。

---

## 二、测试模块结构

每个业务模块的 `*-test` 子模块结构：

```
loadup-modules-{mod}-test/
├── pom.xml                               # parent → loadup-parent
└── src/
    └── test/
        ├── java/io/github/loadup/modules/{mod}/
        │   ├── {Mod}TestApplication.java  # @SpringBootApplication 启动类
        │   ├── {Entity}ServiceIT.java     # 集成测试
        │   └── {Entity}ServiceTest.java   # 单元测试（如有纯逻辑）
        └── resources/
            ├── application.yml            # 激活 test profile（固定内容）
            ├── application-test.yml       # 本地开发配置
            ├── application-ci.yml         # CI 环境配置
            └── schema.sql                 # 测试建表语句（与生产 schema 同步）
```

### 2.1 测试启动类

```java
// {Mod}TestApplication.java
@SpringBootApplication
public class ConfigTestApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigTestApplication.class, args);
    }
}
```

### 2.2 三个 yml 文件（必须同时存在）

```yaml
# application.yml（固定，不修改）
spring:
  profiles:
    active: test
```

```yaml
# application-test.yml（本地开发）
spring:
  application:
    name: loadup-config-test
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      minimum-idle: 2
      maximum-pool-size: 10
      connection-timeout: 30000
      connection-test-query: SELECT 1
  sql:
    init:
      mode: always
      schema-locations: classpath:schema.sql
      continue-on-error: false

mybatis-flex:
  configuration:
    map-underscore-to-camel-case: true
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
    cache-enabled: false
  global-config:
    print-sql: true

testcontainers:
  reuse:
    enable: true   # 本地重用容器，加快速度

logging:
  level:
    io.github.loadup: DEBUG
    org.springframework.jdbc: DEBUG
```

```yaml
# application-ci.yml（CI 环境）
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 5
      minimum-idle: 2
      connection-timeout: 30000
      connection-test-query: SELECT 1
  sql:
    init:
      mode: always
      schema-locations: classpath:schema.sql

mybatis-flex:
  configuration:
    map-underscore-to-camel-case: true
    cache-enabled: false
  global-config:
    print-sql: false

testcontainers:
  reuse:
    enable: false

logging:
  level:
    root: WARN
    io.github.loadup: INFO
```

---

## 三、集成测试写法

### 3.1 标准集成测试模板

```java
package io.github.loadup.modules.config;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import io.github.loadup.components.testcontainers.annotation.ContainerType;
import io.github.loadup.components.testcontainers.annotation.EnableTestContainers;
import io.github.loadup.modules.config.app.service.ConfigItemService;
import io.github.loadup.modules.config.client.command.ConfigItemCreateCommand;
import io.github.loadup.modules.config.client.dto.ConfigItemDTO;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;

@SpringBootTest(classes = ConfigTestApplication.class)
@EnableTestContainers(ContainerType.MYSQL)
class ConfigItemServiceIT {

    @Autowired
    private ConfigItemService configItemService;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @BeforeEach
    void setUp() {
        // 清理测试数据，保证测试独立性
        jdbcTemplate.execute("DELETE FROM config_item WHERE config_key LIKE 'test.%'");
    }

    @Test
    void create_shouldPersist_whenValidCommand() {
        // given
        ConfigItemCreateCommand cmd = new ConfigItemCreateCommand();
        cmd.setConfigKey("test.key.example");
        cmd.setConfigValue("hello");
        cmd.setValueType("STRING");
        cmd.setCategory("test");

        // when
        String id = configItemService.create(cmd);

        // then
        assertThat(id).isNotBlank();
        ConfigItemDTO dto = configItemService.getByKey("test.key.example");
        assertThat(dto).isNotNull();
        assertThat(dto.getConfigValue()).isEqualTo("hello");
    }

    @Test
    void create_shouldThrow_whenKeyAlreadyExists() {
        // given：先创建
        ConfigItemCreateCommand cmd = new ConfigItemCreateCommand();
        cmd.setConfigKey("test.key.duplicate");
        cmd.setConfigValue("v1");
        cmd.setCategory("test");
        configItemService.create(cmd);

        // when + then：重复创建应抛异常
        assertThatThrownBy(() -> configItemService.create(cmd))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

### 3.2 测试方法命名规范

格式：`方法名_期望结果_触发条件`

| 方法名                                       | 场景           |
|-------------------------------------------|--------------|
| `create_shouldPersist_whenValidCommand`   | 正常创建成功       |
| `create_shouldThrow_whenKeyAlreadyExists` | 业务约束：key 重复  |
| `update_shouldUpdateValue_whenExists`     | 更新成功         |
| `delete_shouldSoftDelete_whenExists`      | 软删除成功        |
| `getByKey_shouldReturnNull_whenNotFound`  | 查询不存在返回 null |
| `listAll_shouldReturnEmpty_whenNoData`    | 空列表场景        |

### 3.3 测试数据隔离

```java
@BeforeEach
void setUp() {
    // ✅ 方案一：DELETE 指定范围数据（推荐，速度快）
    jdbcTemplate.execute("DELETE FROM config_item WHERE config_key LIKE 'test.%'");

    // ✅ 方案二：用 @Transactional 事务回滚（简单场景）
    // 在类上加 @Transactional，每个 @Test 方法执行后自动回滚
    // 注意：如果 Service 内部也有事务，可能不适用
}
```

---

## 四、单元测试写法

适用于无数据库交互的纯业务逻辑验证：

```java
package io.github.loadup.modules.config;

import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.verify;

import io.github.loadup.modules.config.app.service.ConfigItemService;
import io.github.loadup.modules.config.client.command.ConfigItemCreateCommand;
import io.github.loadup.modules.config.domain.gateway.ConfigItemGateway;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class ConfigItemServiceTest {

    @Mock
    private ConfigItemGateway gateway;

    @InjectMocks
    private ConfigItemService service;

    @Test
    void create_shouldThrow_whenKeyAlreadyExists() {
        // given
        given(gateway.existsByKey(any())).willReturn(true);

        // when + then
        assertThatThrownBy(() -> service.create(new ConfigItemCreateCommand()))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("配置键已存在");
    }

    @Test
    void create_shouldCallGatewaySave_whenValid() {
        // given
        given(gateway.existsByKey(any())).willReturn(false);
        ConfigItemCreateCommand cmd = validCommand();

        // when
        service.create(cmd);

        // then
        verify(gateway).save(any());
    }

    private ConfigItemCreateCommand validCommand() {
        ConfigItemCreateCommand cmd = new ConfigItemCreateCommand();
        cmd.setConfigKey("test.key");
        cmd.setConfigValue("value");
        cmd.setCategory("test");
        return cmd;
    }
}
```

---

## 五、架构约束测试（ArchUnit）

在 `loadup-application` 中定义，CI 自动检测依赖方向违规：

```java
// loadup-application/src/test/java/.../ArchitectureTest.java
package io.github.loadup;

import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

@AnalyzeClasses(packages = "io.github.loadup")
class ArchitectureTest {

    // domain 层不得引用 Spring/ORM 注解
    @ArchTest
    static final ArchRule domainShouldBeFrameworkFree = noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("com.mybatisflex..", "org.springframework.stereotype..");

    // infrastructure 不得直接引用 app service
    @ArchTest
    static final ArchRule infraShouldNotDependOnApp = noClasses()
        .that().resideInAPackage("..infrastructure..")
        .should().dependOnClassesThat()
        .resideInAPackage("..app.service..");
}
```

执行：`mvn test -pl loadup-application -Dtest=ArchitectureTest`

---

## 六、测试执行命令

```bash
# 单模块所有测试（单元 + 集成）
mvn test -pl modules/loadup-modules-config/loadup-modules-config-test

# 仅运行集成测试（IT 结尾）
mvn verify -pl modules/loadup-modules-config/loadup-modules-config-test \
    -Dtest="*IT" -DfailIfNoTests=false

# 仅运行单元测试（跳过集成测试）
mvn test -pl modules/loadup-modules-config/loadup-modules-config-test \
    -Dtest="*Test" -DfailIfNoTests=false

# 运行指定测试方法
mvn test -pl modules/loadup-modules-config/loadup-modules-config-test \
    -Dtest="ConfigItemServiceIT#create_shouldPersist_whenValidCommand"

# CI 模式（使用 ci profile，容器不重用）
mvn verify -P github -Dspring.profiles.active=ci \
    -Dtestcontainers.reuse.enable=false

# 所有模块测试
mvn verify
```

---

## 七、测试 pom.xml 依赖模板

```xml
<!-- loadup-modules-{mod}-test/pom.xml -->
<parent>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-parent</artifactId>
    <version>0.0.2-SNAPSHOT</version>
    <relativePath>../../../pom.xml</relativePath>
</parent>

<dependencies>
    <!-- 被测模块 -->
    <dependency>
        <groupId>io.github.loadup-cloud</groupId>
        <artifactId>loadup-modules-{mod}-app</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- 测试框架 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.github.loadup-cloud</groupId>
        <artifactId>loadup-testify-spring-boot-starter</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- 集成测试（真实 MySQL 容器） -->
    <dependency>
        <groupId>io.github.loadup-cloud</groupId>
        <artifactId>loadup-components-testcontainers</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.github.loadup-cloud</groupId>
        <artifactId>loadup-components-database</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- 缓存实现（测试时使用本地缓存） -->
    <dependency>
        <groupId>io.github.loadup-cloud</groupId>
        <artifactId>loadup-components-cache-binder-caffeine</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

---

## 八、常见测试问题快速排查

| 问题                                          | 原因                       | 解决                                        |
|---------------------------------------------|--------------------------|-------------------------------------------|
| `NoSuchBeanDefinitionException: XxxService` | AutoConfiguration 未注册    | 检查 `META-INF/spring/*.imports` 文件         |
| `Table 'testdb.xxx' doesn't exist`          | schema.sql 未加载           | 检查 `application-test.yml` 的 `sql.init` 配置 |
| Docker 未启动报错                                | Testcontainers 需要 Docker | 启动 Docker Desktop                         |
| 测试间数据互相干扰                                   | `@BeforeEach` 未清理数据      | 在 `@BeforeEach` 中 DELETE 测试数据             |
| `@MockBean` 替代 DB 被 CI 拒绝                   | 违反测试规范                   | 改用 `@EnableTestContainers`                |
| 集成测试缓慢                                      | 容器每次重新启动                 | 本地开启 `testcontainers.reuse.enable: true`  |

> 更多 Testcontainers 问题详见 [troubleshooting-guide.md](troubleshooting-guide.md#3-testcontainers-连接失败)
