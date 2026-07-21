---
id: components-scheduler
title: LoadUp Scheduler Component
---

> 来源: `loadup-components/loadup-components-scheduler/README.md`

# LoadUp Scheduler Component

统一的调度/任务组件，提供统一 API 与 Binder 插件（SimpleJob/Quartz/XXL-Job/PowerJob）。支持注解式任务
`@DistributedScheduler`、任务注册/暂停/触发、动态创建/修改 Cron 等。

主要点：

- 统一 API 层：SchedulerBinding / SchedulerTask / SchedulerTaskRegistry
- Binder 层支持多种调度实现（SimpleJob、Quartz、XXL-Job、PowerJob）
- 完整测试覆盖与性能基准
- 配置示例及生产/开发差异说明

快速配置示例：

```yaml
loadup:
  scheduler:
    type: simplejob  # 或 quartz/xxljob/powerjob
```

原始模块 README 路径：

- `loadup-components/loadup-components-scheduler/README.md`

(已整理为 docs 页面，含快速开始与常见问题摘要。)
