---
title: "Gotone 多渠道通知"
---
Gotone 按 `serviceCode` 展开 SMS、Email、Push、站内信和 Webhook。一个编码可同时启用多个渠道。系统根据渠道、模板、设备平台及账号优先级自动选择实际供应商。

| Maven 坐标后缀 | 用途 |
| --- | --- |
| `loadup-modules-gotone-client` | 业务调用方依赖。提供 `GotoneFacade`、投递查询和配置管理契约。 |
| `loadup-modules-gotone-core` | 配置、模板、持久化受理、Task、渠道适配器和数据库迁移。 |
| `loadup-modules-gotone-web` | HTTP 接入，传递依赖 core 和 client。 |

坐标由 `loadup-modules-bom` 管理，groupId 为 `io.github.loadup-cloud`。现有宿主使用 PostgreSQL，宿主还须提供 Framework Task。Gotone 建表 DDL 已在 MySQL 8.0、H2 2.4 验证可创建 11 张表，但其他引擎的完整接入尚未验证。选择 Flyway 位置 `classpath:db/migration/modules/gotone`，并准备 Framework 的 Task 和序列分配表。迁移只建表和索引，不插入示例服务、账号或渠道数据。Boot 和 Pay NotificationCenter 同时装配 Gotone 与 Inbox。

## 配置与投递

超级管理员在「通知管理 → 服务编码」选择渠道、声明参数并填写各渠道模板。表达式 `${orderNo}` 引用同名参数。草稿只对创建人可见，发布后整版生效。历史模板可以恢复并发布成新版本。邮件 HTML 中的变量只用于文字内容，Webhook JSON 变量放在字符串值中。Inbox 跳转地址须为站内路径。

审批提醒使用 `approval.pending` 服务编码。首次启用前，管理员须声明 `actionUrl` 并发布一个 `IN_APP` 模板，例如 `category=APPROVAL`、`title=待办审批`、`body=你有一项审批待处理`、`actionPath=${actionUrl}`。迁移不会预置此服务或模板。

「系统发送配置」维护独立的供应商账号、优先级、连接超时、凭证引用和短信备案映射。账号运行时从 `kms:<名称>` 或 `env:GOTONE_SECRET_<名称>` 读取秘密。

| 渠道 | 内置发送能力与必要配置 |
| --- | --- |
| SMS | `aliyun` 需要 AccessKey ID、密钥引用及服务编码版本对应的备案模板，面向中国大陆 `+86` 手机。 `twilio` 需要 Account SID、发送号码和 Auth Token 或 API Key Secret 引用。 |
| Email | `smtp` 使用宿主的 `spring-boot-starter-mail`，每个系统账号独立配置主机、端口、用户名、发件地址和 STARTTLS 或 SMTPS。 |
| Push | `fcm` 使用 Firebase 项目 ID、应用标识和平台，凭证引用指向服务账号 JSON。 |
| Inbox | 使用 Inbox 模块在本地入箱，提交时提供收件人用户 ID。 |
| Webhook | `https` 使用已登记的目的地和可选 HMAC 签名凭证。宿主设置 `loadup.modules.gotone.webhook.allowed-hosts` 为逗号分隔的受信域名。带令牌的 URL 使用目的地凭证引用。 |

按需通过 core 的 `GotoneProvider` 注册额外适配器。阿里云、Twilio 与 FCM 的传输协议见 [阿里云 RPC 文档](https://www.alibabacloud.com/help/en/sdk/product-overview/rpc-mechanism)、[Twilio Messages 文档](https://www.twilio.com/docs/messaging/api/message-resource) 和 [FCM HTTP v1 文档](https://firebase.google.com/docs/cloud-messaging/send/v1-api)。

## 调用

跨模块从业务提交后的后台任务调用 `GotoneFacade.submit(source, senderId, command)`。服务间使用 `POST /api/internal/v1/gotone/submit`，要求 SERVICE JWT 及 `gotone:submit` scope。使用 `GET /api/internal/v1/gotone/requests/{id}` 查询本来源的投递，要求 `gotone:status` scope。

```json
{
  "serviceCode": "servicecodeA",
  "requestKey": "order-paid-7391",
  "parameters": { "orderNo": "7391" },
  "recipientsByChannel": {
    "SMS": [{ "address": "+8613800000000" }],
    "EMAIL": [{ "address": "buyer@example.org" }],
    "PUSH": [{ "address": "fcm-registration-token", "platform": "ANDROID", "appId": "buyer-app" }],
    "IN_APP": [{ "address": "user-42" }]
  }
}
```

响应返回 `requestId`、配置版本、受理时间和目标数。相同来源与请求键重试时返回原受理结果，内容冲突会被拒绝。Webhook 目标由发布版本提供。管理端可以发布、查看各目标的尝试、登记供应商核对结果，或对已确认失败且仍在有效期内的目标登记重试。

宿主启用 Gotone 时需提供 Inbox 接收实现。Gotone 将 `deliveryId` 传给 `InboxReceiver` 作为幂等键，入箱与标记 `SENT` 共用一个本地事务。收件箱页面调用 `/api/inbox`。Gotone 模块与 HTTP 开关分别为 `loadup.modules.gotone.enabled` 和 `loadup.modules.gotone.web.enabled`。 [Inbox 接入说明](../inbox/)
