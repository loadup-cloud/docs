# LoadUp Testify - 自动化测试框架

## 概述

Testify 是一款基于 Spring Boot 生态构建的 **声明式、数据驱动** 集成测试框架。它通过 YAML 定义测试逻辑，利用 AOP 拦截技术实现动态
Mock，并提供了一套支持异步重试和复杂逻辑比对的断言系统。

## 核心特性

- ✅ **YAML 驱动**: 测试用例用 YAML 编写，简洁直观
- ✅ **数据驱动**: 基于 TestNG DataProvider，支持参数化测试
- ✅ **动态 Mock**: 利用 Spring AOP 实现智能 Mock，无需修改业务代码
- ✅ **强大断言**: 支持 Response、Database、Exception 多维度断言
- ✅ **变量引擎**: 集成 SpEL 和 Java Faker，动态生成测试数据
- ✅ **异步重试**: 支持数据库断言的异步重试机制

## 技术栈

| 技术                  | 说明                  |
|---------------------|---------------------|
| Java 17+            | 编程语言                |
| Spring Boot         | IoC/AOP 基础          |
| TestNG              | 测试驱动 (DataProvider) |
| Spring AOP          | Mock 拦截             |
| SpEL                | 变量引擎                |
| Java Faker          | 数据模拟                |
| Jackson             | JSON 处理             |
| JsonPath            | 路径提取                |
| JSONAssert          | 对象比对                |
| Spring JdbcTemplate | 数据库操作               |

## 架构设计

### 核心流程

```
┌─────────────────────────────────────────────┐
│  1. YAML 解析 (TestifyDataProvider)          │
│     ↓                                        │
│  2. 变量预解析 (VariableEngine)              │
│     ↓                                        │
│  3. Mock 激活 (MockInterceptor AOP)          │
│     ↓                                        │
│  4. 闭包执行 (runTest Lambda)                │
│     ↓                                        │
│  5. 自动断言 (Response/Database/Exception)   │
└─────────────────────────────────────────────┘
```

### 模块结构

```
loadup-testify/
├── loadup-testify-core/              # 核心引擎
│   └── TestifyBase                   # 测试基类
├── loadup-testify-assert-engine/     # 断言引擎
│   ├── ResponseAssert                # 响应断言
│   ├── DatabaseAssert                # 数据库断言
│   └── ExceptionAssert               # 异常断言
├── loadup-testify-mock-engine/       # Mock 引擎
│   └── MockInterceptor               # AOP 拦截器
├── loadup-testify-data-engine/       # 数据引擎
│   └── VariableEngine                # 变量解析
├── loadup-testify-infra-container/   # 基础设施
│   └── TestcontainersSupport         # 容器支持
└── loadup-testify-spring-boot-starter/ # Spring Boot 集成
```

## 快速开始

### 1. 添加依赖

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-testify-spring-boot-starter</artifactId>
    <version>0.0.2-SNAPSHOT</version>
    <scope>test</scope>
</dependency>
```

### 2. 创建测试基类

```java
@SpringBootTest
public class UserServiceTest extends TestifyBase {
    
    @Autowired
    private UserService userService;
    
    @Test(dataProvider = "testifyDataProvider")
    public void testCreateUser(String userId, String username) {
        runTest(() -> {
            return userService.createUser(userId, username);
        });
    }
}
```

### 3. 编写 YAML 测试用例

在 `src/test/resources/testify/` 目录下创建 `UserServiceTest.yml`:

```yaml
testCreateUser:
  - case_name: "创建用户成功"
    variables:
      userId: "${time.now()}"
      userName: "${faker.name().firstName()}"
      email: "test@example.com"
    
    input:
      - "${userId}"
      - "${userName}"
    
    setup:
      clean_sql: "DELETE FROM t_user WHERE user_id = '${userId}'"
    
    mocks:
      - bean: "emailService"
        method: "sendWelcomeEmail"
        args: ["${email}"]
        thenReturn:
          success: true
    
    expect:
      response:
        userId: "${userId}"
        username: "${userName}"
        status: "ACTIVE"
      
      database:
        query: "SELECT * FROM t_user WHERE user_id = '${userId}'"
        expect:
          - username: "${userName}"
            email: "${email}"
```

### 4. 运行测试

```bash
mvn test -Dtest=UserServiceTest#testCreateUser
```

## YAML 配置详解

### 变量配置 (variables)

支持动态函数和静态值：

```yaml
variables:
  userId: "${time.now()}"              # 时间戳
  userName: "${faker.name().firstName()}" # Faker 生成
  email: "test@example.com"            # 静态值
  randomId: "${random.uuid()}"         # 随机 UUID
```

### 参数输入 (input)

测试方法的参数列表（顺序必须一致）：

```yaml
input:
  - "${userId}"      # 第一个参数
  - "${userName}"    # 第二个参数
  - "ACTIVE"         # 第三个参数（静态值）
```

### 环境准备 (setup)

测试前的数据准备和清理：

```yaml
setup:
  clean_sql: |
    DELETE FROM t_user WHERE user_id = '${userId}';
    DELETE FROM t_user_profile WHERE user_id = '${userId}';
  
  insert_sql: |
    INSERT INTO t_department (id, name) VALUES ('dept_001', 'IT部门');
```

### 智能 Mock (mocks)

基于 AOP 的动态 Mock：

```yaml
mocks:
  - bean: "orderService"           # Bean 名称
    method: "createOrder"          # 方法名
    args: ["${userName}"]          # 参数匹配
    thenReturn:                    # 返回值
      orderId: "123456"
      status: "SUCCESS"
    delay: 500                     # 延时（毫秒）
  
  - bean: "paymentService"
    method: "pay"
    args: ["any"]                  # 匹配任意参数
    thenThrow: "com.example.PaymentException"  # 抛出异常
```

### 断言配置 (expect)

#### Response 响应断言

```yaml
expect:
  response:
    userId: "${userId}"
    username: "${userName}"
    status: "ACTIVE"
    "profile.email": "${email}"    # JsonPath 表达式
```

#### Database 数据库断言

```yaml
expect:
  database:
    query: "SELECT * FROM t_user WHERE user_id = '${userId}'"
    expect:
      - username: "${userName}"
        email: "${email}"
        status: 1
    retry:
      max_attempts: 3
      delay: 1000                  # 每次重试间隔（毫秒）
```

#### Exception 异常断言

```yaml
expect:
  exception:
    type: "java.lang.IllegalArgumentException"
    message: "User ID cannot be empty"  # 支持模糊匹配
```

## 高级特性

### 1. JsonPath 表达式

支持复杂对象断言：

```yaml
expect:
  response:
    "user.profile.address.city": "Beijing"
    "orders[0].amount": 100.0
    "tags[*]": ["VIP", "Active"]
```

### 2. 数据库异步重试

适用于异步操作场景：

```yaml
expect:
  database:
    query: "SELECT status FROM t_order WHERE order_id = '${orderId}'"
    expect:
      - status: "PAID"
    retry:
      max_attempts: 5
      delay: 2000
```

### 3. 多 Mock 组合

单个测试用例可配置多个 Mock：

```yaml
mocks:
  - bean: "inventoryService"
    method: "checkStock"
    thenReturn: { available: true }
  
  - bean: "paymentService"
    method: "pay"
    thenReturn: { success: true }
  
  - bean: "notificationService"
    method: "notify"
    thenReturn: { sent: true }
```

### 4. 变量引用

变量可在所有配置中引用：

```yaml
variables:
  userId: "user_001"
  orderId: "order_${time.now()}"

input:
  - "${userId}"

setup:
  clean_sql: "DELETE FROM t_order WHERE user_id = '${userId}'"

mocks:
  - bean: "orderService"
    args: ["${userId}"]

expect:
  response:
    userId: "${userId}"
    orderId: "${orderId}"
```

## 完整示例

```yaml
testCompleteScenario:
  - case_name: "完整业务流程测试"
    variables:
      userId: "${time.now()}"
      userName: "${faker.name().fullName()}"
      email: "${faker.internet().emailAddress()}"
      orderId: "order_${time.now()}"
      amount: 99.99
    
    input:
      - "${userId}"
      - "${userName}"
      - "${email}"
      - ${amount}
    
    setup:
      clean_sql: |
        DELETE FROM t_user WHERE user_id = '${userId}';
        DELETE FROM t_order WHERE order_id = '${orderId}';
    
    mocks:
      - bean: "inventoryService"
        method: "checkStock"
        args: ["PRODUCT_001", 1]
        thenReturn:
          available: true
          stock: 100
      
      - bean: "paymentService"
        method: "createPayment"
        args: ["${orderId}", ${amount}]
        thenReturn:
          paymentId: "pay_${time.now()}"
          status: "SUCCESS"
        delay: 200
      
      - bean: "emailService"
        method: "sendOrderConfirmation"
        args: ["${email}", "${orderId}"]
        thenReturn:
          sent: true
    
    expect:
      response:
        orderId: "${orderId}"
        userId: "${userId}"
        status: "CONFIRMED"
        amount: ${amount}
        "payment.status": "SUCCESS"
      
      database:
        - query: "SELECT * FROM t_order WHERE order_id = '${orderId}'"
          expect:
            - order_id: "${orderId}"
              user_id: "${userId}"
              status: "CONFIRMED"
              amount: ${amount}
          retry:
            max_attempts: 3
            delay: 1000
        
        - query: "SELECT * FROM t_user WHERE user_id = '${userId}'"
          expect:
            - user_id: "${userId}"
              username: "${userName}"
              email: "${email}"
```

## 最佳实践

### 1. 测试隔离

每个测试用例应该独立，使用 `setup.clean_sql` 清理数据：

```yaml
setup:
  clean_sql: "DELETE FROM t_user WHERE user_id = '${userId}'"
```

### 2. 动态数据生成

使用 Faker 和时间戳生成唯一数据，避免冲突：

```yaml
variables:
  userId: "${time.now()}"
  email: "${faker.internet().emailAddress()}"
```

### 3. 合理使用 Mock

只 Mock 外部依赖和复杂逻辑，保持核心业务代码真实执行：

```yaml
mocks:
  - bean: "externalApiClient"  # Mock 外部 API
    method: "callThirdParty"
    thenReturn: { success: true }
```

### 4. 数据库断言重试

对异步操作使用重试机制：

```yaml
expect:
  database:
    query: "SELECT status FROM t_async_task WHERE id = '${taskId}'"
    expect:
      - status: "COMPLETED"
    retry:
      max_attempts: 5
      delay: 2000
```

### 5. 清晰的用例命名

```yaml
testCreateUser:
  - case_name: "创建用户成功 - 正常流程"
  - case_name: "创建用户失败 - 用户名已存在"
  - case_name: "创建用户失败 - 邮箱格式错误"
```

## 故障排查

### 常见问题

**1. Mock 不生效**

- 检查 Bean 名称是否正确
- 确认方法签名匹配
- 查看 AOP 拦截日志

**2. 变量解析失败**

- 检查变量名拼写
- 确认 SpEL 表达式语法
- 查看变量引擎日志

**3. 数据库断言失败**

- 检查 SQL 语法
- 确认数据库连接
- 增加重试次数和延时

**4. 响应断言失败**

- 打印实际响应内容
- 检查 JsonPath 表达式
- 确认数据类型匹配

## 更多资源

- [完整文档](../loadup-testify/README.md)
- [示例项目](../loadup-testify/loadup-testify-demo/)
- [API 文档](javadoc/)

---

**© 2025 LoadUp Framework. All rights reserved.**
