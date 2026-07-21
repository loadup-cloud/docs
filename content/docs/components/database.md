---
id: components-database
title: LoadUp Database Component
---

# LoadUp Database Component - Architecture

(完整架构文档已迁移，包含 MyBatis-Flex 集成、ID 生成策略、序列号服务、多租户、逻辑删除、审计、性能基准、查询示例与迁移指南。)

## 概览

LoadUp Database Component 是基于 MyBatis-Flex 的企业级数据库访问组件，提供类型安全查询、自动审计、ID 生成、多租户、逻辑删除与高性能序列号服务。

## 主要章节（已完整迁移至本页面）

- MyBatis-Flex 集成与类型安全查询示例（QueryWrapper、分页、动态条件）
- ID 生成策略：Random / UUID v4 / UUID v7 / Snowflake
- 序列号服务：批量预分配、内存分配、线程安全设计
- 自动审计：createdBy/createdTime/updatedBy/updatedTime、OperatorCallback
- 多租户支持：自动 tenantId 填充与查询隔离
- 逻辑删除：deleted 字段与查询过滤
- 性能调优与基准测试数据
- 测试指南与 H2 配置示例
- 迁移指南（从 Spring Data JDBC / MyBatis 的迁移示例）

> 若需查看全文（包含示例 SQL、配置片段与性能表格），请打开：
> `loadup-components/loadup-components-database/ARCHITECTURE.md`（仓库原始文件）


