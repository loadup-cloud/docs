# loadup-commons 概览

`loadup-commons` 提供项目的基础类型与工具，建议优先复用 commons 中的实现以保持一致性。

子模块示例：

- `loadup-commons-api`：通用接口与 Result/DTO 定义

- `loadup-commons-dto`：常用 DTO 定义

- `loadup-commons-util`：工具类（DateUtils、StringUtils、JsonUtils 等）

常用工具类位置（示例）：

- 日期处理：`io.github.loadup.commons.utils.DateUtils`

- 字符串处理：`io.github.loadup.commons.utils.StringUtils`

- JSON 处理：`io.github.loadup.commons.utils.JsonUtils`

- 验证工具：`io.github.loadup.commons.utils.Validator`

使用建议：

- 在生成或实现新功能前，优先搜索 commons 中是否已有可复用的实现（Utilities、Result、Exception）。

- 公共异常：`BusinessException`，全局异常处理应返回 `Result<T>`。
