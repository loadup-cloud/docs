# 重构工程手册

> 适用于 LoadUp 项目中任意规模的重构任务。遵循本手册可降低重构引发的回归风险，并保证重构结果符合 COLA 4.0 分层规范。

---

## 重构前置检查清单

在开始任何重构前，确认以下全部条件：

- [ ] 已有测试覆盖重构范围（集成测试优先）。如缺失，**先补测试**再重构
- [ ] 本地 `mvn clean verify` 全部通过（绿色 baseline）
- [ ] 已确认重构范围不跨越多个 sprint，大重构先拆解为独立小步骤
- [ ] 重构分支命名：`refactor/{模块}-{简短描述}`（如 `refactor/config-split-gateway`）
- [ ] 涉及 Flyway 迁移的，已准备好 `V{n+1}__*.sql` 脚本

---

## 常见重构场景与操作步骤

### 场景 A：将 Service 从直接调用 Mapper 改为通过 Gateway 抽象

**触发条件**：App 层 Service 直接注入 Mapper 或 DO，违反分层原则。

1. 在 `domain.gateway` 包新增 `XxxGateway` 接口，声明业务所需的仓储方法
2. 在 `infrastructure.repository` 包新增 `XxxGatewayImpl implements XxxGateway`，内部使用 MapStruct + Mapper 实现
3. 修改 `app.service.XxxService`：将 `@Autowired XxxMapper` 替换为 `final XxxGateway gateway`
4. 删除 Service 中对 DO / Mapper 的直接引用
5. 运行集成测试，验证行为一致
6. 移除 app 子模块 pom.xml 中对 infrastructure Mapper 的直接依赖（如有）

### 场景 B：从 DO 中抽取 domain model（纯 POJO）

**触发条件**：domain 层直接使用带 `@Table` 的 DO，违反 domain 零 ORM 原则。

1. 在 `domain.model` 包新增 `XxxEntity`（纯 POJO，无任何注解），包含业务字段
2. 在 `infrastructure.converter` 包新增 `XxxConverter`（MapStruct `@Mapper`），实现 `toModel` / `toEntity`
3. 修改 `XxxGateway` 接口：返回值和参数改为 `XxxEntity`
4. 修改 `XxxGatewayImpl`：在读写路径调用 converter 转换
5. 修改 Service：使用 `XxxEntity` 而非 `XxxDO`
6. 在 domain 子模块 pom.xml 中确认无 ORM 依赖

### 场景 C：拆分过大的 Service 方法

**触发条件**：单个 Service 方法超过 50 行或承担多个业务职责。

1. 识别方法内独立的业务子步骤（校验 / 查询 / 构建 / 持久化 / 事件通知）
2. 将各子步骤提取为 `private` 方法，每个方法不超过 20 行
3. 需要复用的子步骤可提取为 package-private 方法（便于单元测试）
4. 保持原方法签名不变（公开 API 不破坏下游）
5. 补充单元测试覆盖各子步骤的边界条件

### 场景 D：升级模块依赖版本

**触发条件**：三方库版本过期或存在安全漏洞。

1. 所有版本变更在 `loadup-dependencies/pom.xml` 完成，子模块无需改动
2. 修改 `<properties>` 中的版本号
3. 执行 `mvn clean verify -U` 验证全量编译和测试通过
4. 检查升级的 BREAKING CHANGE（查阅依赖的 CHANGELOG）
5. 重点回归：MyBatis-Flex / Spring Boot / Lombok 的升级需全模块测试

### 场景 E：消除模块横向依赖

**触发条件**：`loadup-modules-xxx` 不应依赖 `loadup-modules-yyy`。

1. 识别被依赖的具体类（通常是 DTO 或 Command）
2. 如果被依赖的只是数据结构，将其移至 `loadup-commons-dto` 或 `loadup-commons-api`
3. 如果被依赖的是业务逻辑，通过 Gateway 接口抽象调用，或考虑合并为一个模块
4. 修改 pom.xml，移除横向依赖声明
5. 执行 `mvn dependency:analyze` 确认无 undeclared/unused 依赖

### 场景 F：新增 Flyway 迁移（不破坏现有数据）

1. 在 `src/main/resources/db/migration/` 新建 `V{n+1}__{描述}.sql`
2. 使用 `ALTER TABLE ... ADD COLUMN ... DEFAULT xxx` 而非 `MODIFY` 列（保持向后兼容）
3. 新列允许为 NULL 或有默认值（避免存量数据违反约束）
4. 同步更新 `{mod}-test/src/test/resources/schema.sql`（与生产 schema 保持一致）
5. 本地运行 `mvn clean verify` 验证 Testcontainers 能正确应用迁移

---

## 重构验收标准

重构完成后，必须满足以下全部条件才能合并 PR：

| 检查项     | 验收命令 / 方法                                                    |
|---------|--------------------------------------------------------------|
| 代码格式化通过 | `mvn spotless:check`                                         |
| 全量构建通过  | `mvn clean verify`                                           |
| 分层规范合规  | 对照 [code-review-checklist.md](code-review-checklist.md) 逐项确认 |
| 集成测试覆盖  | 核心路径有 `*IT.java` 集成测试                                        |
| 无循环依赖   | `mvn dependency:analyze -pl {module}` 无 CYCLE                |
| PR 描述包含 | 重构动机、影响范围、测试结论                                               |

---

## 重构风险矩阵

| 改动范围                    | 风险  | 对策                                 |
|-------------------------|-----|------------------------------------|
| 单 Service 方法内部          | 低   | 单元测试覆盖即可                           |
| 新增 Gateway 抽象层          | 低-中 | 集成测试验证数据读写                         |
| 修改 DO 字段 / 表结构          | 中   | Flyway 脚本 + schema.sql 同步          |
| 修改公共 BaseDO / Converter | 高   | 全量 `mvn clean verify` + Staging 验证 |
| 拆分 / 合并业务模块             | 高   | 独立分支 + 分阶段迁移 + 灰度发布                |
| 升级 Spring Boot 主版本      | 高   | 专项 spike 分支，全量回归                   |

---

## 常见陷阱

1. **MapStruct 生成失败**：修改 DO 后需重新执行 `mvn generate-sources` 才能更新 Converter 实现，否则编译报错
2. **APT 表定义过期**：`Tables.java` 是 APT 生成产物，DO 字段变更后必须重新编译 infrastructure 模块
3. **Testcontainers schema 不同步**：生产 Flyway 加了字段但测试 `schema.sql` 没更新，会导致 IT 测试 insert 失败
4. **Bean 名覆盖**：多个 AutoConfiguration 扫描到同名 Bean，排查 `@ConditionalOnMissingBean` 是否正确配置
5. **软删除过滤失效**：`QueryWrapper` 中手写条件覆盖了 MyBatis-Flex 的全局软删除过滤，需补 `.eq(XxxDO::getDeleted, 0)`
