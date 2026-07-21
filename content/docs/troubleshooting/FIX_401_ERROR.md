# 401 错误修复说明

## 问题原因

启动 application 后所有请求都返回 401，原因是：

1. **`loadup-components-security` 引入了 Spring Security**
    - `loadup-modules-upms-app` 依赖了 `loadup-components-security`
    - Spring Security 默认会保护所有端点

2. **之前的配置不再生效**
    - 重构后的 `SecurityAutoConfiguration` 只有 `@EnableMethodSecurity`
    - 没有配置 `SecurityFilterChain`
    - 所有请求都被 Spring Security 默认拦截

3. **缺少必要的依赖**
    - 缺少 `spring-security-web` 依赖

## 修复方案

### 1. 添加 `SecurityFilterChain` 配置

**文件**: `loadup-components-security/src/main/java/.../config/SecurityAutoConfiguration.java`

**修改内容**:

```java
@AutoConfiguration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean(SecurityFilterChain.class)
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .formLogin(AbstractHttpConfigurer::disable)
            .httpBasic(AbstractHttpConfigurer::disable)
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(authorize -> 
                authorize.anyRequest().permitAll());  // 允许所有请求
        
        return http.build();
    }
}
```

**关键点**:

- ✅ `anyRequest().permitAll()` - 允许所有请求通过（因为认证由 Gateway 完成）
- ✅ `@ConditionalOnMissingBean` - 如果应用自定义了 SecurityFilterChain，则不会创建默认的
- ✅ `@PreAuthorize` 仍然有效 - 方法级授权基于 SecurityContext

### 2. 添加依赖

**文件**: `loadup-components-security/pom.xml`

**添加**:

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-web</artifactId>
</dependency>
```

### 3. 移除无效配置

**文件**: `loadup-application/src/main/resources/application.yml`

**移除**:

```yaml
loadup:
  security:
    ignore-urls:      # ❌ 已删除，不再使用
      - /api/v1/auth/login
      - ...
```

**原因**: 认证现在由 Gateway 的 `RouteConfig.securityCode` 控制，不再通过 Spring Security Filter Chain 的白名单。

## 架构说明

### 当前架构（正确的）

```
Client Request
    ↓
Spring Security Filter Chain (permitAll - 放行所有请求)
    ↓
DispatcherServlet
    ↓
Gateway SecurityAction (根据 RouteConfig.securityCode 认证)
    ├─ OFF: 跳过
    ├─ default: JWT 验证
    ├─ signature: 签名验证
    └─ internal: 内部调用验证
    ↓
Proxy Action (转发到业务 Bean)
    ↓
Business Service (@PreAuthorize 检查角色)
```

### 为什么 Spring Security Filter 放行所有请求？

1. **认证由 Gateway 完成**
    - Gateway 的 `SecurityAction` 负责认证
    - 可以根据路由动态选择认证策略（JWT/签名/内部调用）

2. **授权由 `@PreAuthorize` 完成**
    - `@PreAuthorize` 基于 SecurityContext
    - Gateway 认证成功后会填充 SecurityContext
    - Spring Security 的方法拦截器会检查权限

3. **分层清晰**
    - Filter Chain: 放行（不做认证）
    - Gateway: 认证（Authentication）
    - Method Security: 授权（Authorization）

## 配置路由认证

现在需要在路由配置中指定 `securityCode`：

### 示例配置

```yaml
# 方式 1: FILE 存储 (当前使用)
# 在 gateway-config/ 目录下创建路由 JSON 文件

# 方式 2: DATABASE 存储
# 在数据库 t_gateway_route 表中配置
```

### 路由配置示例

```json
{
  "routeId": "auth-login",
  "path": "/api/v1/auth/login",
  "securityCode": "OFF",  // 公开接口，跳过认证
  "proxyType": "bean",
  "targetBean": "authenticationController"
}
```

```json
{
  "routeId": "user-api",
  "path": "/api/v1/users/**",
  "securityCode": "default",  // JWT 认证
  "proxyType": "bean",
  "targetBean": "userController"
}
```

### securityCode 选项

| Code           | 说明     | 使用场景           |
|----------------|--------|----------------|
| `OFF` 或 `null` | 跳过认证   | 登录、注册、公开接口     |
| `default`      | JWT 认证 | 用户接口、需要登录的 API |
| `signature`    | 签名验签   | Open API、第三方集成 |
| `internal`     | 内部调用   | 服务间调用、内网 API   |

## 测试验证

### 1. 启动应用

```bash
cd loadup-application
mvn spring-boot:run
```

### 2. 测试公开接口（应该可以访问）

```bash
# 假设 /actuator/health 配置为 securityCode=OFF
curl http://localhost:8080/actuator/health

# 应该返回 200
```

### 3. 测试受保护接口（需要 JWT）

```bash
# 假设 /api/v1/users/profile 配置为 securityCode=default

# 不带 Token - 应该返回 401
curl http://localhost:8080/api/v1/users/profile

# 带 Token - 应该返回 200
curl -H "Authorization: Bearer <your-jwt-token>" \
  http://localhost:8080/api/v1/users/profile
```

## 后续工作

### 1. 配置路由（高优先级）

需要在 Gateway 配置中添加路由，指定每个路径的 `securityCode`。

**文件位置**:

- FILE 模式: `loadup-application/src/main/resources/gateway-config/`
- DATABASE 模式: `t_gateway_route` 表

### 2. 示例路由配置

创建文件: `gateway-config/upms-routes.json`

```json
[
  {
    "routeId": "auth-login",
    "path": "/api/v1/auth/login",
    "method": "POST",
    "securityCode": "OFF",
    "proxyType": "bean",
    "targetBean": "authenticationController",
    "targetMethod": "login"
  },
  {
    "routeId": "user-profile",
    "path": "/api/v1/users/profile",
    "method": "GET",
    "securityCode": "default",
    "proxyType": "bean",
    "targetBean": "userController",
    "targetMethod": "getProfile"
  }
]
```

### 3. 添加 @PreAuthorize 注解

在 UPMS Service 中添加权限注解：

```java
@Service
public class UserService {
    
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(String userId) {
        // 只有 ADMIN 可以删除
    }
    
    @PreAuthorize("hasRole('ADMIN') or #userId == principal.userId")
    public void updateUser(String userId, UserDTO dto) {
        // ADMIN 可以修改任何用户，普通用户只能改自己
    }
}
```

## 相关文档

- [GATEWAY_AUTH_DELIVERY.md](../GATEWAY_AUTH_DELIVERY.md) - Gateway 认证实施总交付
- [loadup-gateway-core/SECURITY.md](../loadup-gateway/loadup-gateway-core/SECURITY.md) - 认证策略详细文档
- [loadup-components-security/README.md](../loadup-components/loadup-components-security/README.md) - Security 组件使用文档

## 总结

**修复的核心**:

- ✅ Spring Security Filter Chain 配置为 `permitAll()`（放行所有请求）
- ✅ 认证由 Gateway `SecurityAction` 完成（根据 `securityCode`）
- ✅ 授权由 `@PreAuthorize` 完成（基于 SecurityContext）

**不再使用**:

- ❌ `loadup.security.ignore-urls` 配置
- ❌ Spring Security Filter Chain 的认证功能

**现在使用**:

- ✅ `RouteConfig.securityCode` 控制认证策略
- ✅ Gateway 填充 SecurityContext
- ✅ `@PreAuthorize` 检查权限
