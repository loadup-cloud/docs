---
id: components-extension
title: LoadUp Extension Framework
---

> 来源: `loadup-components/loadup-components-extension/README.md`

# LoadUp Extension Framework

[![Maven](https://img.shields.io/badge/Maven-1.0.0--SNAPSHOT-blue.svg)]()
[![Java](https://img.shields.io/badge/Java-17%2B-orange.svg)]()
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-green.svg)]()

一个基于 Spring Boot 3 的企业级扩展点框架，提供灵活的业务扩展能力。支持多维度场景匹配（bizCode、useCase、scenario）和 SPI 机制。

## 核心特性

- 🎯 **多维度场景匹配** - 支持 bizCode、useCase、scenario 三级场景划分
- 🔌 **灵活执行模式** - 单一匹配、批量执行、结果收集
- 🚀 **Spring Boot 3 原生支持** - 标准的 SPI 机制和自动配置
- ⚡ **优先级控制** - 支持扩展点执行顺序控制
- 📊 **降级匹配** - 智能的4级降级匹配策略
- 🔄 **两种注册方式** - 注解方式和 SPI 方式

---

## 快速开始

### 1. 添加依赖

```xml

<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-extension</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

### 2. 定义扩展点接口

```java
public interface PaymentService extends IExtensionPoint {
    PaymentResult pay(PaymentRequest request);
}
```

### 3. 实现扩展点（注解方式）

```java

@Extension(
        bizCode = "alipay",
        useCase = "online",
        scenario = "scan",
        priority = 1
)
@Component
public class AlipayService implements PaymentService {
    @Override
    public PaymentResult pay(PaymentRequest request) {
        // 支付宝支付实现
        return new PaymentResult();
    }
}
```

### 4. 使用扩展点

```java

@Service
public class OrderService {

    @Autowired
    private ExtensionExecutor extensionExecutor;

    public void processPayment(String channel) {
        BizScenario scenario = BizScenario.valueOf(channel, "online", "scan");

        PaymentResult result = extensionExecutor.execute(
                PaymentService.class,
                scenario,
                service -> service.pay(request)
        );
    }
}
```

---

## 核心API

### 6个核心方法

```java
// 1. 执行单个扩展点（有返回值）
<E, R> R execute(Class<E> class, BizScenario scenario, Function<E, R> action)

// 2. 执行单个扩展点（无返回值）
<E> void run(Class<E> class, BizScenario scenario, Consumer<E> action)

// 3. 执行指定场景的所有扩展点
<E> void executeAll(Class<E> class, BizScenario scenario, Consumer<E> action)

// 4. 执行指定bizCode的所有扩展点
<E> void executeAll(Class<E> class, String bizCode, Consumer<E> action)

// 5. 收集指定场景的所有执行结果
<E, R> List<R> collect(Class<E> class, BizScenario scenario, Function<E, R> action)

// 6. 收集指定bizCode的所有执行结果
<E, R> List<R> collect(Class<E> class, String bizCode, Function<E, R> action)
```

---

## 使用场景示例

### 场景1: 支付渠道选择

```java
public interface PaymentService extends IExtensionPoint {
    PaymentResult pay(PaymentRequest request);
}

// 支付宝实现
@Extension(bizCode = "alipay", useCase = "online", scenario = "scan")
@Component
public class AlipayService implements PaymentService {
    public PaymentResult pay(PaymentRequest request) {
        // 支付宝扫码支付
    }
}

// 微信实现
@Extension(bizCode = "wechat", useCase = "online", scenario = "scan")
@Component
public class WechatPayService implements PaymentService {
    public PaymentResult pay(PaymentRequest request) {
        // 微信扫码支付
    }
}

// 使用
public void processPayment(String channel) {
    BizScenario scenario = BizScenario.valueOf(channel, "online", "scan");
    PaymentResult result = extensionExecutor.execute(
            PaymentService.class,
            scenario,
            service -> service.pay(request)
    );
}
```

### 场景2: 消息多渠道推送

```java
public interface MessagePushService extends IExtensionPoint {
    boolean push(Message message);
}

// 实现多个推送渠道
@Extension(bizCode = "sms", priority = 1)
@Component
public class SmsPushService implements MessagePushService {}

@Extension(bizCode = "email", priority = 2)
@Component
public class EmailPushService implements MessagePushService {}

@Extension(bizCode = "app", priority = 3)
@Component
public class AppPushService implements MessagePushService {}

// 群发所有渠道
public void broadcastMessage(Message message) {
    for (String channel : Arrays.asList("sms", "email", "app")) {
        extensionExecutor.executeAll(
                MessagePushService.class,
                channel,
                service -> service.push(message)
        );
    }
}
```

### 场景3: 订单处理

```java
public interface OrderProcessService extends IExtensionPoint {
    void processOrder(Order order);

    boolean validateOrder(Order order);
}

// 国内订单
@Extension(bizCode = "domestic", useCase = "standard", scenario = "normal", priority = 1)
@Component
public class DomesticOrderProcessor implements OrderProcessService {
    public void processOrder(Order order) {
        // 国内订单处理逻辑
    }
}

// 国际订单
@Extension(bizCode = "international", useCase = "standard", scenario = "normal", priority = 1)
@Component
public class InternationalOrderProcessor implements OrderProcessService {
    public void processOrder(Order order) {
        // 国际订单处理逻辑
    }
}

// 使用
public void processSingleOrder(Order order) {
    BizScenario scenario = BizScenario.builder()
            .bizCode(order.isInternational() ? "international" : "domestic")
            .useCase("standard")
            .scenario("normal")
            .build();

    extensionExecutor.run(
            OrderProcessService.class,
            scenario,
            service -> service.processOrder(order)
    );
}
```

### 场景4: 责任链模式

```java
// 定义审批服务
public interface ApprovalService extends IExtensionPoint {
    ApprovalResult approve(ApprovalRequest request);
}

// 多级审批实现
@Extension(bizCode = "approval", useCase = "expense", scenario = "standard", priority = 1)
@Component
public class ManagerApprovalService implements ApprovalService {}

@Extension(bizCode = "approval", useCase = "expense", scenario = "standard", priority = 2)
@Component
public class DirectorApprovalService implements ApprovalService {}

// 执行责任链
public void processApproval(ApprovalRequest request) {
    BizScenario scenario = BizScenario.valueOf("approval", "expense", "standard");

    List<ApprovalResult> results = extensionExecutor.collect(
            ApprovalService.class,
            scenario,
            service -> service.approve(request)
    );

    boolean allApproved = results.stream().allMatch(ApprovalResult::isApproved);
}
```

---

## SPI 方式实现

### 实现 ExtensionProvider

```java

@Component
public class CustomPaymentProvider implements ExtensionProvider {

    @Override
    public Class<? extends IExtensionPoint> getExtensionType() {
        return PaymentService.class;
    }

    @Override
    public String getBizCode() {
        return "custom";
    }

    @Override
    public String getUseCase() {
        return "online";
    }

    @Override
    public String getScenario() {
        return "special";
    }

    @Override
    public int getPriority() {
        return 10;
    }

    @Override
    public IExtensionPoint createExtension() {
        return new CustomPaymentService();
    }

    private static class CustomPaymentService implements PaymentService {
        public PaymentResult pay(PaymentRequest request) {
            // 自定义支付实现
        }
    }
}
```

---

## BizScenario 构建

### Builder 模式

```java
BizScenario scenario = BizScenario.builder()
        .bizCode("payment")
        .useCase("online")
        .scenario("scan")
        .build();
```

### valueOf 静态方法

```java
// 只指定 bizCode
BizScenario.valueOf("payment");

// 指定 bizCode 和 useCase
BizScenario.

valueOf("payment","online");

// 指定完整场景
BizScenario.

valueOf("payment","online","scan");
```

---

## 降级匹配策略

框架提供智能的4级降级匹配：

```
请求场景: bizCode=alipay, useCase=mobile, scenario=special

匹配顺序:
1. alipay.mobile.special (精确匹配)
2. alipay.mobile.*       (忽略 scenario)
3. alipay.*.*            (只匹配 bizCode)
4. alipay.default.default (默认实现)
```

---

## 高级特性

### 1. 优先级控制

```java

@Extension(bizCode = "payment", priority = 1)  // 优先级高
public class HighPriorityPayment implements PaymentService {}

@Extension(bizCode = "payment", priority = 10) // 优先级低
public class LowPriorityPayment implements PaymentService {}
```

### 2. 动态场景选择

```java
public void smartProcessing(Order order) {
    String bizCode = order.isInternational() ? "international" : "domestic";
    String useCase = order.isVip() ? "vip" : "standard";
    String scenario = order.isUrgent() ? "urgent" : "normal";

    BizScenario bizScenario = BizScenario.valueOf(bizCode, useCase, scenario);

    extensionExecutor.run(
            OrderProcessService.class,
            bizScenario,
            service -> service.processOrder(order)
    );
}
```

### 3. 降级容错

```java
public void processWithFallback(Order order) {
    try {
        // 尝试特定场景
        BizScenario scenario = BizScenario.valueOf("domestic", "promotion", "flash-sale");
        extensionExecutor.run(OrderProcessService.class, scenario,
                service -> service.processOrder(order));
    } catch (ExtensionExecutor.ExtensionNotFoundException e) {
        // 降级到标准场景
        BizScenario fallback = BizScenario.valueOf("domestic", "standard", "normal");
        extensionExecutor.run(OrderProcessService.class, fallback,
                service -> service.processOrder(order));
    }
}
```

---

## 最佳实践

### 1. 合理划分场景维度

```java
// ✅ 推荐：清晰的三级划分
bizCode:"payment"      // 业务大类
useCase:"online"       // 使用场景
scenario:"scan"        // 细分场景

// ❌ 不推荐：过度细分
bizCode:"payment.alipay.scan.barcode.vip"
```

### 2. 提供默认实现

```java

@Extension(
        bizCode = "payment",
        useCase = "default",
        scenario = "default"
)
@Component
public class DefaultPaymentService implements PaymentService {
    // 默认实现，避免找不到扩展点
}
```

### 3. 使用明确的方法

根据业务需求选择合适的执行方法：

- 只需要一个结果 → `execute()`
- 执行无返回值操作 → `run()`
- 需要通知所有实现 → `executeAll()`
- 需要收集多个结果 → `collect()`

### 4. 开启日志

```yaml
logging:
  level:
    io.github.loadup.components.extension: DEBUG
```

---

## Spring Boot 3 集成

### 自动配置

框架使用 Spring Boot 3 的自动配置机制，无需手动配置：

```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
└── io.github.loadup.components.extension.config.ExtensionAutoConfiguration
```

### 覆盖默认配置

如需自定义，只需提供自己的 Bean：

```java

@Configuration
public class CustomExtensionConfig {

    @Bean
    public ExtensionRegistry customExtensionRegistry() {
        // 自定义实现
        return new CustomExtensionRegistry();
    }
}
```

---

## 常见问题

### Q1: 如何调试扩展点匹配过程？

**A:** 开启 DEBUG 日志级别：

```properties
logging.level.io.github.loadup.components.extension=DEBUG
```

### Q2: 多个扩展点匹配时如何选择？

**A:**

- 对于 `execute()`: 使用 priority 最小的（优先级最高）
- 对于 `executeAll()`: 按 priority 升序执行所有匹配的扩展点

### Q3: 扩展点找不到会怎样？

**A:** 抛出 `ExtensionNotFoundException` 异常，可以通过提供默认实现避免。

### Q4: 如何在运行时动态添加扩展点？

**A:** 实现 `ExtensionProvider` 并注册为 Spring Bean，框架会自动发现。

---

## 测试覆盖率

当前测试覆盖率：

- **类覆盖率**: 100%
- **方法覆盖率**: 65.3%
- **行覆盖率**: 63.5%
- **指令覆盖率**: 54.0%

核心功能已经过充分测试，可以安全使用。

---

## 技术栈

- Java 21+
- Spring Boot 4.x
- Lombok
- SLF4J

---

## 许可证

GNU General Public License v3.0 (GPL-3.0)

Copyright (C) 2025 LoadUp Framework

详见 [LICENSE](../../LICENSE) 文件。

---

## 参与贡献

欢迎提交 Issue 和 Pull Request！

---

## 联系方式

- 项目地址: [GitHub](https://github.com/loadup-cloud/loadup-framework)
- 问题反馈: [Issues](https://github.com/loadup-cloud/loadup-framework/issues)
