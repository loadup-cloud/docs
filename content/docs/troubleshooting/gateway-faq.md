# Gateway FAQ

## 常见问题

### 认证相关

#### Q: 为什么不在 Filter 中做认证？

**A**: Gateway 的路由是动态的，需要先匹配路由才能知道应该用哪种认证策略。Filter 在 DispatcherServlet 之前执行，此时还没有路由信息。

#### Q: 支持多种认证方式并存吗？

**A**: 支持。每个路由可以配置不同的 `securityCode`，同一个应用可以同时支持 JWT、签名、内部调用等多种方式。

#### Q: 如何实现 OAuth2？

**A**: 实现一个 `OAuth2SecurityStrategy`，在 `process` 方法中：

1. 验证 Access Token（调用授权服务器或本地验证）
2. 获取用户信息
3. 填充 SecurityContext

#### Q: JWT Token 验证的性能如何？

**A**:

- JWT 验证：~1ms（本地解析和校验）
- 签名验证：~2ms（HMAC 计算）
- 内部调用：~0.1ms（IP 判断）

#### Q: 支持 Token 缓存吗？

**A**: 可以在具体策略中实现。例如 JWT Token 可以缓存解析结果（设置短时间 TTL），签名 App Secret 可以缓存到本地。

### 路由配置

#### Q: 路由未找到（404）

**原因**: 路由配置不存在或未加载

**解决**:

```bash
# 检查路由配置文件
cat resources/gateway-config/routes.csv

# 检查日志
grep "Route loaded" logs/application.log
```

#### Q: 路由通配符如何使用？

**A**:

- `*` 匹配单层路径: `/api/users/*` 匹配 `/api/users/123`，不匹配 `/api/users/123/orders`
- `**` 匹配多层路径: `/api/users/**` 匹配 `/api/users/123/orders/456`

#### Q: 路由优先级如何确定？

**A**: 按配置顺序匹配，先配置的优先。建议将具体路径放在前面，通配符路径放在后面。

### 请求转发

#### Q: Bean 不存在（500）

**原因**: Spring Bean 名称错误或 Bean 未注册

**解决**:

```bash
# 检查 Bean 是否存在
curl http://localhost:8080/actuator/beans | grep userService

# 检查路由配置
bean://userService:getUser  # 确保 Bean 名称和方法名正确
```

#### Q: HTTP 代理超时

**原因**: 目标服务响应慢或网络问题

**解决**:

```csv
# 在路由配置中增加超时时间
path,method,target,securityCode,properties
/api/orders/**,*,http://order-service:8080,default,timeout=60000
```

#### Q: Spring Bean 代理如何传递参数？

**A**: 支持三种方式：

1. 直接参数: `public UserDTO getUser(String userId)`
2. GatewayRequest: `public UserDTO getUser(GatewayRequest request)`
3. DTO 对象: `public UserDTO createUser(CreateUserRequest request)`

### 异常处理

#### Q: 如何自定义错误响应格式？

**A**: 错误响应由 `ExceptionAction` 统一处理，返回标准 Result 格式。如需自定义，可以：

1. 实现自定义的 `ExceptionAction`
2. 在 `GatewayAutoConfiguration` 中替换默认实现

#### Q: 业务异常如何处理？

**A**: 直接在业务代码中抛出异常，`ExceptionAction` 会自动捕获并转换为统一格式：

```java
@Service
public class UserService {
    public UserDTO getUser(String userId) {
        if (!exists(userId)) {
            throw new BusinessException("用户不存在");
        }
        return userRepository.findById(userId);
    }
}
```

### 性能优化

#### Q: 如何提高转发性能？

**A**:

1. 优先使用 Spring Bean 代理（无网络开销）
2. 合理配置 HTTP 连接池
3. 启用响应缓存（可选）
4. 使用 JDK 21 虚拟线程（可选）

#### Q: 路由配置如何缓存？

**A**: 路由配置加载后自动缓存在内存中。如使用数据库存储，建议：

1. 配置合理的缓存时间
2. 提供手动刷新接口
3. 监听配置变更事件

#### Q: 模板转换影响性能吗？

**A**: 模板编译后会缓存，性能影响很小（~0.1ms）。建议：

- 只在必要时使用模板
- 避免在模板中执行复杂逻辑

### 安全

#### Q: JWT Secret 如何配置？

**A**: 推荐使用环境变量：

```yaml
loadup:
  gateway:
    security:
      secret: ${JWT_SECRET:default-secret-for-dev}
```

```bash
# 生产环境
export JWT_SECRET="your-production-secret-key"
```

#### Q: 如何防止重放攻击？

**A**:

1. JWT 认证：设置合理的过期时间
2. 签名认证：验证时间戳，拒绝超过 5 分钟的请求
3. 使用 Nonce（随机数），并缓存已使用的 Nonce

#### Q: 如何限流？

**A**: 可以实现自定义 `GatewayAction`：

```java
@Component
public class RateLimitAction implements GatewayAction {
    @Override
    public void execute(GatewayContext context, GatewayActionChain chain) {
        String userId = getUserId(context);
        if (!rateLimiter.tryAcquire(userId)) {
            throw GatewayExceptionFactory.tooManyRequests();
        }
        chain.proceed(context);
    }
}
```

### 开发调试

#### Q: 如何查看请求日志？

**A**: 配置日志级别：

```yaml
logging:
  level:
    io.github.loadup.gateway: DEBUG
    io.github.loadup.gateway.core.action: INFO
    io.github.loadup.gateway.core.security: DEBUG
```

#### Q: 如何测试认证？

**A**:

```bash
# 1. 登录获取 Token
TOKEN=$(curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"123456"}' \
  | jq -r '.data.token')

# 2. 使用 Token 访问接口
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/v1/users/profile
```

#### Q: 如何调试 Spring Bean 代理？

**A**:

1. 在 Bean 方法中打断点
2. 检查日志确认 Bean 是否被正确调用
3. 使用 `@Slf4j` 记录参数和返回值

### 部署运维

#### Q: 支持动态更新路由吗？

**A**:

- 文件配置：需要重启应用
- 数据库配置：支持动态更新，需配置刷新机制

#### Q: 如何监控 Gateway？

**A**:

1. 使用 Spring Boot Actuator
2. 添加 Metrics（Micrometer）
3. 实现自定义监控 Action

#### Q: 如何做灰度发布？

**A**:

1. 在路由配置中添加版本号或灰度标识
2. 根据请求头或用户属性选择不同的目标服务
3. 使用 `properties` 字段配置灰度比例

## 相关文档

- [Gateway 完整文档](../gateway.md)
- [项目概览](../project-overview.md)
- [快速开始](../quick-start.md)
