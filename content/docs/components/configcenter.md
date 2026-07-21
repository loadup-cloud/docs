<!-- last-sync: 2026-04-29 -->
---
id: components-configcenter
title: LoadUp ConfigCenter Component
---

# LoadUp ConfigCenter Component

`loadup-components-configcenter` 提供统一的配置中心抽象层，通过 **Binder/Binding 架构**
屏蔽底层配置源差异，支持本地文件（Local）、Nacos、Apollo 等多种实现的无缝切换，并预留 Spring Cloud Config / Consul / etcd 扩展点。

## 特性

- **统一接口**：通过 `ConfigCenterBinding` 提供 `getConfig`、`publishConfig`、`removeConfig`、`addListener`、`removeListener`
  等统一操作
- **灵活切换**：仅需修改 `loadup.configcenter.default-binder` 即可切换配置源，无需改动业务代码
- **实时监听**：`ConfigChangeListener` 回调，支持配置变更推送通知
- **Spring 自动刷新**：可选的 `@EnableConfigAutoRefresh` 注解，配置变更自动触发 Spring 上下文刷新
- **可扩展 SPI**：`ConfigCenterBinderPlugin` 标记接口，便于第三方接入 Spring Cloud Config、Consul、etcd 等
- **Spring Boot 3 自动配置**：开箱即用，零侵入

## 模块结构

```
loadup-components-configcenter/
├── loadup-components-configcenter-api/              # 核心接口、模型、自动配置
├── loadup-components-configcenter-binder-local/     # 本地文件 Binder（文件读写 + 轮询监听）
├── loadup-components-configcenter-binder-nacos/     # Nacos Binder（基于 nacos-client 2.4.3）
├── loadup-components-configcenter-binder-apollo/    # Apollo Binder（基于 apollo-client 2.3.0）
└── loadup-components-configcenter-test/             # 集成测试 + 单元测试
```

## 快速开始

### Maven 依赖

在 `loadup-dependencies` BOM 管理下，不需要写 `<version>`：

```xml
<!-- API 模块（必须引入） -->
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-configcenter-api</artifactId>
</dependency>

<!-- 选择一个 Binder 实现 -->

<!-- Local（本地文件，开发 / 无外部依赖场景） -->
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-configcenter-binder-local</artifactId>
</dependency>

<!-- Nacos -->
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-configcenter-binder-nacos</artifactId>
</dependency>

<!-- Apollo -->
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-configcenter-binder-apollo</artifactId>
</dependency>
```

### 配置（application.yml）

```yaml
loadup:
  configcenter:
    default-binder: local   # local | nacos | apollo

    binders:
      # ── Local ──────────────────────────────────────────
      local:
        base-path: config          # 配置文件根目录（相对于工作目录）
        refresh-interval: 30s      # 文件轮询间隔

      # ── Nacos ──────────────────────────────────────────
      nacos:
        server-addr: 127.0.0.1:8848
        namespace: ""              # 为空时使用 public namespace
        default-group: DEFAULT_GROUP
        username: nacos
        password: nacos
        timeout: 3000

      # ── Apollo ─────────────────────────────────────────
      apollo:
        meta: http://apollo-configservice:8080
        app-id: my-application
        env: DEV
        cluster: default
        apollo-namespace: application
        default-group: DEFAULT_GROUP
```

### 注入并使用

```java
@Service
@RequiredArgsConstructor
public class MyService {

    private final ConfigCenterBindingManager configCenter;

    public void demo() {
        // 获取绑定（使用 default binder）
        ConfigCenterBinding binding = configCenter.getBinding();

        // 读取配置
        Optional<String> value = binding.getConfig("feature.enabled");
        value.ifPresent(v -> log.info("feature.enabled = {}", v));

        // 写入配置（Local / Nacos 支持；Apollo 需 Open API）
        binding.publishConfig("feature.enabled", "true");

        // 监听变更
        binding.addListener("feature.enabled", event -> {
            log.info("Config changed: {} -> {}", event.getOldContent(), event.getNewContent());
        });
    }
}
```

## 配置中心 Binding 与 Binder 说明

### Binding（逻辑分组）

`ConfigCenterBinding` 代表一个逻辑配置分组（类比 Cache 中的"缓存空间"），通过
`ConfigCenterBindingManager.getBinding(bizTag)` 获取。

### Binder（底层实现）

`ConfigCenterBinder` 对接具体的配置中心客户端，核心方法：

| 方法                                                  | 说明      |
|-----------------------------------------------------|---------|
| `getConfig(dataId, group, settings)`                | 读取配置    |
| `publishConfig(dataId, group, content, settings)`   | 写入/更新配置 |
| `removeConfig(dataId, group, settings)`             | 删除配置    |
| `addListener(dataId, group, listener, settings)`    | 注册变更监听  |
| `removeListener(dataId, group, listener, settings)` | 注销变更监听  |

### securityCode 对应关系

`ConfigCenterBinding` 中的 `group` 参数优先级：  
**方法参数** → `bindingCfg.group` → `binderCfg.defaultGroup`

## 各 Binder 说明

### Local Binder

| 能力   | 支持情况                                        |
|------|---------------------------------------------|
| 读配置  | ✅ 先读文件，文件不存在则回退 Spring `Environment`        |
| 写配置  | ✅ 写入 `{basePath}/{group}/{dataId}` 文件       |
| 删配置  | ✅ 删除对应文件                                    |
| 监听变更 | ✅ 基于文件 `lastModified` 轮询（`refreshInterval`） |

本地文件存储路径规则：`{base-path}/{group}/{dataId}`

### Nacos Binder

| 能力   | 支持情况                                            |
|------|-------------------------------------------------|
| 读配置  | ✅ 通过 `nacos-client` `ConfigService.getConfig()` |
| 写配置  | ✅ `ConfigService.publishConfig()`               |
| 删配置  | ✅ `ConfigService.removeConfig()`                |
| 监听变更 | ✅ `ConfigService.addListener()`，基于 Nacos 长轮询推送  |

依赖：`com.alibaba.nacos:nacos-client:2.4.3`

### Apollo Binder

| 能力   | 支持情况                                                 |
|------|------------------------------------------------------|
| 读配置  | ✅ 通过 `apollo-client` `Config.getProperty()`          |
| 写配置  | ⚠️ 未实现（需 Apollo Portal Open API，当前打印 WARN 并返回 false） |
| 删配置  | ⚠️ 未实现（同上）                                           |
| 监听变更 | ✅ `Config.addChangeListener()`，Apollo 实时推送           |

> **注意**：Apollo 的 namespace 与 `group` 参数映射：调用时传入的 `group` 会被视为 Apollo namespace 加载；默认使用
`binderCfg.apolloNamespace`（默认 `application`）。

依赖：`com.ctrip.framework.apollo:apollo-client:2.3.0`

## Spring 自动刷新

引入 `spring-cloud-context` 并启用注解后，任何通过 `addListener` 监听的配置变更将自动触发 `ContextRefresher.refresh()`，刷新
`@RefreshScope` Bean：

```java
@SpringBootApplication
@EnableConfigAutoRefresh   // 启用配置自动刷新
public class MyApplication { ... }
```

> `@EnableConfigAutoRefresh` 通过 `@ConditionalOnClass(ContextRefresher.class)` 条件激活，
> 未引入 `spring-cloud-context` 时自动跳过，不影响普通项目。

## 自定义 Binder（SPI 扩展）

实现 `ConfigCenterBinderPlugin` 标记接口，可对接 Spring Cloud Config、Consul、etcd 等：

```java
@Component
public class ConsulConfigCenterBinder
        extends AbstractConfigCenterBinder<ConsulBinderCfg, ConfigCenterBindingCfg>
        implements ConfigCenterBinderPlugin {

    @Override
    public String getBinderType() { return "consul"; }

    // 实现 getConfig / publishConfig / addListener ...
}
```

在 `loadup-dependencies` 注册版本后，配置 `loadup.configcenter.default-binder: consul` 即可生效。

## ConfigChangeEvent 字段

| 字段           | 类型                 | 说明                               |
|--------------|--------------------|----------------------------------|
| `dataId`     | `String`           | 配置项 key                          |
| `group`      | `String`           | 配置分组                             |
| `namespace`  | `String`           | 命名空间                             |
| `oldContent` | `String`           | 变更前内容（删除时为原内容，新增时为 null）         |
| `newContent` | `String`           | 变更后内容（删除时为 null）                 |
| `changeType` | `ConfigChangeType` | `ADDED` / `MODIFIED` / `DELETED` |

## 测试

单元测试（无 Spring 上下文）：

```bash
mvn test -pl components/loadup-components-configcenter/loadup-components-configcenter-test \
    -Dtest=DefaultConfigCenterBindingTest
```

集成测试（本地 Binder，Spring 上下文，无需外部依赖）：

```bash
mvn test -pl components/loadup-components-configcenter/loadup-components-configcenter-test \
    -Dtest=ConfigCenterLocalBindingIT
```

## 包结构

| 子模块           | 包路径                                                     | 说明                                                                          |
|---------------|---------------------------------------------------------|-----------------------------------------------------------------------------|
| api           | `io.github.loadup.components.configcenter.model`        | `ConfigEntry`、`ConfigChangeEvent`、`ConfigChangeType`、`ConfigChangeListener` |
| api           | `io.github.loadup.components.configcenter.cfg`          | `ConfigCenterBinderCfg`、`ConfigCenterBindingCfg`                            |
| api           | `io.github.loadup.components.configcenter.binder`       | `ConfigCenterBinder` 接口                                                     |
| api           | `io.github.loadup.components.configcenter.binding`      | `ConfigCenterBinding` 接口                                                    |
| api           | `io.github.loadup.components.configcenter.binding.impl` | `AbstractConfigCenterBinder`、`DefaultConfigCenterBinding`                   |
| api           | `io.github.loadup.components.configcenter.manager`      | `ConfigCenterBindingManager`                                                |
| api           | `io.github.loadup.components.configcenter.properties`   | `ConfigCenterGroupProperties`、`ConfigCenterBinderType`                      |
| api           | `io.github.loadup.components.configcenter.spi`          | `ConfigCenterBinderPlugin`（SPI 扩展标记接口）                                      |
| api           | `io.github.loadup.components.configcenter.annotation`   | `@EnableConfigAutoRefresh`                                                  |
| binder-local  | `io.github.loadup.components.configcenter.local`        | Local 文件 Binder                                                             |
| binder-nacos  | `io.github.loadup.components.configcenter.nacos`        | Nacos Binder                                                                |
| binder-apollo | `io.github.loadup.components.configcenter.apollo`       | Apollo Binder                                                               |
