---
id: commons-dto
title: LoadUp Commons DTO
---

# LoadUp Commons DTO

## 📋 概述

LoadUp Commons DTO 提供了框架的数据传输对象（DTO）定义。

## 🎯 功能特性

- 通用DTO基类
- 分页查询DTO
- 数据传输对象规范
- 字段验证注解

## 📦 Maven 依赖

```xml

<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-commons-dto</artifactId>
</dependency>
```

## 🚀 主要内容

### 1. 基础DTO

提供DTO基类：

```java
public class BaseDTO implements Serializable {
    private Long          id;
    private LocalDateTime createTime;
    private LocalDateTime updateTime;
    // ...
}
```

### 2. 分页DTO

分页查询参数：

```java
public class PageDTO {
    private Integer page = 1;
    private Integer size = 10;
    private String  sortField;
    private String  sortOrder;
    // ...
}
```

### 3. 查询DTO

通用查询参数：

```java
public class QueryDTO extends PageDTO {
    private String              keyword;
    private Map<String, Object> filters;
    // ...
}
```

## 📝 使用示例

```java
// 分页查询
PageDTO pageDTO = new PageDTO();
pageDTO.

setPage(1);
pageDTO.

setSize(20);

// 查询DTO
QueryDTO queryDTO = new QueryDTO();
queryDTO.

setKeyword("search");
queryDTO.

addFilter("status",1);
```

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
