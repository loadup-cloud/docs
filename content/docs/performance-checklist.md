# LoadUp 性能检查清单

本文档列出 LoadUp 开发中需要关注的性能优化要点，按层级分区，供开发和 Code Review 使用。

---

## 一、数据库查询层（最高优先级）

### 1.1 禁止 N+1 查询

**问题：** 先查列表，再对每条记录单独查关联数据。

```java
// 🚫 禁止：N+1 查询
List<DictType> types = dictTypeGateway.findAll();
types.forEach(type -> {
    List<DictItem> items = dictItemGateway.findByDictCode(type.getDictCode()); // N 次查询
    type.setItems(items);
});

// ✅ 正确：批量 IN 查询
List<String> dictCodes = types.stream().map(DictType::getDictCode).toList();
List<DictItem> allItems = dictItemGateway.findByDictCodeIn(dictCodes);
Map<String, List<DictItem>> itemsByCode = allItems.stream()
    .collect(Collectors.groupingBy(DictItem::getDictCode));
```

### 1.2 索引利用

```java
// ✅ 确认查询条件有对应索引
QueryWrapper qw = QueryWrapper.create()
    .eq(CONFIG_ITEM_DO.TENANT_ID, tenantId)   // idx_tenant_id
    .eq(CONFIG_ITEM_DO.CATEGORY, category)     // idx_category / 复合索引
    .eq(CONFIG_ITEM_DO.DELETED, 0);

// 复合索引字段顺序：高区分度字段靠前
// KEY idx_tenant_category (tenant_id, category, enabled)
```

**新增查询时必须确认：**

- [ ] WHERE 条件字段有索引覆盖
- [ ] 复合索引满足最左前缀原则
- [ ] `EXPLAIN` 确认不出现 `type=ALL`（全表扫描）

```bash
# 本地验证查询计划
EXPLAIN SELECT id, config_key, config_value
FROM config_item
WHERE tenant_id = 'xxx' AND category = 'system' AND deleted = 0;
```

### 1.3 明确指定查询字段

```java
// 🚫 禁止：SELECT *（尤其避免拉取大字段 TEXT/BLOB）
QueryWrapper qw = QueryWrapper.create().where(CONFIG_ITEM_DO.DELETED.eq(0));
mapper.selectAll();  // 会 SELECT *

// ✅ 正确：显式指定列表接口必要字段
QueryWrapper qw = QueryWrapper.create()
    .select(
        CONFIG_ITEM_DO.ID,
        CONFIG_ITEM_DO.CONFIG_KEY,
        CONFIG_ITEM_DO.CATEGORY,
        CONFIG_ITEM_DO.ENABLED
    )
    .eq(CONFIG_ITEM_DO.DELETED, 0);
```

### 1.4 分页强制上限

```java
// ✅ 分页查询必须限制最大页面大小
public PageDTO<ConfigItemDTO> listByPage(ConfigItemQuery query) {
    int safePageSize = Math.min(query.getPageSize(), 200);  // 单页上限 200
    Page<ConfigItemDO> page = mapper.paginateAs(
        query.getPageNum(), safePageSize, qw, ConfigItemDO.class);
    // ...
}
```

### 1.5 批量写入

```java
// 🚫 禁止：循环单条 insert
items.forEach(item -> mapper.insert(toEntity(item)));

// ✅ 正确：批量 insert，每批 ≤ 1000 条
int batchSize = 500;
for (int i = 0; i < items.size(); i += batchSize) {
    List<DictItemDO> batch = items.subList(i, Math.min(i + batchSize, items.size()))
        .stream().map(this::toEntity).toList();
    mapper.insertBatch(batch);
}
```

---

## 二、缓存策略

### 2.1 本地缓存（Caffeine）适用场景

适合读多写少、数据变更可容忍短暂延迟的场景：

```java
@Component
public class ConfigLocalCache {

    // 规格：10 分钟过期，最多 1000 条（根据数据量调整）
    private final Cache<String, ConfigItem> cache = Caffeine.newBuilder()
        .expireAfterWrite(Duration.ofMinutes(10))
        .maximumSize(1000)
        .recordStats()           // 开启统计（便于监控命中率）
        .build();

    public ConfigItem getOrLoad(String key, Supplier<ConfigItem> loader) {
        return cache.get(key, k -> loader.get());
    }

    public void evict(String key) {
        cache.invalidate(key);
    }

    // 监控：每分钟打印命中率（接入 Micrometer 后自动上报）
    public CacheStats stats() {
        return cache.stats();
    }
}
```

**必须 evict 的时机：**

```java
@Transactional(rollbackFor = Exception.class)
public void update(ConfigItemUpdateCommand cmd) {
    gateway.update(toModel(cmd));
    localCache.evict(cmd.getConfigKey());   // 写后立即失效
}

@Transactional(rollbackFor = Exception.class)
public void delete(String configKey) {
    gateway.deleteById(configKey);
    localCache.evict(configKey);            // 删除后立即失效
}
```

### 2.2 分布式缓存（Redis/Redisson）适用场景

适合多实例部署、需要跨节点一致性的场景：

```java
// 使用 CacheService 抽象（loadup-components-cache-api）
// 不直接调用 RedissonClient，通过 CacheService 接口操作
private final CacheService cacheService;

public UserDTO getUserById(String userId) {
    String cacheKey = "user:" + userId;
    return cacheService.getOrLoad(cacheKey, Duration.ofMinutes(30),
        () -> converter.toDTO(gateway.findById(userId).orElseThrow()));
}
```

### 2.3 缓存穿透防护

```java
// ✅ 对查询结果为空也缓存（防止缓存穿透）
ConfigItem item = localCache.getOrLoad(key, () -> {
    return gateway.findByKey(key).orElse(ConfigItem.EMPTY);  // 空对象模式
});
if (item == ConfigItem.EMPTY) {
    return null;
}
```

---

## 三、事务管理

### 3.1 事务粒度

```java
// 🚫 禁止：在事务中执行远程调用或耗时操作
@Transactional
public String create(ConfigItemCreateCommand cmd) {
    gateway.save(item);
    externalApi.call(item.getConfigKey());  // 远程调用在事务内，会锁定更长时间
}

// ✅ 正确：先提交事务，再做异步操作
@Transactional(rollbackFor = Exception.class)
public String create(ConfigItemCreateCommand cmd) {
    gateway.save(item);
    return item.getId();
}

// 事务外单独触发
public String createAndNotify(ConfigItemCreateCommand cmd) {
    String id = create(cmd);             // 事务提交
    notifyService.sendAsync(id);         // 异步通知，事务外
    return id;
}
```

### 3.2 只读事务

查询方法声明为只读事务，数据库可做优化：

```java
// ✅ 只读查询加 readOnly=true
@Transactional(readOnly = true)
public List<ConfigItemDTO> listAll() {
    return gateway.findAll().stream().map(this::toDTO).toList();
}
```

---

## 四、Service 层

### 4.1 避免大对象返回

```java
// ✅ 列表接口只返回摘要字段，详情接口返回完整字段
// 列表 DTO（无 configValue 等大字段）
public class ConfigItemSummaryDTO {
    private String id;
    private String configKey;
    private String category;
    private Boolean enabled;
}

// 详情 DTO（含全部字段）
public class ConfigItemDTO extends ConfigItemSummaryDTO {
    private String configValue;
    private String description;
    // ...
}
```

### 4.2 流式处理大数据集

```java
// 大量数据导出时使用游标查询，避免一次性加载到内存
// MyBatis-Flex 游标查询
mapper.selectCursorByQuery(qw, cursor -> {
    cursor.forEach(row -> writer.writeRow(converter.toDTO(row)));
});
```

---

## 五、连接池配置参考

### 5.1 HikariCP 推荐配置

```yaml
# application.yml
spring:
  datasource:
    hikari:
      pool-name: LoadupMainPool
      minimum-idle: 5
      maximum-pool-size: 20            # 根据 CPU 核数调整：cores * 2 + disk_spindles
      connection-timeout: 30000        # 30 秒，获取连接超时
      idle-timeout: 600000             # 10 分钟，空闲连接超时
      max-lifetime: 1800000            # 30 分钟，最大生命周期
      connection-test-query: SELECT 1
      leak-detection-threshold: 60000  # 60 秒连接泄漏检测
```

### 5.2 连接泄漏排查

```bash
# 开启泄漏检测日志
logging:
  level:
    com.zaxxer.hikari: DEBUG

# 日志中查找 Apparent connection leak detected
grep "Apparent connection leak" logs/application.log
```

---

## 六、性能 Checklist（Code Review 必检）

```
数据库查询
[ ] 无 N+1 查询（列表+关联数据均批量查询）
[ ] WHERE 条件字段有索引覆盖
[ ] 无 SELECT * 查询（显式指定字段）
[ ] 分页查询有 pageSize 上限（≤ 200）
[ ] 批量写入使用 insertBatch（单批 ≤ 1000）

缓存
[ ] 读多写少数据有本地缓存
[ ] 写操作后有 evict（不遗漏）
[ ] 缓存空值防止缓存穿透

事务
[ ] 事务中无远程调用
[ ] 纯查询方法有 @Transactional(readOnly = true)
[ ] 事务粒度最小化（不含非必要操作）

接口设计
[ ] 列表接口返回摘要 DTO（无大字段）
[ ] 大数据导出使用游标/分批处理
```

---

## 七、常见性能反模式速查

| 反模式         | 影响                | 正确做法                     |
|-------------|-------------------|--------------------------|
| N+1 查询      | 接口延迟随数据量线性增长      | 批量 IN 查询                 |
| SELECT *    | 拉取无用大字段，网络/内存浪费   | 显式指定字段                   |
| 无分页上限       | 一次返回万条数据导致 OOM    | 强制 pageSize ≤ 200        |
| 事务中远程调用     | 长事务锁行，并发下性能急剧下降   | 事务外异步处理                  |
| 写后不 evict   | 缓存读到脏数据           | 写操作后立即 evict             |
| 单条循环 insert | 批量插入 1000 条耗时 10x | mapper.insertBatch()     |
| 全表查询无 WHERE | 全表扫描              | 必须有 deleted/tenant_id 过滤 |
