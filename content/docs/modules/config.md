# Config 模块（loadup-modules-config）

系统参数管理与数据字典模块，提供运行时可配置的键值存储、变更历史追踪和多级字典体系。

---

## 模块结构

```
loadup-modules-config/
├── loadup-modules-config-client/          # 对外 DTO + Command + Query
├── loadup-modules-config-domain/          # 领域模型（POJO）+ Gateway 接口 + 枚举
├── loadup-modules-config-infrastructure/  # DO + GatewayImpl + 本地缓存
├── loadup-modules-config-app/             # @Service 业务编排 + AutoConfiguration
└── loadup-modules-config-test/            # 集成测试（Testcontainers MySQL）
```

---

## 功能概览

| 功能        | 说明                                                            |
|-----------|---------------------------------------------------------------|
| 系统参数管理    | 全局 Key-Value 配置，支持多种值类型，运行时可修改                                |
| 类型化取值     | `getTypedValue(key, Integer.class, default)` 自动转换类型           |
| 加密标记      | `encrypted=true` 的配置项在 DTO 层自动脱敏为 `******`                    |
| 系统内置保护    | `systemDefined=true` 的配置项禁止删除                                 |
| 变更历史      | 每次 CREATE / UPDATE / DELETE 自动写入 `config_history`             |
| Spring 事件 | 写操作发布 `ConfigChangedEvent`，联动缓存自动 evict                       |
| 数据字典类型    | 字典分类（`DictType`），如 `gender`、`user_status`                     |
| 数据字典项     | 字典条目（`DictItem`），支持级联（`parentValue`）和 CSS 样式类                 |
| 本地缓存      | 基于 Caffeine，配置项 TTL 5 min / 最大 2000 条，字典 TTL 5 min / 最大 500 组 |

---

## 领域模型

### ConfigItem（系统参数）

| 字段              | 类型        | 说明                                                      |
|-----------------|-----------|---------------------------------------------------------|
| `id`            | `String`  | 主键（UUID 无连字符）                                           |
| `configKey`     | `String`  | 配置键（全局唯一）                                               |
| `configValue`   | `String`  | 配置值（加密项 DTO 层展示 `******`）                               |
| `valueType`     | `String`  | 值类型：`STRING / INTEGER / LONG / DOUBLE / BOOLEAN / JSON` |
| `category`      | `String`  | 分类（如 `system`、`upload`、`security`）                      |
| `editable`      | `Boolean` | 是否允许修改                                                  |
| `encrypted`     | `Boolean` | 是否加密存储                                                  |
| `systemDefined` | `Boolean` | 系统内置，禁止删除                                               |
| `sortOrder`     | `Integer` | 排序权重                                                    |
| `enabled`       | `Boolean` | 是否启用                                                    |

### ConfigHistory（变更历史）

| 字段           | 类型           | 说明                         |
|--------------|--------------|----------------------------|
| `configKey`  | `String`     | 关联配置键                      |
| `oldValue`   | `String`     | 变更前值                       |
| `newValue`   | `String`     | 变更后值                       |
| `changeType` | `ChangeType` | `CREATE / UPDATE / DELETE` |
| `operator`   | `String`     | 操作人                        |

### DictType（字典类型）

| 字段              | 类型        | 说明                    |
|-----------------|-----------|-----------------------|
| `dictCode`      | `String`  | 字典编码（全局唯一，如 `gender`） |
| `dictName`      | `String`  | 字典名称（如 `性别`）          |
| `systemDefined` | `Boolean` | 系统内置，禁止删除             |
| `sortOrder`     | `Integer` | 排序权重                  |
| `enabled`       | `Boolean` | 是否启用                  |

### DictItem（字典项）

| 字段            | 类型        | 说明                        |
|---------------|-----------|---------------------------|
| `dictCode`    | `String`  | 所属字典类型编码                  |
| `itemLabel`   | `String`  | 显示标签（如 `男`）               |
| `itemValue`   | `String`  | 字典值（如 `1`）                |
| `parentValue` | `String`  | 父级值，用于级联字典                |
| `cssClass`    | `String`  | CSS 样式类（如 `text-success`） |
| `sortOrder`   | `Integer` | 排序权重                      |

---

## 枚举

### ValueType

```
STRING, INTEGER, LONG, DOUBLE, BOOLEAN, JSON
```

### ChangeType

```
CREATE, UPDATE, DELETE
```

---

## Service API

### ConfigItemService

| 方法                                           | 说明                                  |
|----------------------------------------------|-------------------------------------|
| `listAll()`                                  | 查询所有配置项                             |
| `listByCategory(category)`                   | 按分类查询                               |
| `getByKey(configKey)`                        | 按键查询单条                              |
| `getValue(configKey)`                        | 获取原始字符串值                            |
| `getTypedValue(key, Class<T>, defaultValue)` | 获取类型化值（自动转换）                        |
| `create(cmd)`                                | 创建配置项（写历史 + 发事件）                    |
| `update(cmd)`                                | 更新配置值（校验 editable + 写历史 + 发事件）      |
| `delete(configKey)`                          | 删除配置项（校验 systemDefined + 写历史 + 发事件） |
| `refreshCache()`                             | 手动清空本地缓存                            |

### DictService

| 方法                                      | 说明                 |
|-----------------------------------------|--------------------|
| `listAllTypes()`                        | 查询所有字典类型           |
| `getDictData(dictCode)`                 | 按字典编码查询字典项列表       |
| `getCascadeData(dictCode, parentValue)` | 查询级联子项             |
| `getDictLabel(dictCode, itemValue)`     | 通过值反查标签            |
| `createType(cmd)`                       | 创建字典类型             |
| `deleteType(dictCode)`                  | 删除字典类型（级联删除字典项）    |
| `createItem(cmd)`                       | 创建字典项（写后 evict 缓存） |
| `deleteItem(id)`                        | 删除字典项              |

---

## 缓存策略

基于 **Caffeine** 本地缓存，无需 Redis。

| 缓存                | 最大容量   | TTL   | evict 时机                              |
|-------------------|--------|-------|---------------------------------------|
| 配置项（by configKey） | 2000 条 | 5 min | 写操作后由 `ConfigChangedEvent` 自动触发       |
| 字典项（by dictCode）  | 500 组  | 5 min | `createItem` / `deleteItem` 后手动 evict |

`ConfigChangedEvent` 由 `ConfigItemService` 在每次写操作后发布，`ConfigLocalCache` 通过 `@EventListener` 自动响应，无需显式调用。

---

## Gateway 路由配置

在 `loadup-application/src/main/resources/application.yml` 中添加以下路由：

```yaml
loadup:
  gateway:
    routes:
      # ── 系统参数 ──
      - path: /api/v1/config/list
        method: POST
        target: "bean://configItemService:listAll"
        securityCode: "default"
      - path: /api/v1/config/list-by-category
        method: POST
        target: "bean://configItemService:listByCategory"
        securityCode: "default"
      - path: /api/v1/config/get
        method: POST
        target: "bean://configItemService:getByKey"
        securityCode: "default"
      - path: /api/v1/config/value
        method: POST
        target: "bean://configItemService:getValue"
        securityCode: "OFF"
      - path: /api/v1/config/create
        method: POST
        target: "bean://configItemService:create"
        securityCode: "default"
      - path: /api/v1/config/update
        method: POST
        target: "bean://configItemService:update"
        securityCode: "default"
      - path: /api/v1/config/delete
        method: POST
        target: "bean://configItemService:delete"
        securityCode: "default"
      - path: /api/v1/config/refresh-cache
        method: POST
        target: "bean://configItemService:refreshCache"
        securityCode: "default"
      # ── 数据字典 ──
      - path: /api/v1/dict/types
        method: POST
        target: "bean://dictService:listAllTypes"
        securityCode: "default"
      - path: /api/v1/dict/data
        method: POST
        target: "bean://dictService:getDictData"
        securityCode: "OFF"
      - path: /api/v1/dict/cascade
        method: POST
        target: "bean://dictService:getCascadeData"
        securityCode: "OFF"
      - path: /api/v1/dict/label
        method: POST
        target: "bean://dictService:getDictLabel"
        securityCode: "OFF"
      - path: /api/v1/dict/type/create
        method: POST
        target: "bean://dictService:createType"
        securityCode: "default"
      - path: /api/v1/dict/type/delete
        method: POST
        target: "bean://dictService:deleteType"
        securityCode: "default"
      - path: /api/v1/dict/item/create
        method: POST
        target: "bean://dictService:createItem"
        securityCode: "default"
      - path: /api/v1/dict/item/delete
        method: POST
        target: "bean://dictService:deleteItem"
        securityCode: "default"
```

---

## 数据库表结构

### config_item

```sql
CREATE TABLE IF NOT EXISTS config_item (
    id             VARCHAR(64)  NOT NULL                                          COMMENT 'ID',
    tenant_id      VARCHAR(64)                                                    COMMENT '租户ID',
    config_key     VARCHAR(200) NOT NULL                                          COMMENT '配置键（全局唯一）',
    config_value   TEXT                                                           COMMENT '配置值',
    value_type     VARCHAR(20)  NOT NULL DEFAULT 'STRING'                        COMMENT '值类型: STRING/INTEGER/LONG/DOUBLE/BOOLEAN/JSON',
    category       VARCHAR(50)  NOT NULL DEFAULT 'default'                       COMMENT '分类',
    description    VARCHAR(500)                                                   COMMENT '描述',
    editable       TINYINT      NOT NULL DEFAULT 1                                COMMENT '是否可编辑',
    encrypted      TINYINT      NOT NULL DEFAULT 0                                COMMENT '是否加密存储',
    system_defined TINYINT      NOT NULL DEFAULT 0                                COMMENT '是否系统内置',
    sort_order     INT          NOT NULL DEFAULT 0                                COMMENT '排序',
    enabled        TINYINT      NOT NULL DEFAULT 1                                COMMENT '是否启用',
    created_by     VARCHAR(64)                                                    COMMENT '创建人',
    created_at     DATETIME     NOT NULL                 COMMENT '创建时间',
    updated_by     VARCHAR(64)                                                    COMMENT '更新人',
    updated_at     DATETIME              NOT NULL         COMMENT '更新时间',
    deleted        TINYINT      NOT NULL DEFAULT 0                                COMMENT '删除标记',
    PRIMARY KEY (id),
    UNIQUE KEY uk_config_key (config_key),
    KEY idx_tenant_id (tenant_id),
    KEY idx_category  (category),
    KEY idx_enabled   (enabled)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='系统参数配置表';
```

### dict_type

```sql
CREATE TABLE IF NOT EXISTS dict_type (
    id             VARCHAR(64)  NOT NULL                                          COMMENT 'ID',
    tenant_id      VARCHAR(64)                                                    COMMENT '租户ID',
    dict_code      VARCHAR(100) NOT NULL                                          COMMENT '字典编码（全局唯一）',
    dict_name      VARCHAR(200) NOT NULL                                          COMMENT '字典名称',
    description    VARCHAR(500)                                                   COMMENT '描述',
    system_defined TINYINT      NOT NULL DEFAULT 0                                COMMENT '是否系统内置',
    sort_order     INT          NOT NULL DEFAULT 0                                COMMENT '排序',
    enabled        TINYINT      NOT NULL DEFAULT 1                                COMMENT '是否启用',
    created_by     VARCHAR(64)                                                    COMMENT '创建人',
    created_at     DATETIME     NOT NULL                 COMMENT '创建时间',
    updated_by     VARCHAR(64)                                                    COMMENT '更新人',
    updated_at     DATETIME              NOT NULL         COMMENT '更新时间',
    deleted        TINYINT      NOT NULL DEFAULT 0                                COMMENT '删除标记',
    PRIMARY KEY (id),
    UNIQUE KEY uk_dict_code (dict_code),
    KEY idx_tenant_id (tenant_id),
    KEY idx_enabled   (enabled)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='数据字典类型表';
```

### dict_item

```sql
CREATE TABLE IF NOT EXISTS dict_item (
    id           VARCHAR(64)  NOT NULL                                            COMMENT 'ID',
    tenant_id    VARCHAR(64)                                                      COMMENT '租户ID',
    dict_code    VARCHAR(100) NOT NULL                                            COMMENT '字典编码',
    item_label   VARCHAR(200) NOT NULL                                            COMMENT '显示标签',
    item_value   VARCHAR(200) NOT NULL                                            COMMENT '字典值',
    parent_value VARCHAR(200)                                                     COMMENT '父级值（级联字典）',
    css_class    VARCHAR(100)                                                     COMMENT 'CSS 样式类',
    sort_order   INT          NOT NULL DEFAULT 0                                  COMMENT '排序',
    enabled      TINYINT      NOT NULL DEFAULT 1                                  COMMENT '是否启用',
    created_by   VARCHAR(64)                                                      COMMENT '创建人',
    created_at   DATETIME     NOT NULL                   COMMENT '创建时间',
    updated_by   VARCHAR(64)                                                      COMMENT '更新人',
    updated_at   DATETIME              NOT NULL           COMMENT '更新时间',
    deleted      TINYINT      NOT NULL DEFAULT 0                                  COMMENT '删除标记',
    PRIMARY KEY (id),
    UNIQUE KEY uk_dict_code_value (dict_code, item_value),
    KEY idx_tenant_id    (tenant_id),
    KEY idx_dict_code    (dict_code),
    KEY idx_parent_value (parent_value),
    KEY idx_enabled      (enabled)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='数据字典项表';
```

### config_history

```sql
CREATE TABLE IF NOT EXISTS config_history (
    id          VARCHAR(64)  NOT NULL                                             COMMENT 'ID',
    tenant_id   VARCHAR(64)                                                       COMMENT '租户ID',
    config_key  VARCHAR(200) NOT NULL                                             COMMENT '配置键',
    old_value   TEXT                                                              COMMENT '变更前值',
    new_value   TEXT                                                              COMMENT '变更后值',
    change_type VARCHAR(20)  NOT NULL                                             COMMENT '变更类型: CREATE/UPDATE/DELETE',
    operator    VARCHAR(64)  NOT NULL DEFAULT 'system'                            COMMENT '操作人',
    remark      VARCHAR(500)                                                      COMMENT '备注',
    created_at  DATETIME     NOT NULL                    COMMENT '创建时间',
    updated_at  DATETIME              NOT NULL            COMMENT '更新时间',
    deleted     TINYINT      NOT NULL DEFAULT 0                                   COMMENT '删除标记',
    PRIMARY KEY (id),
    KEY idx_tenant_id  (tenant_id),
    KEY idx_config_key (config_key),
    KEY idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='配置变更历史表';
```

### 预置种子数据

```sql
INSERT IGNORE INTO dict_type (id, dict_code, dict_name, system_defined, sort_order, enabled, created_at)
VALUES
    ('01', 'user_status', '用户状态', 1, 1, 1, NOW()),
    ('02', 'gender',      '性别',     1, 2, 1, NOW()),
    ('03', 'yes_no',      '是否',     1, 3, 1, NOW());

INSERT IGNORE INTO dict_item (id, dict_code, item_label, item_value, sort_order, enabled, created_at)
VALUES
    ('001', 'user_status', '正常', '1', 1, 1, NOW()),
    ('002', 'user_status', '停用', '0', 2, 1, NOW()),
    ('003', 'user_status', '锁定', '2', 3, 1, NOW()),
    ('004', 'gender',      '男',   '1', 1, 1, NOW()),
    ('005', 'gender',      '女',   '2', 2, 1, NOW()),
    ('006', 'gender',      '未知', '0', 3, 1, NOW()),
    ('007', 'yes_no',      '是',   '1', 1, 1, NOW()),
    ('008', 'yes_no',      '否',   '0', 2, 1, NOW());

INSERT IGNORE INTO config_item
    (id, config_key, config_value, value_type, category, description, editable, system_defined, enabled, created_at)
VALUES
    ('c01', 'system.name',                   'LoadUp Framework', 'STRING',  'system',   '系统名称',             0, 1, 1, NOW()),
    ('c02', 'upload.max-file-size',          '10485760',         'LONG',    'upload',   '文件上传大小限制(字节)', 1, 0, 1, NOW()),
    ('c03', 'security.password-expire-days', '90',               'INTEGER', 'security', '密码过期天数',           1, 0, 1, NOW());
```

---

## 测试

测试模块：`loadup-modules-config-test`，使用 Testcontainers 启动真实 MySQL，禁止 `@MockBean` 替代数据库。

```bash
# 运行所有集成测试（需要 Docker）
mvn test -pl modules/loadup-modules-config/loadup-modules-config-test
```

### 测试类说明

| 测试类                   | 场景                                      |
|-----------------------|-----------------------------------------|
| `ConfigItemServiceIT` | 配置项 CRUD、类型化取值、重复键校验、系统内置保护、缓存 evict 验证 |
| `DictServiceIT`       | 字典类型 / 字典项 CRUD、级联查询、标签反查、系统内置保护        |

### 测试资源文件

`src/test/resources/` 目录下包含以下文件：

| 文件                     | 说明                                                     |
|------------------------|--------------------------------------------------------|
| `application.yml`      | 激活 `test` profile（固定内容：`spring.profiles.active: test`） |
| `application-test.yml` | 本地开发配置（Testcontainers reuse=true，print-sql=true）       |
| `application-ci.yml`   | CI 流水线配置（Testcontainers reuse=false，print-sql=false）   |
| `schema.sql`           | 测试数据库 Schema（与生产 schema 完全一致）                          |

