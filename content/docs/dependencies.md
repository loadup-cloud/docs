# loadup-dependencies（BOM）

`loadup-dependencies` 模块用于集中管理依赖版本（BOM），所有组件与子项目通过该模块进行统一版本管理。

## 当前核心版本

| 依赖                | 版本            | 说明                              |
|-------------------|---------------|---------------------------------|
| Java              | **21**        | LTS 长期支持版本                      |
| Spring Boot       | **4.1.0**     | 企业级应用框架                         |
| Spring Cloud      | **2025.1.2**  | 微服务生态                           |
| MyBatis-Flex      | **1.11.7**    | 类型安全 ORM（使用 spring-boot4-starter）|
| MySQL Connector/J | **9.1.0**     | MySQL 驱动                        |
| Flyway            | **12.6.1**    | 数据库版本管理                         |
| Lombok            | **1.18.46**   | 代码生成                            |
| MapStruct         | **1.6.3**     | 对象转换                            |
| OpenTelemetry     | **1.62.0**    | 分布式链路追踪                         |
| Testcontainers    | **2.0.5**     | 集成测试容器                          |
| Mockito           | **5.23.0**    | 单元测试 Mock                       |
| knife4j           | **4.5.0**     | OpenAPI 3 文档 UI（`knife4j-openapi3-jakarta-spring-boot-starter`）|

## 注意事项

- 不要直接修改 `loadup-dependencies` 中的版本，任何依赖升级应通过团队审查与 PR 流程。
- 若添加第三方库，需要在 `loadup-dependencies/pom.xml` 中声明并经过审批。
- 子模块引用同项目内模块时，**不得**在 `<dependency>` 中写 `<version>`，版本由 BOM 统一管理。

查看 `loadup-dependencies/pom.xml` 了解当前完整版本锁定策略。
