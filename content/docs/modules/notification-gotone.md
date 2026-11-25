---
title: "Gotone 多渠道投递"
aliases:
  - /docs/components/channels/
  - /docs/components/gotone-api/
  - /docs/components/gotone-binder-email/
  - /docs/components/gotone-binder-push/
  - /docs/components/gotone-binder-sms/
  - /docs/components/gotone-binder-webhook/
  - /docs/components/gotone-engine/
  - /docs/components/gotone-store-jdbc/
  - /docs/components/gotone-test/
  - /docs/components/gotone/
---

Gotone 是 [LoadUp Modules Notification](../notification/) 的多渠道投递引擎。业务按 `serviceCode` 调用
`NotificationService`，一次请求使用该业务启用的全部渠道。站内收件箱通过 `IN_APP` 渠道接入。

## 模块结构

| 模块              | 职责                                                                                            |
|-------------------|-------------------------------------------------------------------------------------------------|
| `-api`            | `NotificationService` facade + `NotificationChannelProvider` SPI + 可选存储 SPI                 |
| `-engine`         | 纯发送引擎（零存储、零 DB）：可用性过滤、provider 降级链、resilience4j 包装、异步发送、模板渲染 |
| `-store-jdbc`     | 默认 Spring JDBC 存储实现（3 张表，PostgreSQL Flyway 迁移）                                     |
| `-binder-email`   | SMTP（Spring Mail），真实实现                                                                   |
| `-binder-sms`     | aliyun / huawei / yunpian（stub，待接厂商 SDK）                                                 |
| `-binder-push`    | fcm（stub，待接 Firebase Admin SDK）                                                            |
| `-binder-webhook` | dingtalk / wechat / feishu，JDK HttpClient 真实 HTTP 发送                                       |

## 引入

通过 `loadup-modules-bom` 管理版本。编译时依赖 `loadup-modules-notification-gotone-api`。引入
`loadup-modules-notification-core` 会自动装配引擎和收件箱渠道；只做外部投递时也可独立引入
`loadup-modules-notification-gotone-engine`。渠道适配器与 JDBC 配置存储可按需添加。

## 按 serviceCode 投递

为同一个 `serviceCode` 配置多条渠道记录，每个渠道一条。可实现 `ChannelConfigProvider`，也可引入
`loadup-modules-notification-gotone-store-jdbc` 并配置
`loadup.flyway.locations` 包含 `classpath:db/migration/modules/notification-gotone`。JDBC 路由读取
`gotone_service_channel` 中相同 `service_code` 的启用记录，按 `priority` 升序处理。例如给
`ORDER_CONFIRM` 配置 `IN_APP`、`EMAIL`、`SMS` 三条记录。每行可配置 `template_content`，
以及邮件标题等渠道参数 `channel_config`。JDBC 中将 `provider`、`fallback_providers` 留空即可自动选择。

```java
var result = notificationService.send(NotificationRequest.builder()
        .serviceCode("ORDER_CONFIRM")
        .receiversByChannel(Map.of(
                "IN_APP", List.of("user-123"),
                "EMAIL", List.of("buyer@example.com"),
                "SMS", List.of("+8613800000000")))
        .templateParams(Map.of("title", "订单确认", "body", "订单 123 已确认", "category", "ORDER"))
        .build());
```

路由配置决定发送哪些渠道，`receiversByChannel` 决定各渠道的目的地址。`receivers` 仍可作为共享默认值，仅使用
`receiversByChannel` 时，缺少地址的渠道单独报告失败。聚合结果中的 `success` 表示至少有一个渠道成功，
`channelResults` 提供逐渠道结果。也可调用 `sendAsync(request)` 将请求交给应用的任务执行器。无渠道路由时可用
`channels` 显式请求直连。

## 系统渠道能力

- **IN_APP** 使用站内用户 ID。模板参数支持 `title`、`body`、`category`，以及可选的 `senderId`、`actionUrl` 和
  `requestKey`。模板渲染结果优先作为正文，业务重试时复用 `requestKey`。
- **EMAIL** 引入 `loadup-modules-notification-gotone-binder-email` 后复用系统的 `JavaMailSender`。在部署层通过
  标准 `spring.mail.*` 配置 SMTP、身份和 TLS，或注册应用自己的发送器。邮件地址来自渠道收件人，发件人来自
  `channelConfig.from` 或 `spring.mail.username`。
- **SMS** 使用系统注册且可用的 `NotificationChannelProvider`，实现的 `getChannelType()` 返回 `SMS`。现有厂商
  binder 中的 Aliyun、Huawei、Yunpian 是占位实现。实际投递需要部署注册具备真实传输能力的 SMS Provider。

自动选择遵循 Spring Provider 排序，跳过不可用实现，并在可用实现间按顺序降级。已有路由的首选和备用供应商字段可以
继续用于有固定供应商需求的场景。

## 能力矩阵

| 能力                        | 支持                                      |
|-----------------------------|-------------------------------------------|
| ServiceCode 路由            | ✓                                        |
| 直接指定渠道（无配置时）    | ✓                                        |
| 多渠道一次发送              | ✓                                        |
| 渠道内 provider 降级链      | ✓                                        |
| 熔断 + 重试（resilience4j） | ✓                                        |
| 异步发送                    | ✓                                        |
| 模板渲染（`${var}`）        | ✓                                        |
| 发送记录（可选）            | ✓（store-jdbc）                          |
| 失败重试（跨实例）          | 消费方使用 PowerJob/Task 并保证业务幂等 |

## 后台任务与告警

跨实例补偿由消费方的原生 PowerJob Processor / Task 编排，Gotone 只负责发送。旧的 RetryTask 专用通知桥已移除；按 PowerJob
原生告警机制接入运维通知，不重复构建执行历史。

## 文档

设计决策与扩展点见 [ARCHITECTURE.md](./#architecture)。

<a id="architecture"></a>
## 设计与实现

实现边界、模块源码和适配器文档见[源码目录](https://github.com/loadup-cloud/loadup-modules/tree/main/loadup-modules-notification/loadup-modules-notification-gotone)。
