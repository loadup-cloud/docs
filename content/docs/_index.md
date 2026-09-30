---
title: "文档"
weight: 1
---

LoadUp 是供 Spring Boot 应用按需引入的框架/SDK。使用 `loadup-dependencies` BOM 管理版本，
再选择需要的技术组件或通用业务模块。`loadup-application` 用于本地集成验证。

## 按任务查找

| 任务 | 页面 |
|---|---|
| 引入 BOM、管理版本 | [依赖管理](dependencies/) |
| 选择缓存、数据库、认证等能力 | [技术组件](components/) |
| 接入用户权限管理 | [业务模块](modules/) |
| 使用基础 DTO、日志与链路追踪 | [通用基础](commons/) |
| 编写集成测试 | [Testify](testing/) |
| 运行本地验证应用 | [Application](application/) |

每个能力的页面先介绍用途、接入与配置，再说明内部设计。技术子模块可从所属能力页继续查看。
