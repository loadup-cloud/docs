---
id: commons-dto
title: LoadUp Commons DTO
---

# LoadUp Commons API

## 📋 概述

LoadUp Commons API 提供了框架的核心API接口和基础定义。

## 🎯 功能特性

- 通用API接口定义
- 基础响应模型
- 异常体系
- 通用工具接口

## 📦 Maven 依赖

```xml

<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-commons-api</artifactId>
</dependency>
```

## 🚀 主要内容

### 1. 响应模型

提供统一的响应模型：

```java
public class Result<T> {
    private int    code;
    private String message;
    private T      data;
    // ...
}
```

### 2. 异常定义

统一的异常体系：

```java
public class BusinessException extends RuntimeException {
    private int    code;
    private String message;
    // ...
}
```

### 3. 通用接口

定义通用的接口规范。

## 📝 使用示例

```java
// 返回成功响应
return Result.success(data);

// 返回失败响应
return Result.

error("操作失败");

// 抛出业务异常
throw new

BusinessException("业务处理失败");
```

这是一个为您整理的完整 `README.md` 文件。它涵盖了我们重构后的**组件化动态绑定架构**的设计理念、核心组件说明以及详细的配置手册。

---

# Loadup Framework - Dynamic Binding Kernel

## 1. 简介

Loadup 动态绑定内核是一个基于 Spring Boot 的轻量级组件微内核架构。它旨在解决分布式系统中**
多策略组件（DFS、Cache、RPC、NoSQL）**的动态装配问题。

通过该内核，开发者可以实现：

* **多实例隔离**：同一套驱动（如 S3），通过不同配置生成多个业务隔离的实例。
* **运行时动态路由**：根据 YAML 配置自动选择驱动（Binder）和业务实现（Binding）。
* **零侵入注入**：使用 `@BindingClient` 注解像 `@Autowired` 一样自动注入动态生成的对象。

---

## 2. 核心概念

* **Binder (驱动)**：底层执行单元（如 `S3DfsBinder`, `RedisCacheBinder`）。
* **Binding (业务绑定)**：对外的业务逻辑封装（如 `OrderDfsBinding`），内部组合了一个或多个 Binder。
* **BindingMetadata (元数据)**：定义了 `Binding`、`Binder` 及其对应配置类 `Cfg` 的映射关系。
* **ManagerSupport (内核)**：负责配置解析、泛型捕获、实例生命周期管理及缓存。

---

## 3. 快速开始

### 第一步：定义配置 (YAML)

在配置文件中定义业务标识（bizTag）及其关联的驱动类型。

```yaml
loadup:
  dfs:
    default-binder: local  # 默认驱动
    bindings:
      user-avatar: # 业务标识 (bizTag)
        binder-type: s3
        bucket: user-data
      order-pdf:
        # 未指定 binder-type，将使用默认的 local
        root-path: /data/orders
    binders:
      s3:
        endpoint: http://oss-cn-hangzhou.aliyuncs.com
        access-key: AK_123
      local:
        root-path: /tmp/storage

```

### 第二步：业务使用

在任何 Spring Bean 中直接注入即可，无需关心它是如何实例化的。

```java

@Service
public class UserService {

    @BindingClient("user-avatar")
    private DfsBinding avatarStorage;

    public void uploadAvatar(String userId, byte[] data) {
        avatarStorage.upload(userId + ".jpg", data);
    }
}

```

---

## 4. 架构实现指南

### 4.1 定义元数据注册 (Starter 层)

在您的 `AutoConfiguration` 中向 Manager 注册实现类。

```java

@PostConstruct
public void register() {
    dfsBindingManager.register("s3", new BindingMetadata<>(
            S3BindingCfg.class,   // 业务配置类
            S3BinderCfg.class,    // 驱动配置类
            S3DfsBinder.class,    // 驱动实现类
            ctx -> new S3DfsBinding() // 工厂方法
    ));
}

```

### 4.2 实现业务逻辑 (API 层实现)

继承 `AbstractBinding` 获得自动装配能力。

```java
public class S3DfsBinding extends AbstractDfsBinding<S3DfsBinder, S3BindingCfg> {
    @Override
    protected void afterInit() {
        // 此时 this.cfg 和 this.binders 已由内核自动填充
        log.info("S3 Binding [{}] initialized", name);
    }
}

```

---

## 5. 核心技术点说明

### 泛型捕获 (Generic Capture)

内核通过 `captureAndCreate` 私有泛型方法，解决了 Java 泛型在运行时类型擦除导致的 `Object cannot be converted`
问题，实现了全过程的类型安全。

### 自动装配工厂 (Spring Autowire Support)

内核使用 `AutowireCapableBeanFactory.createBean()` 实例化 Binder，这意味着您的驱动类虽然不是 Spring Bean，但依然可以使用
`@Autowired` 注入
Spring 容器中的其他组件（如 `RestTemplate` 或 `DataSource`）。

### 类型推断 (Type Inference)

`getBinding` 方法支持返回类型推断，调用方无需强制转换：

```java
// 自动推断类型
S3DfsBinding binding = manager.getBinding("user-avatar");

```

---

## 6. 模块依赖结构

* `loadup-framework-api`: 包含内核基类、注解、上下文定义。
* `loadup-component-dfs-api`: 包含 DFS 相关的抽象定义。
* `loadup-component-dfs-s3`: S3 驱动的具体实现。
* `loadup-dfs-spring-boot-starter`: 自动配置与注册中心。

---

## 7. 注意事项

1. **配置类规范**：配置类（Cfg）必须提供无参构造函数，并符合 JavaBean 规范。
2. **Binder 作用域**：内核创建的 Binder 实例在 Binding 维度是唯一的，请勿手动将其标注为 `@Component`。
3. **缓存机制**：Manager 内部持有缓存，多次获取同一个 `bizTag` 会返回同一个单例 Binding 对象。

---

## 📄 许可证

GNU General Public License v3.0 (GPL-3.0)

```
Copyright (C) 2025 LoadUp Framework

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.
```

详见 [LICENSE](../LICENSE) 文件。

---

**最后更新**: 2025-12-30

