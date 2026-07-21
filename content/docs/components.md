# loadup-components 概览

`loadup-components` 包含可复用的中间件与适配器实现，供业务模块复用。

## 核心组件列表

### 基础设施组件

- [Authorization](./components/authorization.md)：轻量级方法级授权框架，基于 AOP 实现

- [Cache](./components/cache.md)：缓存抽象与实现（Caffeine / Redis 绑定器）

- [Captcha](./components/captcha.md)：验证码生成与验证

- [ConfigCenter](./components/configcenter.md)：配置中心抽象与实现（Local / Nacos / Apollo 绑定器，支持配置推送与监听）

- [Database](./components/database.md)：数据库访问与工具，包含 migration 与 schema 示例

- [Extension](./components/extension.md)：插件化扩展框架

- [Liquibase](./components/liquibase.md)：数据库版本管理与迁移

- [Testcontainers](./components/testcontainers.md)：测试容器支持

### 业务能力组件

- [DFS](./components/dfs.md)：分布式文件存储支持（local / s3 / database 绑定器）

- [GlobalUnique](./components/globalunique.md)：基于数据库唯一键的全局幂等性控制

- [Gotone](./components/gotone.md)：统一消息通知服务（邮件、短信、推送、Webhook）

- [RetryTask](./components/retrytask.md)：分布式重试任务框架

- [Scheduler](./components/scheduler.md)：任务调度（Quartz / XXL-Job / SimpleJob 等）

- [Signature](./components/signature.md)：数字签名和摘要计算组件

- [Tracer](./components/tracer.md)：分布式追踪（OpenTelemetry / Jaeger 集成）

## 如何使用

组件通常以 `starter` 或 `api` + `binder` 的形式提供：

1. 在项目中引入对应 artifact（BOM 管理下无需指定版本）。

2. 根据各组件文档中的示例，配置 application.yml 并启用 starter。

更多细节参见对应组件的详细文档。
