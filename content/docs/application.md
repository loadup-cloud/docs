# loadup-application 概览

`loadup-application` 包含可运行的应用组合（启动器），例如管理后台与对外 API 服务。

常见子模块：

- `admin`：管理后台
- `api`：对外 REST API 服务

部署建议：

- `loadup-application` 负责组合 `modules` 与 `components`，并提供运行时配置与 profiles
- 运行前请确认 `application.yml` 中的数据源、缓存与第三方服务配置正确

查看 `loadup-application` 下的具体子模块 README 获取更多信息。
