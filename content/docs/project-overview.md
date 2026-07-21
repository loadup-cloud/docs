# 项目概览（LoadUp Framework）

本页面基于根 `README.md` 汇总，提供项目的概览、技术栈与快速开始要点，方便在 Docusaurus 中展示。

## 项目简介

LoadUp Framework 是一个基于 Spring Boot 的微服务开发框架，提供了一系列可复用的组件和最佳实践，旨在帮助团队快速搭建企业级应用。

## 关键模块（高层）

- `loadup-dependencies` - 依赖与 BOM 管理
- `loadup-commons` - 通用工具、DTO、异常、基础类
- `loadup-components` - 可复用中间件组件（缓存、DFS、调度、tracer 等）
- `loadup-modules` - 业务模块（如 system/upms 等）
- `loadup-gateway` - API 网关
- `loadup-application` - 应用启动器（admin / api 等）

## 技术栈（摘要）

- Java 21
- Spring Boot 4.1.0
- MyBatis-Flex 1.11.7
- Redis, HikariCP, Flyway
- OpenAPI (Swagger)

## 快速开始（摘录）

```bash
git clone <repository>
cd loadup-framework
mvn clean install -DskipTests
cd loadup-application/loadup-app-admin
mvn spring-boot:run
```

更多详细信息请参阅仓库根 `README.md`。
