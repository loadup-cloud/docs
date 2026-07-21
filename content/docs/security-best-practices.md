# LoadUp 安全最佳实践

本文档覆盖 LoadUp 项目中涉及安全的开发规范，对应 OWASP Top 10 常见风险类别，提供可操作的代码级指导。

---

## 一、认证与授权

### 1.1 JWT Token 安全配置

```yaml
# application.yml — JWT 配置示例
loadup:
  security:
    jwt:
      secret: ${JWT_SECRET}          # 从环境变量注入，禁止硬编码
      expiration: 7200               # 2 小时，不建议超过 24 小时
      refresh-expiration: 604800     # 7 天 refresh token
```

**规则：**

- JWT secret 必须从环境变量/Vault 注入，**禁止硬编码在代码或配置文件中**
- secret 长度 ≥ 256 位（32 字节）
- Token 中不得包含密码、身份证等敏感字段（只放 userId、roles、tenantId）

### 1.2 方法级权限校验

使用 `@RequirePermission`（`loadup-components-authorization`）：

```java
// ✅ 正确：方法级鉴权
@Service
@RequiredArgsConstructor
public class ConfigItemService {

    @RequirePermission("config:item:create")
    @Transactional(rollbackFor = Exception.class)
    public String create(@Valid ConfigItemCreateCommand cmd) {
        // ...
    }

    @RequirePermission("config:item:list")
    public List<ConfigItemDTO> listAll() {
        // ...
    }
}
```

**权限码命名规范：** `{模块}:{资源}:{操作}`  
示例：`upms:user:create`、`config:item:delete`、`log:audit:query`

### 1.3 Gateway 安全级别

```yaml
# application.yml — 按接口敏感度设置 securityCode
loadup:
  gateway:
    routes:
      - path: /api/v1/config/value         # 公开查询
        method: POST
        target: "bean://configItemService:getValue"
        securityCode: "OFF"                # 明确无需认证
      - path: /api/v1/config/create        # 写操作
        method: POST
        target: "bean://configItemService:create"
        securityCode: "default"            # JWT 认证
      - path: /api/v1/admin/config/delete  # 管理操作
        method: POST
        target: "bean://configItemService:delete"
        securityCode: "default"            # JWT + @RequirePermission 双重校验
```

> ⚠️ `securityCode: "OFF"` 需显式声明，不得将写操作路由设为 OFF。

---

## 二、SQL 注入防护（OWASP A03）

### 2.1 强制使用 QueryWrapper

**禁止**字符串拼接构造 SQL：

```java
// 🚫 禁止：存在 SQL 注入风险
String sql = "SELECT * FROM config_item WHERE config_key = '" + key + "'";

// ✅ 正确：使用 QueryWrapper
QueryWrapper qw = QueryWrapper.create()
    .eq(CONFIG_ITEM_DO.CONFIG_KEY, key)
    .eq(CONFIG_ITEM_DO.DELETED, 0);
return mapper.selectOneByQuery(qw);
```

### 2.2 Like 查询转义

模糊查询时转义特殊字符：

```java
// ✅ 正确：like 查询需转义 %、_、\
String escapedKeyword = keyword.replace("\\", "\\\\")
    .replace("%", "\\%")
    .replace("_", "\\_");

QueryWrapper qw = QueryWrapper.create()
    .likeRaw(CONFIG_ITEM_DO.CONFIG_KEY, escapedKeyword + "%");
```

### 2.3 禁止 SELECT *

```java
// 🚫 禁止
mapper.selectAll();  // 包含 SELECT *

// ✅ 正确：明确指定字段（QueryWrapper 中用 select）
QueryWrapper qw = QueryWrapper.create()
    .select(CONFIG_ITEM_DO.ID, CONFIG_ITEM_DO.CONFIG_KEY, CONFIG_ITEM_DO.CONFIG_VALUE)
    .eq(CONFIG_ITEM_DO.DELETED, 0);
```

---

## 三、敏感数据保护（OWASP A02）

### 3.1 密码字段处理

```java
// ✅ DTO 中隐藏密码字段
public class UserDTO {
    private String userId;
    private String username;

    @JsonIgnore            // 序列化时不输出
    private String password;

    // 或者不在 DTO 中包含 password 字段
}

// ✅ 存储时必须加密（BCrypt）
String encodedPassword = passwordEncoder.encode(rawPassword);
```

### 3.2 日志脱敏

```java
// 🚫 禁止
log.info("用户登录：phone={}, password={}", phone, password);

// ✅ 正确：脱敏后记录
log.info("用户登录：phone={}", maskPhone(phone));

// 脱敏工具（loadup-commons-util）
// maskPhone("13812345678")   → "138****5678"
// maskEmail("user@test.com") → "u***@test.com"
// maskIdCard("320102...")    → "3201**********1234"
```

### 3.3 配置项加密存储

对于 `config_item` 表中涉及密钥、密码的配置：

```java
// ConfigItemDO 中 encrypted=1 的配置值在存储前需加密
// 通过 loadup-components-signature 或 AES 加密
if (cmd.isEncrypted()) {
    value = encryptionService.encrypt(rawValue);
}
```

### 3.4 禁止 Token 入日志

```java
// 🚫 禁止
log.debug("Request headers: {}", request.getHeaders());  // 可能含 Authorization

// ✅ 正确：明确排除敏感 header
log.debug("Request path: {}, method: {}", request.getPath(), request.getMethod());
```

---

## 四、输入验证（OWASP A03/A08）

### 4.1 Command 层级联校验

```java
// ✅ Command 类上加 JSR-380 约束
public class ConfigItemCreateCommand {

    @NotBlank(message = "配置键不能为空")
    @Size(max = 200, message = "配置键长度不超过 200")
    @Pattern(regexp = "^[a-zA-Z0-9._-]+$", message = "配置键只允许字母、数字、点、下划线、连字符")
    private String configKey;

    @Size(max = 10000, message = "配置值长度不超过 10000")
    private String configValue;
}

// ✅ Service 层加 @Valid 触发验证
public String create(@Valid ConfigItemCreateCommand cmd) { ... }
```

### 4.2 分页参数防护

```java
// ✅ 分页查询加上限校验，避免超大查询
public PageDTO<ConfigItemDTO> listByPage(ConfigItemQuery query) {
    if (query.getPageSize() > 200) {
        query.setPageSize(200);   // 强制上限 200 条
    }
    // ...
}
```

### 4.3 文件上传校验（DFS 组件）

```java
// ✅ 上传时校验文件类型和大小
if (!ALLOWED_TYPES.contains(file.getContentType())) {
    throw new IllegalArgumentException("不支持的文件类型：" + file.getContentType());
}
if (file.getSize() > MAX_FILE_SIZE) {
    throw new IllegalArgumentException("文件大小超过限制");
}
```

---

## 五、CSRF 防护（OWASP A01）

LoadUp Gateway 使用 JWT Bearer Token，天然防御 CSRF（Token 无法通过 Cookie 伪造）。

**规则：**

- 所有写操作必须通过 `Authorization: Bearer <token>` Header 传递 Token
- 禁止将 Token 写入 Cookie

---

## 六、依赖安全（OWASP A06）

### 6.1 检查高危漏洞

```bash
# OWASP Dependency-Check（本地）
mvn org.owasp:dependency-check-maven:check

# 查看报告
open target/dependency-check-report.html
```

### 6.2 版本管理规范

- 所有三方依赖版本在 `loadup-dependencies/pom.xml` 中统一管理
- 定期（每季度）扫描 `mvn versions:display-dependency-updates`
- 高危 CVE（CVSS ≥ 7.0）必须在下一个迭代修复

### 6.3 已知敏感依赖版本要求

| 依赖                  | 最低安全版本  | 说明                |
|---------------------|---------|-------------------|
| `jjwt-api`          | 0.12.x  | 旧版 0.9.x 有已知漏洞    |
| `spring-boot`       | 3.4.x   | 避免 3.0-3.2 中的已知漏洞 |
| `mysql-connector-j` | 8.0.33+ | 修复连接池绕过漏洞         |

---

## 七、日志安全

### 7.1 日志记录规范

```java
// ✅ 正确：记录操作事件，不记录敏感内容
log.info("用户 [{}] 创建了配置项 [{}]", userId, configKey);

// 🚫 禁止：记录完整请求体（可能含敏感数据）
log.debug("请求体：{}", JSON.toJSONString(requestBody));

// 🚫 禁止：记录密码、Token
log.info("用户认证：username={}, token={}", username, token);
```

### 7.2 操作日志（log 模块）

通过 `loadup-modules-log` 记录敏感操作：

```java
// 在 Service 方法上使用 @OperationLog 注解（log 模块 AOP 拦截）
@OperationLog(module = "config", operation = "create", description = "创建配置项")
public String create(@Valid ConfigItemCreateCommand cmd) { ... }
```

---

## 八、多租户数据隔离

### 8.1 租户 ID 不得由外部传入

```java
// 🚫 禁止：从请求参数读取 tenantId
String tenantId = request.getParameter("tenantId");

// ✅ 正确：从认证上下文中获取
String tenantId = SecurityContext.getCurrentTenant();
```

### 8.2 跨租户查询防护

Gateway 层的 `TenantInterceptor` 自动注入 `tenant_id` 过滤条件，但需要确认：

- 不使用绕过 MyBatis-Flex 的原生 SQL
- 批量查询仍通过 `BaseMapper` 进行

---

## 九、安全配置检查 Checklist（PR 必检）

```
安全 Checklist：

认证授权
[ ] 写操作接口均有 @RequirePermission 或 securityCode=default
[ ] 公开接口（securityCode=OFF）已经过评审
[ ] JWT secret 来自环境变量，非硬编码

数据保护
[ ] DTO 中密码/Token 字段标注 @JsonIgnore
[ ] 新增敏感字段已在日志中脱敏
[ ] config_item 中加密字段设置 encrypted=1

SQL 安全
[ ] 无字符串拼接 SQL
[ ] Like 查询已转义特殊字符
[ ] 无 SELECT * 查询

输入校验
[ ] Command 类有 JSR-380 约束注解
[ ] Service 层有 @Valid 触发校验
[ ] 分页查询有 PageSize 上限

依赖
[ ] 无新增高危 CVE 依赖
[ ] 新增依赖已在 loadup-dependencies 中声明版本
```
