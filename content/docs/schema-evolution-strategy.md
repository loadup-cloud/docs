# LoadUp Schema 演进策略

本文档定义 LoadUp 项目中数据库 Schema 变更的全生命周期管理规范，包括：

- Flyway 脚本命名与存放规则
- 各类 DDL 变更的安全操作方式
- 多租户环境下的特殊注意事项
- 测试 schema 的同步维护要求
- 回滚策略

---

## 一、核心原则

| 原则            | 说明                                         |
|---------------|--------------------------------------------|
| **已提交脚本不可修改** | 任何已应用到任何环境的 Flyway 脚本严禁修改，否则 checksum 校验失败 |
| **向前兼容**      | 部署新版本时，旧版本代码和新 schema 必须能共存（零宕机部署）         |
| **先加后删**      | 新增列/表先上线，再上代码使用；删除列/表先删代码，再删列/表            |
| **测试同步**      | 测试 schema.sql 必须与生产 Flyway 脚本最终状态完全一致      |

---

## 二、文件组织结构

### 2.1 Flyway 脚本位置

每个业务模块的 infrastructure 层存放 Flyway 脚本：

```
modules/loadup-modules-{mod}/
├── schema.sql                          # 完整建表语句（供测试、初次查看）
└── loadup-modules-{mod}-infrastructure/
    └── src/main/resources/
        └── db/migration/
            ├── V1__init_{mod}.sql      # 初始建表
            ├── V2__add_{column}.sql    # 增量变更
            └── V3__create_{table}.sql  # 新增表
```

### 2.2 schema.sql 的用途与维护

`schema.sql`（模块根目录）是所有 Flyway 脚本合并后的最终状态，用于：

1. 集成测试（`spring.sql.init.schema-locations: classpath:schema.sql`）
2. 新成员快速搭建本地环境
3. Schema 全景阅读

**维护规则：** 每次新增 Flyway 脚本后，必须同步更新 `schema.sql`，两者必须保持一致。

---

## 三、命名规范

### 3.1 脚本命名格式

```
V{version}__{描述}.sql
    ↑          ↑
   版本号    双下划线分隔，描述用小写_连接
```

| 示例                              | 说明    |
|---------------------------------|-------|
| `V1__init_config.sql`           | 初始建表  |
| `V2__add_config_item_tags.sql`  | 新增列   |
| `V3__create_audit_log.sql`      | 新增表   |
| `V4__add_index_created_at.sql`  | 新增索引  |
| `V5__drop_column_old_field.sql` | 删除废弃列 |

### 3.2 版本号规则

- 版本号在**同一数据库实例**内必须全局唯一
- 多模块共用一个 DB（多数场景）时，版本号跨模块也须唯一
- 建议按时间戳命名避免冲突：`V20260402_001__init_config.sql`

**⚠️ 多模块版本号建议前缀策略：**

| 模块     | 版本号前缀     | 示例                       |
|--------|-----------|--------------------------|
| upms   | 1000-1999 | `V1001__init_upms.sql`   |
| config | 2000-2999 | `V2001__init_config.sql` |
| log    | 3000-3999 | `V3001__init_log.sql`    |
| 新模块    | 4000+     | `V4001__init_xxx.sql`    |

---

## 四、各类 DDL 操作规范

### 4.1 新建表

**脚本模板：**

```sql
-- V2001__init_config.sql
-- Module: loadup-modules-config
-- Description: 初始化配置模块表结构

CREATE TABLE IF NOT EXISTS config_item (
    id             VARCHAR(64)  NOT NULL                                   COMMENT 'ID',
    tenant_id      VARCHAR(64)                                             COMMENT '租户ID',
    config_key     VARCHAR(200) NOT NULL                                   COMMENT '配置键',
    config_value   TEXT                                                    COMMENT '配置值',
    -- ... 业务字段 ...
    created_by     VARCHAR(64)                                             COMMENT '创建人',
    created_at     DATETIME     NOT NULL          COMMENT '创建时间',
    updated_by     VARCHAR(64)                                             COMMENT '更新人',
    updated_at     DATETIME              NOT NULL  COMMENT '更新时间',
    deleted        TINYINT      NOT NULL DEFAULT 0                         COMMENT '删除标记',
    PRIMARY KEY (id),
    KEY idx_tenant_id (tenant_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='表注释';
```

**每张表必须包含的 5 个标准字段（缺一不可）：**

| 字段           | 类型            | 约束                                   |
|--------------|---------------|--------------------------------------|
| `id`         | `VARCHAR(64)` | `NOT NULL PRIMARY KEY`               |
| `tenant_id`  | `VARCHAR(64)` | 可为 NULL                              |
| `created_at` | `DATETIME`    | `NOT NULL ` |
| `updated_at` | `DATETIME`    | `NOT NULL`   |
| `deleted`    | `TINYINT`     | `NOT NULL DEFAULT 0`                 |

> ⚠️ 主键使用 `VARCHAR(64)`，业务层 UUID 赋值。禁止 `BIGINT AUTO_INCREMENT`。  
> ⚠️ 禁止使用 `BOOLEAN`/`BOOL`，统一用 `TINYINT(0/1)`。

---

### 4.2 新增列

**安全部署顺序：**

1. 先执行 Flyway 脚本（新增列，有默认值）
2. 再部署使用新列的代码

```sql
-- V2002__add_config_item_description.sql
ALTER TABLE config_item
    ADD COLUMN description VARCHAR(500) COMMENT '描述' AFTER config_value;
```

**规则：**

- 新增列必须有默认值，或允许 NULL（不能对现有行造成约束违反）
- 禁止 `NOT NULL` 无默认值的列（会锁全表或报错）

---

### 4.3 修改列类型或长度

**高风险操作，需：**

1. 评估数据量（超过 100 万行需要 Online DDL 方案）
2. 使用 `ALGORITHM=INPLACE, LOCK=NONE`（MySQL 8.0+）

```sql
-- V2003__expand_config_value_length.sql
ALTER TABLE config_item
    MODIFY COLUMN config_key VARCHAR(500) NOT NULL COMMENT '配置键'
    ALGORITHM=INPLACE, LOCK=NONE;
```

> MySQL 8.0 `INPLACE` 支持的场景有限，大表变更建议使用 `pt-online-schema-change` 或 `gh-ost`。

---

### 4.4 重命名列

**最危险的操作，禁止直接重命名。** 使用以下三步安全方案：

```
步骤 1：新增新名列（新旧列并存）
步骤 2：部署代码读新列、双写新旧列（灰度）
步骤 3：确认全量切换后，删除旧列（部署最终版本后下一个迭代）
```

```sql
-- V2004__add_label_column.sql（步骤1：新增）
ALTER TABLE config_item
    ADD COLUMN config_label VARCHAR(200) COMMENT '配置标签（原 display_name）' AFTER config_key;

-- V2006__drop_old_display_name.sql（步骤3：若干迭代后删旧列）
ALTER TABLE config_item
    DROP COLUMN display_name;
```

---

### 4.5 删除列

**安全部署顺序（先删代码，再删列）：**

1. 部署不再使用该列的代码（DO 中去掉该字段，MapStruct 中去掉映射）
2. 确认线上无引用后，执行删列 Flyway 脚本

```sql
-- V2005__drop_obsolete_sort_order.sql
ALTER TABLE dict_item
    DROP COLUMN sort_order;
```

---

### 4.6 新增索引

索引操作在 MySQL 8.0 中通常是在线的（不锁表），但仍需评估：

```sql
-- V2006__add_index_config_category.sql
CREATE INDEX idx_category_enabled
    ON config_item (category, enabled)
    ALGORITHM=INPLACE, LOCK=NONE;
```

---

### 4.7 删除表

**最高风险操作，必须：**

1. 确认所有模块代码中无任何对该表的引用（Mapper、DO、QueryWrapper）
2. 先将表重命名保留 1 个迭代（`_deprecated_` 前缀）
3. 下一迭代再彻底删除

```sql
-- V2009__rename_deprecated_old_table.sql（步骤1：重命名保留）
RENAME TABLE old_config TO _deprecated_old_config_20260402;

-- V2011__drop_deprecated_old_table.sql（步骤2：下个迭代删除）
DROP TABLE IF EXISTS _deprecated_old_config_20260402;
```

---

## 五、多租户迁移注意事项

LoadUp 的多租户依赖 `tenant_id` 软隔离，所有表必须有 `tenant_id` 列。

### 5.1 新表的租户索引

```sql
-- 所有带多租户的表必须加 idx_tenant_id
KEY idx_tenant_id (tenant_id),
-- 或复合索引（常用查询维度）
KEY idx_tenant_category (tenant_id, category, enabled)
```

### 5.2 迁移已有数据时填充 tenant_id

```sql
-- 数据迁移时需填充 tenant_id
UPDATE config_item SET tenant_id = 'default_tenant' WHERE tenant_id IS NULL;
```

### 5.3 单租户场景

单租户部署中 `tenant_id` 可留空，但列本身不能删除（统一 BaseDO 定义）。

---

## 六、测试 Schema 同步规范

### 6.1 三个 schema.sql 的同步要求

| 文件位置                                                                                     | 用途                | 维护规则                |
|------------------------------------------------------------------------------------------|-------------------|---------------------|
| `modules/loadup-modules-{mod}/schema.sql`                                                | 可读性参考，PR review 用 | 每次 Flyway 变更后同步     |
| `modules/.../loadup-modules-{mod}-infrastructure/src/main/resources/db/migration/V*.sql` | 生产迁移              | 版本递增，不可修改已提交脚本      |
| `modules/loadup-modules-{mod}-test/src/test/resources/schema.sql`                        | 集成测试建表            | 必须与生产 schema 最终状态一致 |

### 6.2 验证同步的方法

```bash
# 对比模块根 schema.sql 和测试 schema.sql 的表结构（忽略注释差异）
diff <(grep "^CREATE TABLE\|^\s*[a-zA-Z].*COMMENT\|PRIMARY KEY\|KEY " \
    modules/loadup-modules-config/schema.sql | sort) \
     <(grep "^CREATE TABLE\|^\s*[a-zA-Z].*COMMENT\|PRIMARY KEY\|KEY " \
    modules/loadup-modules-config/loadup-modules-config-test/src/test/resources/schema.sql | sort)
```

### 6.3 PR 规则

每个包含 Flyway 迁移脚本的 PR，必须同步：

- [ ] `schema.sql`（模块根目录）
- [ ] `*-test/src/test/resources/schema.sql`
- [ ] `DO` 实体类字段（如有新增列）
- [ ] MapStruct `Converter`（如有新增字段映射）

---

## 七、回滚策略

Flyway 社区版不支持自动回滚。LoadUp 采用**向前修复**策略：

### 7.1 回滚 DDL 变更

| 原操作      | 回滚方式                                             |
|----------|--------------------------------------------------|
| 新增列      | `V{n+1}__rollback_drop_column.sql` 删除该列          |
| 新增表      | `V{n+1}__rollback_drop_table.sql` 删除该表           |
| 扩展列长度    | 先验证数据可以收缩，再 `V{n+1}__rollback_shrink_column.sql` |
| 删除列（已执行） | 无法回滚，须重新 `ADD COLUMN`，数据已丢失                      |

> **落地原则：** 删列前必须确保 100% 确认不需要该数据，建议先逻辑废弃（加 `_deprecated_` 注释或重命名）。

### 7.2 数据回滚

高风险数据操作前需备份：

```sql
-- 操作前备份
CREATE TABLE config_item_backup_20260402 AS SELECT * FROM config_item;

-- 确认后删除备份
DROP TABLE config_item_backup_20260402;
```

---

## 八、本地开发快速参考

### 8.1 新增 Flyway 脚本的完整 checklist

```
新 DDL 变更 checklist：

[ ] 1. 在 db/migration/ 下创建 V{n}__{描述}.sql，版本号不重复
[ ] 2. 脚本中包含 IF NOT EXISTS / IF EXISTS 保护（幂等）
[ ] 3. 新建表包含 5 个标准字段（id/tenant_id/created_at/updated_at/deleted）
[ ] 4. 同步更新 modules/{mod}/schema.sql
[ ] 5. 同步更新 {mod}-test/src/test/resources/schema.sql
[ ] 6. 更新对应 DO 实体类字段
[ ] 7. 更新 MapStruct Converter（如需）
[ ] 8. 运行集成测试验证：mvn test -pl modules/loadup-modules-{mod}/loadup-modules-{mod}-test
```

### 8.2 在集成测试中验证迁移

```bash
# 运行指定模块的集成测试（启动 MySQL Testcontainer）
mvn test -pl modules/loadup-modules-config/loadup-modules-config-test

# 仅运行 schema 相关测试
mvn test -pl modules/loadup-modules-config/loadup-modules-config-test \
    -Dtest=*SchemaIT
```

### 8.3 查看当前 Flyway 状态

```bash
# 查看迁移历史（需要连接数据库）
mvn flyway:info -pl modules/loadup-modules-config/loadup-modules-config-infrastructure \
    -Dflyway.url=jdbc:mysql://localhost:3306/loadup \
    -Dflyway.user=root \
    -Dflyway.password=xxx
```

---

## 附：常见反模式

| 反模式                       | 正确做法                |
|---------------------------|---------------------|
| 修改已提交的迁移脚本                | 新建版本号更高的脚本修复        |
| `NOT NULL` 无默认值新增列        | 新增列必须有默认值或允许 NULL   |
| 直接重命名列                    | 三步渐进方案：新增 → 双写 → 删旧 |
| 部署代码前先删列                  | 先删代码，再删列            |
| 测试 schema.sql 滞后于生产       | PR 必须同步更新测试 schema  |
| 多模块版本号重复                  | 按模块区段分配版本号前缀        |
| 使用 `BOOLEAN` 类型           | 统一使用 `TINYINT`（0/1） |
| 表缺少 `tenant_id`/`deleted` | 所有表必须包含 5 个标准字段     |
