---
id: components-cache
title: LoadUp Cache Component
---

# LoadUp Cache Component

LoadUp Cache 组件提供了统一的缓存抽象层，支持多种缓存实现的无缝切换。通过简单的配置即可在不同的缓存实现（如 Redis、Caffeine
等）之间切换，无需修改任何业务代码。

## 目录

- [特性](#特性)
- [模块结构](#模块结构)
- [包结构说明](#包结构说明)
- [快速开始](#快速开始)
- [配置说明](#配置说明)
- [防缓存击穿和雪崩配置](#防缓存击穿和雪崩配置)
- [切换缓存实现](#切换缓存实现)
- [扩展自定义缓存实现](#扩展自定义缓存实现)
- [API 参考](#api-参考)
- [最佳实践](#最佳实践)
- [实现方案总结](#实现方案总结)
- [重构总结](#重构总结)
- [测试说明](#测试说明)
- [技术栈](#技术栈)
- [许可证](#许可证)

## 特性

- **统一接口**：通过 `CacheBinding` 接口提供统一的缓存操作方法（get/set/delete 等）
- **灵活切换**：通过配置文件轻松切换不同的缓存实现，无需修改代码
- **可扩展**：支持自定义缓存实现，只需实现 `CacheBinder` 接口
- **防雪崩**：支持随机过期偏移，避免大量缓存同时过期
- **防击穿**：支持热点数据长缓存、访问延期、高优先级保护
- **防穿透**：支持空值缓存，避免恶意查询
- **IDE 自动提示**：完善的配置元数据，支持 IDE 配置项自动补全和提示
- **Spring Boot 3 集成**：基于 Spring Boot 3 自动配置机制

## 模块结构

```
loadup-components-cache/
├── loadup-components-cache-api/          # 核心接口定义和配置
├── loadup-components-cache-binder-redis/ # Redis 缓存实现
├── loadup-components-cache-binder-caffeine/ # Caffeine 缓存实现
└── loadup-components-cache-test/         # 测试模块
```

## 包结构说明

### 详细包结构

```
loadup-components-cache/
├── loadup-components-cache-api/              # 核心 API 和配置模块
│   └── src/main/java/com/github/loadup/components/cache/
│       ├── api/                              # 核心接口
│       │   ├── CacheBinder.java             # 缓存绑定器接口
│       │   └── CacheBinding.java            # 缓存绑定接口
│       ├── binding/                          # 接口实现
│       │   └── DefaultCacheBinding.java     # 默认缓存绑定实现
│       ├── cfg/                              # 配置类
│       │   └── LoadUpCacheConfig.java       # 每个cache的配置
│       ├── config/                           # 自动配置
│       │   ├── CacheProperties.java         # 统一配置属性
│       │   └── CacheAutoConfiguration.java  # 自动配置类
│       ├── constans/                         # 常量
│       │   └── CacheConstants.java          # 缓存常量
│       └── util/                             # 工具类
│           └── CacheExpirationUtil.java     # 过期时间计算工具
│
├── loadup-components-cache-binder-caffeine/  # Caffeine 实现
│   └── src/main/java/com/github/loadup/components/cache/caffeine/
│       ├── CaffeineCacheAutoConfiguration.java    # Caffeine 自动配置
│       ├── LoadUpCaffeineCacheManager.java       # Caffeine 缓存管理器
│       └── binder/
│           └── CaffeineCacheBinderImpl.java      # Caffeine 绑定实现
│
├── loadup-components-cache-binder-redis/     # Redis 实现
│   └── src/main/java/com/github/loadup/components/cache/redis/
│       ├── RedisCacheAutoConfiguration.java       # Redis 自动配置
│       ├── LoadUpRedisCacheManager.java          # Redis 缓存管理器
│       └── impl/
│           └── RedisCacheBinderImpl.java         # Redis 绑定实现
│
└── loadup-components-cache-test/            # 测试模块
    └── src/test/java/com/github/loadup/components/cache/
        ├── caffeine/                         # Caffeine 测试
        ├── redis/                            # Redis 测试
        ├── common/                           # 通用测试
        └── integration/                      # 集成测试
```

### 包说明

#### api 模块

- **io.github.loadup.components.cache.binder**: 缓存绑定器接口，定义基本的 CRUD 操作
- **io.github.loadup.components.cache.binding**: CacheBinding 的默认实现，提供日志和性能监控
- **io.github.loadup.components.cache.cfg**: 单个缓存的配置类，包含防雪崩、防穿透等策略
- **io.github.loadup.components.cache.model**: 统一的缓存配置属性类和自动配置
- **io.github.loadup.components.cache.util**: 过期时间计算工具，支持随机偏移

#### binder-caffeine 模块

- **io.github.loadup.components.cache.caffeine**: Caffeine 自动配置和扩展的 Caffeine 缓存管理器
- **io.github.loadup.components.cache.caffeine.binder**: Caffeine 的 CacheBinder 实现

#### binder-redis 模块

- **io.github.loadup.components.cache.redis**: Redis 自动配置和扩展的 Redis 缓存管理器
- **io.github.loadup.components.cache.redis.impl**: Redis 的 CacheBinder 实现

### 依赖关系

```
api (核心接口和配置)
 ↑
 ├── binder-caffeine (依赖 api)
 └── binder-redis (依赖 api)
```

### 设计原则

1. **接口隔离**: API 模块只包含接口和配置，不包含具体实现
2. **依赖倒置**: Binder 模块依赖 API 接口，而不是具体实现
3. **单一职责**: 每个包负责特定的功能领域
4. **开闭原则**: 通过配置和接口扩展，无需修改核心代码

## 快速开始

### 1. 使用 Caffeine（本地内存缓存）

#### 添加依赖

```xml

<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-cache-core</artifactId>
    <version>${loadup.framework.version}</version>
</dependency>
<dependency>
<groupId>io.github.loadup-cloud</groupId>
<artifactId>loadup-components-cache-binder-caffeine</artifactId>
<version>${loadup.framework.version}</version>
</dependency>
```

#### 配置文件（application.yml）

```yaml
# 指定使用 Caffeine
loadup:
  cache:
    type: caffeine

# 使用 Spring Boot 标准配置
spring:
  cache:
    caffeine:
      spec: initialCapacity=1000,maximumSize=5000,expireAfterWrite=20m
```

或使用 properties 格式：

```properties
# 指定缓存类型
loadup.cache.binder=caffeine
# 使用 Spring Boot 标准配置
spring.cache.caffeine.spec=initialCapacity=1000,maximumSize=5000,expireAfterWrite=20m
```

### 2. 使用 Redis（分布式缓存）

#### 添加依赖

```xml

<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-cache-core</artifactId>
    <version>${loadup.framework.version}</version>
</dependency>
<dependency>
<groupId>io.github.loadup-cloud</groupId>
<artifactId>loadup-components-cache-binder-redis</artifactId>
<version>${loadup.framework.version}</version>
</dependency>
```

#### 配置文件（application.yml）

```yaml
# 指定使用 Redis
loadup:
  cache:
    type: redis

# 使用 Spring Boot 标准配置
spring:
  redis:
    host: localhost
    port: 6379
    password: yourpassword  # 可选
    database: 0
```

或使用 properties 格式：

```properties
# 指定缓存类型
loadup.cache.binder=redis
# 使用 Spring Boot 标准配置
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.password=
spring.redis.database=0
```

### 3. 使用代码

```java
import io.github.loadup.components.cache.binding.CacheBinding;
import org.springframework.stereotype.Service;
import jakarta.annotation.Resource;

@Service
public class UserService {

    @Resource
    private CacheBinding cacheBinding;

    public void saveUser(String userId, User user) {
        // 保存到缓存
        cacheBinding.set("users", userId, user);
    }

    public User getUser(String userId) {
        // 从缓存获取
        return cacheBinding.get("users", userId, User.class);
    }

    public void deleteUser(String userId) {
        // 删除缓存
        cacheBinding.delete("users", userId);
    }

    public void clearAllUsers() {
        // 清空缓存
        cacheBinding.deleteAll("users");
    }
}
```

## 配置说明

### LoadUp Cache 配置

LoadUp Cache 组件专注于缓存策略配置（如防雪崩、防击穿等），基础连接配置使用 Spring Boot 标准属性。

#### 通用配置

| 配置项                 | 类型     | 默认值      | 说明                          |
|---------------------|--------|----------|-----------------------------|
| `loadup.cache.type` | String | caffeine | 缓存类型，可选值：`redis`、`caffeine` |

#### Caffeine 基础配置

**使用 Spring Boot 标准配置：**

```yaml
spring:
  cache:
    caffeine:
      spec: initialCapacity=1000,maximumSize=5000,expireAfterWrite=20m,expireAfterAccess=10m
```

详见 [Spring Boot Caffeine 文档](https://docs.spring.io/spring-boot/docs/current/reference/html/io.html#io.caching.provider.caffeine)

#### Redis 基础配置

**使用 Spring Boot 标准配置：**

```yaml
spring:
  redis:
    host: localhost
    port: 6379
    password: yourpassword
    database: 0
```

详见 [Spring Boot Redis 文档](https://docs.spring.io/spring-boot/docs/current/reference/html/data.html#data.nosql.redis)

### 按 Cache Name 配置（防击穿和雪崩）

支持为每个 cache name 单独配置策略，有效防止缓存击穿、缓存雪崩和缓存穿透：

| 配置项                             | 类型      | 默认值  | 说明                           |
|---------------------------------|---------|------|------------------------------|
| `expire-after-write`            | String  | -    | 写入后过期时间（如 "30m", "1h", "2d"） |
| `expire-after-access`           | String  | -    | 访问后过期时间                      |
| `maximum-size`                  | Long    | -    | 最大缓存条目数                      |
| `enable-random-expiration`      | Boolean | true | 启用随机过期偏移（防雪崩）                |
| `random-offset-seconds`         | Long    | 60   | 随机偏移范围（秒）                    |
| `cache-null-values`             | Boolean | true | 缓存空值（防穿透）                    |
| `null-value-expire-after-write` | String  | "5m" | 空值过期时间                       |
| `priority`                      | Integer | 5    | 缓存优先级 (1-10)，防止热点数据被淘汰       |

**配置示例**：

```yaml
loadup:
  cache:
    type: redis # 或 caffeine
    redis: # 或 caffeine
      cache-config:
        # 热点数据 - 防击穿
        hot-products:
          expire-after-write: 6h
          expire-after-access: 1h          # 访问后延期
          maximum-size: 1000
          enable-random-expiration: true
          random-offset-seconds: 3600      # 1小时随机偏移
          priority: 9                      # 高优先级，不易淘汰

        # 普通数据 - 防雪崩和穿透
        users:
          expire-after-write: 30m
          maximum-size: 10000
          enable-random-expiration: true
          random-offset-seconds: 300       # 5分钟随机偏移
          cache-null-values: true          # 缓存空值
          null-value-expire-after-write: 5m
          priority: 5
```

**防护机制说明**：

1. **防缓存雪崩 (Cache Avalanche)**：通过 `enable-random-expiration` 和 `random-offset-seconds`
   为每个缓存条目添加随机过期时间偏移，避免大量缓存同时过期
    - 实际过期时间 = 基础过期时间 + random(0, random-offset-seconds)
    - 即使初始化时间相同，过期时间也会分散开来
2. **防缓存击穿 (Cache Breakdown)**：通过长过期时间 + 访问延期 + 高优先级保护热点数据
    - `expire-after-access`: 每次访问后重置过期时间
    - `priority`: 高优先级数据不易被淘汰
    - 热点数据设置更长的基础过期时间
3. **防缓存穿透 (Cache Penetration)**：通过 `cache-null-values` 缓存空值，避免恶意查询
    - `null-value-expire-after-write`: 空值使用较短的过期时间
    - 防止不存在的数据每次都打到数据库

## 防缓存击穿和雪崩配置

### 核心防护策略

#### 1. 防缓存雪崩配置

**问题**: 大量缓存同时过期，导致大量请求直接打到数据库

**解决方案**: 为不同的缓存设置不同的过期时间，并启用随机过期偏移

```yaml
loadup:
  cache:
    type: redis  # 或 caffeine
    redis: # 或 caffeine
      cache-config:
        # 用户缓存 - 30分钟 + 随机偏移
        users:
          expire-after-write: 30m
          maximum-size: 10000
          enable-random-expiration: true      # 启用随机过期
          random-offset-seconds: 300          # 随机偏移 0-300秒 (5分钟)

        # 商品缓存 - 1小时 + 随机偏移
        products:
          expire-after-write: 1h
          maximum-size: 5000
          enable-random-expiration: true
          random-offset-seconds: 600          # 随机偏移 0-600秒 (10分钟)

        # 订单缓存 - 2小时 + 随机偏移
        orders:
          expire-after-write: 2h
          maximum-size: 20000
          enable-random-expiration: true
          random-offset-seconds: 1200         # 随机偏移 0-1200秒 (20分钟)
```

**效果**:

- 用户缓存实际过期时间: 30分钟 + (0~5分钟随机)
- 商品缓存实际过期时间: 1小时 + (0~10分钟随机)
- 订单缓存实际过期时间: 2小时 + (0~20分钟随机)

#### 2. 防缓存击穿配置

**问题**: 热点数据过期瞬间，大量请求同时访问数据库

**解决方案**:

1. 热点数据设置更长的过期时间
2. 启用访问后过期策略
3. 提高热点数据的优先级

```yaml
loadup:
  cache:
    redis:
      cache-config:
        # 热点数据缓存
        hot-products:
          expire-after-write: 6h              # 基础过期时间更长
          expire-after-access: 1h             # 访问后1小时内不过期
          maximum-size: 1000
          enable-random-expiration: true
          random-offset-seconds: 3600         # 1小时随机偏移
          priority: 9                         # 高优先级，不易被淘汰

        # 普通数据缓存
        normal-data:
          expire-after-write: 30m
          maximum-size: 10000
          enable-random-expiration: true
          random-offset-seconds: 300
          priority: 5                         # 普通优先级
```

#### 3. 防缓存穿透配置

**问题**: 查询不存在的数据，缓存和数据库都没有，每次都打到数据库

**解决方案**: 缓存空值，但设置较短的过期时间

```yaml
loadup:
  cache:
    redis:
      cache-config:
        users:
          expire-after-write: 30m
          cache-null-values: true             # 启用空值缓存
          null-value-expire-after-write: 5m   # 空值缓存5分钟
          maximum-size: 10000
```

### 完整配置示例

#### Redis 完整配置

```yaml
# application.yml
loadup:
  cache:
    type: redis
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD:}
      database: 0
      cache-config:
        # ==================== 热点数据 ====================
        # 热门商品 - 长缓存 + 大随机偏移
        hot-products:
          expire-after-write: 6h
          expire-after-access: 1h
          maximum-size: 1000
          enable-random-expiration: true
          random-offset-seconds: 3600        # 1小时
          cache-null-values: true
          null-value-expire-after-write: 10m
          priority: 9

        # 热门用户 - 长缓存 + 访问延期
        hot-users:
          expire-after-write: 4h
          expire-after-access: 30m
          maximum-size: 5000
          enable-random-expiration: true
          random-offset-seconds: 1800        # 30分钟
          priority: 8

        # ==================== 普通数据 ====================
        # 用户信息
        users:
          expire-after-write: 30m
          maximum-size: 10000
          enable-random-expiration: true
          random-offset-seconds: 300         # 5分钟
          cache-null-values: true
          null-value-expire-after-write: 5m
          priority: 5

        # 商品信息
        products:
          expire-after-write: 1h
          maximum-size: 5000
          enable-random-expiration: true
          random-offset-seconds: 600         # 10分钟
          cache-null-values: true
          null-value-expire-after-write: 5m
          priority: 5

        # 订单信息
        orders:
          expire-after-write: 2h
          maximum-size: 20000
          enable-random-expiration: true
          random-offset-seconds: 1200        # 20分钟
          cache-null-values: false           # 订单不缓存空值
          priority: 6

        # ==================== 冷数据 ====================
        # 历史数据 - 短缓存
        history-data:
          expire-after-write: 10m
          maximum-size: 50000
          enable-random-expiration: true
          random-offset-seconds: 120         # 2分钟
          cache-null-values: true
          null-value-expire-after-write: 2m
          priority: 3

        # 统计数据 - 短缓存 + 小随机
        statistics:
          expire-after-write: 5m
          maximum-size: 1000
          enable-random-expiration: true
          random-offset-seconds: 60          # 1分钟
          priority: 2

        # ==================== 会话数据 ====================
        # 用户会话
        sessions:
          expire-after-write: 24h
          expire-after-access: 2h
          maximum-size: 100000
          enable-random-expiration: true
          random-offset-seconds: 7200        # 2小时
          cache-null-values: false
          priority: 7
```

#### Caffeine 完整配置

```yaml
# application.yml
loadup:
  cache:
    type: caffeine
    caffeine:
      initial-capacity: 1000
      maximum-size: 10000
      expire-after-access-seconds: 600
      expire-after-write-seconds: 1800
      cache-config:
        # 热点数据
        hot-data:
          expire-after-write: 6h
          expire-after-access: 1h
          maximum-size: 1000
          enable-random-expiration: true
          random-offset-seconds: 3600
          priority: 9

        # 普通数据
        normal-data:
          expire-after-write: 30m
          maximum-size: 5000
          enable-random-expiration: true
          random-offset-seconds: 300
          cache-null-values: true
          null-value-expire-after-write: 5m
          priority: 5

        # 临时数据
        temp-data:
          expire-after-write: 5m
          maximum-size: 10000
          enable-random-expiration: true
          random-offset-seconds: 60
          priority: 2
```

### 防护策略最佳实践

#### 1. 根据数据特征分类配置

```
热点数据: 长缓存 + 大随机偏移 + 高优先级
普通数据: 中等缓存 + 中等随机偏移 + 普通优先级
冷数据:   短缓存 + 小随机偏移 + 低优先级
临时数据: 极短缓存 + 小随机偏移 + 最低优先级
```

#### 2. 随机偏移时间建议

```
基础过期时间 < 10分钟:  随机偏移 10-20%
基础过期时间 10-60分钟: 随机偏移 15-25%
基础过期时间 > 1小时:   随机偏移 20-30%
```

#### 3. 空值缓存策略

- 对于经常查询不存在的数据: `cache-null-values: true`
- 空值缓存时间应远小于正常值: `null-value-expire-after-write: 5m`
- 订单等业务敏感数据: `cache-null-values: false`

#### 4. 分环境配置

开发环境:

```yaml
# application-dev.yml
loadup:
  cache:
    type: caffeine
    caffeine:
      expire-after-write-seconds: 300  # 5分钟
```

生产环境:

```yaml
# application-prod.yml
loadup:
  cache:
    type: redis
    redis:
      host: redis-cluster.prod
```

### 示例代码

```java

@Service
public class ProductService {

    @Resource
    private CacheBinding cacheBinding;

    public Product getProduct(String productId) {
        // 1. 尝试从缓存获取
        Product product = cacheBinding.get("products", productId, Product.class);

        if (product != null) {
            return product;
        }

        // 2. 缓存未命中，查询数据库
        product = productRepository.findById(productId);

        // 3. 存入缓存（包括空值，防止缓存穿透）
        if (product != null) {
            cacheBinding.set("products", productId, product);
        } else {
            // 缓存空值（配置中已设置较短过期时间）
            cacheBinding.set("products", productId, NULL_OBJECT);
        }

        return product;
    }

    public Product getHotProduct(String productId) {
        // 热点商品使用专门的 hot-products 缓存
        Product product = cacheBinding.get("hot-products", productId, Product.class);

        if (product != null) {
            return product;
        }

        product = productRepository.findById(productId);

        if (product != null) {
            cacheBinding.set("hot-products", productId, product);
        }

        return product;
    }
}
```

### 监控建议

建议监控以下指标:

- 缓存命中率 (Hit Rate)
- 缓存穿透次数
- 缓存雪崩告警
- 各 cache name 的访问频率
- 过期时间分布

### 故障排查

#### 问题: 仍然出现缓存雪崩

**检查项**:

1. 确认 `enable-random-expiration: true`
2. 增加 `random-offset-seconds` 值
3. 检查是否有批量初始化缓存的操作

#### 问题: 缓存穿透严重

**检查项**:

1. 确认 `cache-null-values: true`
2. 检查 `null-value-expire-after-write` 是否过短
3. 考虑使用布隆过滤器

#### 问题: 热点数据频繁失效

**检查项**:

1. 增加 `expire-after-write` 时间
2. 启用 `expire-after-access`
3. 提高 `priority` 值
4. 增加 `maximum-size`

## 切换缓存实现

从 Caffeine 切换到 Redis（或反向切换）非常简单：

1. **更新 POM 依赖**：将对应的 binder 依赖替换
2. **修改配置文件**：更新 `loadup.cache.type` 配置项
3. **无需修改代码**：业务代码保持不变

### 示例：从 Caffeine 切换到 Redis

#### 步骤 1：更新依赖

```xml
<!-- 移除或注释 Caffeine 依赖 -->
<!--
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-cache-binder-caffeine</artifactId>
    <version>${loadup.framework.version}</version>
</dependency>
-->

<!-- 添加 Redis 依赖 -->
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-cache-binder-redis</artifactId>
    <version>${loadup.framework.version}</version>
</dependency>
```

#### 步骤 2：更新配置

```yaml
loadup:
  cache:
    type: redis  # 从 caffeine 改为 redis
    redis:
      host: localhost
      port: 6379
```

#### 步骤 3：重启应用

业务代码无需任何修改，直接重启应用即可。

## 扩展自定义缓存实现

如果需要集成其他缓存实现（如 Memcached、Hazelcast 等），可以按以下步骤进行：

### 1. 创建新的 Binder 模块

```
loadup-components-cache-binder-{cache-name}/
├── pom.xml
└── src/main/java/
    └── com/github/loadup/components/cache/{cache-name}/
        ├── {CacheName}CacheAutoConfiguration.java
        └── impl/{CacheName}CacheBinderImpl.java
```

### 2. 实现 CacheBinder 接口

```java
public class CustomCacheBinderImpl implements CacheBinder {

    @Override
    public String getName() {
        return "CustomCache";
    }

    @Override
    public boolean set(String cacheName, String key, Object value) {
        // 实现 set 逻辑
        return true;
    }

    @Override
    public Object get(String cacheName, String key) {
        // 实现 get 逻辑
        return null;
    }

    @Override
    public <T> T get(String cacheName, String key, Class<T> cls) {
        // 实现 get with type 逻辑
        return null;
    }

    @Override
    public boolean delete(String cacheName, String key) {
        // 实现 delete 逻辑
        return true;
    }

    @Override
    public boolean deleteAll(String cacheName) {
        // 实现 deleteAll 逻辑
        return true;
    }
}
```

### 3. 创建自动配置类

```java

@Configuration
@ConditionalOnProperty(prefix = "loadup.cache", name = "type", havingValue = "custom")
public class CustomCacheAutoConfiguration {

    @Bean
    public CacheBinder cacheBinder() {
        return new CustomCacheBinderImpl();
    }
}
```

### 4. 注册自动配置

在 `src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件中添加：

```
io.github.loadup.components.cache.custom.CustomCacheAutoConfiguration
```

## API 参考

### CacheBinding 接口

```java
public interface CacheBinding {
    /**
     * 设置缓存
     * @param cacheName 缓存名称
     * @param key 缓存键
     * @param value 缓存值
     * @return 是否成功
     */
    boolean set(String cacheName, String key, Object value);

    /**
     * 获取缓存（返回 Object）
     * @param cacheName 缓存名称
     * @param key 缓存键
     * @return 缓存值
     */
    Object get(String cacheName, String key);

    /**
     * 获取缓存（返回指定类型）
     * @param cacheName 缓存名称
     * @param key 缓存键
     * @param cls 目标类型
     * @return 缓存值
     */
    <T> T get(String cacheName, String key, Class<T> cls);

    /**
     * 删除缓存
     * @param cacheName 缓存名称
     * @param key 缓存键
     * @return 是否成功
     */
    boolean delete(String cacheName, String key);

    /**
     * 清空指定缓存
     * @param cacheName 缓存名称
     * @return 是否成功
     */
    boolean deleteAll(String cacheName);
}
```

## 最佳实践

### 1. 开发环境使用 Caffeine

在开发环境中使用 Caffeine 可以减少外部依赖，提高开发效率：

```yaml
# application-dev.yml
loadup:
  cache:
    type: caffeine
    caffeine:
      maximum-size: 1000
      expire-after-write-seconds: 300
```

### 2. 生产环境使用 Redis

在生产环境中使用 Redis 以支持分布式缓存：

```yaml
# application-prod.yml
loadup:
  cache:
    type: redis
    redis:
      host: ${REDIS_HOST:redis.example.com}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD}
      database: 0
```

### 3. 合理设置过期时间

根据数据特性设置合适的过期时间，避免缓存雪崩：

```yaml
loadup:
  cache:
    caffeine:
      expire-after-write-seconds: 1800  # 30分钟
      expire-after-access-seconds: 900  # 15分钟
```

## 实现方案总结

### 实现的核心机制

#### 1. 配置化策略管理

为每个 cache name 单独配置缓存策略，支持以下配置：

- **基础配置**: 过期时间、最大容量
- **防雪崩配置**: 随机过期偏移
- **防穿透配置**: 空值缓存
- **优先级配置**: 缓存优先级和预热

#### 2. 核心防护机制

##### 防缓存雪崩 (Cache Avalanche)

- **问题**: 大量缓存同时过期，流量瞬间打到数据库
- **解决方案**:
    - 随机过期偏移：`enable-random-expiration: true`
    - 实际过期时间 = 基础过期时间 + random(0, random-offset-seconds)
    - 不同 cache name 设置不同的基础过期时间

##### 防缓存击穿 (Cache Breakdown)

- **问题**: 热点数据过期，瞬间大量请求访问数据库
- **解决方案**:
    - 热点数据设置更长的过期时间
    - 使用 `expire-after-access` 访问延期
    - 设置高优先级，不易被淘汰

##### 防缓存穿透 (Cache Penetration)

- **问题**: 查询不存在的数据，绕过缓存直接打数据库
- **解决方案**:
    - 缓存空值：`cache-null-values: true`
    - 空值设置较短过期时间

#### 3. 核心类说明

##### CacheExpirationUtil

过期时间计算工具类，提供以下功能：

- 计算带随机偏移的过期时间
- 计算按百分比的随机偏移
- 解析时间字符串 ("30m", "1h", "2d")

```java
// 计算带随机偏移的过期时间
Duration calculateExpirationWithRandomOffset(Duration base, LoadUpCacheConfig config)

// 计算按百分比的随机偏移
Duration calculateExpirationWithPercentageOffset(Duration base, double percentage)

// 解析时间字符串 ("30m", "1h", "2d")
Duration parseDuration(String durationStr)
```

##### LoadUpCacheConfig

增强的缓存配置类，支持：

- 基础过期配置（expireAfterWrite, expireAfterAccess）
- 随机过期配置（enableRandomExpiration, randomOffsetSeconds）
- 空值缓存配置（cacheNullValues, nullValueExpireAfterWrite）
- 优先级和预热配置

##### LoadUpCaffeineCacheManager / LoadUpRedisCacheManager

自定义缓存管理器，支持：

- 为每个 cache name 注册独立配置
- 动态创建缓存时应用对应策略

### 优势

1. **配置化管理**: 所有策略通过配置文件管理，无需修改代码
2. **灵活切换**: 不同环境使用不同缓存实现（开发用 Caffeine，生产用 Redis）
3. **自动防护**: 随机过期偏移自动生效，无需手动计算
4. **分级管理**: 热点数据、普通数据、临时数据使用不同策略
5. **易于扩展**: 新增缓存类型只需添加配置，无需修改代码

### 实现最佳实践

1. **基础过期时间**: 根据数据更新频率设置
    - 热点数据: 6-24 小时
    - 普通数据: 30分钟-2小时
    - 临时数据: 5-10分钟
2. **随机偏移比例**: 建议为基础时间的 10-30%
    - < 10分钟: 10-20%
    - 10-60分钟: 15-25%
    - >
      >
      > 1小时: 20-30%

3. **空值缓存**:
    - 高频查询: 启用空值缓存
    - 空值过期时间: 基础时间的 10-20%
    - 业务敏感数据: 禁用空值缓存
4. **分环境配置**:
    - 开发: Caffeine + 短过期
    - 测试: Redis + 中等过期
    - 生产: Redis + 完整策略

## 重构总结

### 重构历史 (2025-12-25)

#### 重构目标

1. 移除 `loadup-components-cache-core` 模块
2. 移除 `LoadUpRedisCacheProperties` 和 `LoadUpCaffeineCacheProperties`
3. 统一所有配置到 `CacheProperties` 类
4. 简化模块结构，所有核心类移到 `api` 模块

#### 重构内容

##### 删除的模块和类

- ✅ 删除 `loadup-components-cache-core` 整个模块
- ✅ 删除 `LoadUpRedisCacheProperties` 类
- ✅ 删除 `LoadUpCaffeineCacheProperties` 类
- ✅ 删除 Redis 和 Caffeine binder 的独立 metadata 文件

##### 新增/移动的类

**API 模块新增**:

- `io.github.loadup.components.cache.cfg.CacheBindingCfg` - 统一配置类
- `io.github.loadup.components.cache.model.CacheAutoConfiguration` - 自动配置
- `io.github.loadup.components.cache.binding.DefaultCacheBinding` - 默认实现

##### 统一的配置结构

**新的配置路径**:

```yaml
loadup:
  cache:
    type: redis  # 或 caffeine

    # Redis 配置
    redis:
      host: localhost
      port: 6379
      password:
      database: 0
      cache-config: # 每个cache name的配置
        users:
          expire-after-write: 30m
          maximum-size: 10000
```

**旧的配置路径（已废弃）**:

```yaml
# ❌ 旧路径 - 不再使用
spring:
  cache:
    redis:
      cache-config: ...
    caffeine:
      cache-config: ...
```

##### CacheProperties 类结构

```java

@ConfigurationProperties(prefix = "loadup.cache")
public class CacheProperties {
    private String         type     = "caffeine";
    private RedisConfig    redis    = new RedisConfig();
    private CaffeineConfig caffeine = new CaffeineConfig();

    public static class RedisConfig {
        private String                         host        = "localhost";
        private int                            port        = 6379;
        private String                         password;
        private int                            database    = 0;
        private Map<String, LoadUpCacheConfig> cacheConfig = new HashMap<>();
    }

    public static class CaffeineConfig {
        private int                            initialCapacity          = 1000;
        private long                           maximumSize              = 5000;
        private long                           expireAfterAccessSeconds = 600;
        private long                           expireAfterWriteSeconds  = 1200;
        private boolean                        allowNullValue           = false;
        private Map<String, LoadUpCacheConfig> cacheConfig              = new HashMap<>();
    }
}
```

#### 模块依赖关系

**重构前**:

```
loadup-components-cache (parent)
├── loadup-components-cache-api
├── loadup-components-cache-core (依赖 api)
├── loadup-components-cache-binder-caffeine (依赖 api + core)
├── loadup-components-cache-binder-redis (依赖 api + core)
└── loadup-components-cache-test
```

**重构后**:

```
loadup-components-cache (parent)
├── loadup-components-cache-api (包含所有核心配置)
├── loadup-components-cache-binder-caffeine (依赖 api)
├── loadup-components-cache-binder-redis (依赖 api)
└── loadup-components-cache-test
```

#### 优势

1. **更简单的模块结构** - 从 5 个模块减少到 4 个
2. **统一的配置** - 所有配置在一个地方：`loadup.cache`
3. **更清晰的依赖** - Binder 只依赖 API 模块
4. **更好的封装** - 配置类内聚在一起
5. **更容易维护** - 减少了模块间的依赖关系

#### 兼容性

**配置兼容性**:

- ⚠️ 配置路径已更改：从 `spring.cache.{redis|caffeine}.cache-config` 改为 `loadup.cache.{redis|caffeine}.cache-config`
- ✅ 功能完全兼容，只需更新配置文件

**代码兼容性**:

- ✅ 对外 API 完全兼容
- ✅ `CacheBinding` 接口未变
- ✅ 业务代码无需修改

#### 迁移指南

**更新配置文件**:

旧配置:

```yaml
loadup:
  cache:
    type: redis

spring:
  cache:
    redis:
      cache-config:
        users:
          expire-after-write: 30m
```

新配置:

```yaml
loadup:
  cache:
    type: redis
    redis:
      host: localhost
      port: 6379
      cache-config:
        users:
          expire-after-write: 30m
```

## 测试说明

### 测试结构

```
loadup-components-cache-test/
└── src/test/java/com/github/loadup/components/cache/
    ├── CacheTestApplication.java       # 测试启动类
    ├── common/                          # 通用测试基类
    │   ├── BaseCacheTest.java          # 测试基类
    │   ├── User.java                   # 测试用户实体
    │   └── Product.java                # 测试商品实体
    ├── caffeine/                        # Caffeine 测试
    │   ├── CaffeineBasicOperationsTest.java     # 基础操作测试
    │   ├── CaffeineExpirationTest.java          # 过期策略测试
    │   ├── CaffeineConcurrencyTest.java         # 并发测试
    │   └── AntiAvalancheTest.java               # 防雪崩测试
    ├── redis/                           # Redis 测试（预留）
    └── integration/                     # 集成测试（预留）
```

### 测试用例

#### Caffeine 测试用例

1. **CaffeineBasicOperationsTest** - 基础操作测试 (9个测试方法)
    - 覆盖 CRUD 基本操作
    - 测试不同数据类型
    - 测试批量操作
2. **CaffeineExpirationTest** - 过期策略测试 (6个测试方法)
    - 测试 expire-after-write
    - 测试 expire-after-access
    - 测试最大容量淘汰
    - 测试独立配置
    - 测试随机偏移
3. **CaffeineConcurrencyTest** - 并发测试 (6个测试方法)
    - 测试并发读写
    - 测试高并发一致性
    - 性能压测（支持 >1000 ops/sec）
4. **AntiAvalancheTest** - 防雪崩测试 (5个测试方法)
    - 测试随机过期防雪崩
    - 测试热点数据防击穿
    - 测试优先级策略
    - 测试批量分散过期

**总计：26+ 个测试方法**

### 测试特点

1. **完整性**:
    - ✅ 覆盖基础 CRUD 操作
    - ✅ 覆盖过期策略
    - ✅ 覆盖并发场景
    - ✅ 覆盖防雪崩策略
2. **实用性**:
    - ✅ 使用真实的测试实体（User, Product）
    - ✅ 模拟真实业务场景
    - ✅ 包含性能测试
    - ✅ 包含压力测试
3. **可维护性**:
    - ✅ 使用测试基类减少重复代码
    - ✅ 使用 @DisplayName 提供中文说明
    - ✅ 使用 Given-When-Then 结构
    - ✅ 清晰的断言消息

### 运行测试

```bash
# 运行所有测试
cd loadup-components-cache
mvn clean test

# 运行特定测试类
mvn test -Dtest=CaffeineBasicOperationsTest
mvn test -Dtest=CaffeineExpirationTest
mvn test -Dtest=CaffeineConcurrencyTest
mvn test -Dtest=AntiAvalancheTest

# 运行所有 Caffeine 测试
mvn test -Dtest=io.github.loadup.components.cache.caffeine.*Test
```

### 测试覆盖范围

**功能测试**:

- ✅ Set/Get/Delete 操作
- ✅ 对象序列化/反序列化
- ✅ 不同数据类型支持
- ✅ 批量操作
- ✅ 空值处理

**性能测试**:

- ✅ 单线程性能
- ✅ 多线程并发性能
- ✅ 大批量数据处理
- ✅ 性能指标输出

**可靠性测试**:

- ✅ 并发一致性
- ✅ 数据完整性
- ✅ 异常处理

**策略测试**:

- ✅ 过期策略验证
- ✅ 淘汰策略验证
- ✅ 防雪崩策略验证
- ✅ 随机偏移验证

## 技术栈

- **JDK**: 17+
- **Spring Boot**: 3.x
- **Caffeine**: 3.1.8（高性能本地缓存）
- **Redis**: Lettuce 客户端
- **序列化**: Jackson

## 许可证

GNU General Public License v3.0 (GPL-3.0) - 详见 [LICENSE](../../LICENSE)

## 联系方式

如有问题或建议，请提交 Issue 或 Pull Request。
