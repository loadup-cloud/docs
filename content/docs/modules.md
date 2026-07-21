# loadup-modules 概览

`loadup-modules` 是业务模块集合，按业务域组织具体实现。

## 核心模块

### [Auth - 认证与授权](./modules/auth.md)

统一的认证授权体系，包括：

- **多种认证方式**：用户名密码、手机验证码、第三方OAuth登录
- **JWT无状态认证**：基于Gateway的统一认证，支持水平扩展
- **第三方登录**：支持微信、QQ、GitHub、Google等多种社交平台
- **安全机制**：登录锁定、验证码、防重放攻击等

### [UPMS - 用户权限管理系统](./modules/upms.md)

企业级用户权限管理系统，包括：

- **RBAC3 权限模型**：支持角色继承、职责分离、数据权限
- **组织架构管理**：无限层级部门树、部门维度授权
- **用户中心**：用户管理、OAuth绑定、头像管理
- **审计日志**：完整的操作记录和登录审计

## 开发建议

- 所有业务逻辑放在 modules 下，遵循模块依赖规则（commons → components → modules → application）
- Controller/Service/Mapper/Entity 命名规范参见 [Copilot 指令文档](./copilot-instructions.md)
- 新建业务模块请在 `loadup-modules` 下按功能创建子模块

## 相关文档

- [Gateway 网关](../gateway.md) - API 网关和路由
- [Components 组件](../components.md) - 可复用的组件库
