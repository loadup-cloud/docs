---
title: Inbox 站内收件箱
---

Inbox 保存当前用户收到的站内信，提供分页查询、未读计数、标记已读和归档。实际收到的标题、正文和跳转地址在入箱时固定。

| Maven 坐标 | 职责 |
| --- | --- |
| `loadup-modules-inbox-client` | `InboxReceiver` 幂等接收已渲染消息，`InboxFacade` 管理收件人的查询和状态。 |
| `loadup-modules-inbox-core` | 通过 Database 组件持久化。MyBatis-Flex 根据 `InboxMessageDO` 生成 Mapper 和表定义。 |
| `loadup-modules-inbox-web` | 通过 `/api/inbox` 处理收件人请求。收件人 ID 取自登录身份。 |

三个 JAR 由 `loadup-modules-bom` 管理，groupId 为 `io.github.loadup-cloud`。宿主接入 `inbox-web` 并将 `classpath:db/migration/modules/inbox` 加入 Flyway，在新 schema 中初始化 `inbox_message`，同时提供 Database 组件所需的序列分配表。Inbox 模块开关为 `loadup.modules.inbox.enabled`，Web 开关为 `loadup.modules.inbox.web.enabled`。

Gotone 的 `IN_APP` 任务传入收件人 ID、渲染内容和稳定的 `deliveryId`。Inbox 以 `(recipient_id, request_key)` 去重，重复调用保留首次入箱内容。宿主将两个模块装配在同一个应用中，使用同一个 PostgreSQL DataSource 和 JDBC 事务管理器。入箱和标记 `SENT` 一起提交或回滚。

| 用户操作 | POST 路径 |
| --- | --- |
| 查询消息 | `/api/inbox/list` |
| 查询未读数 | `/api/inbox/unread-count` |
| 标记一条已读 | `/api/inbox/read` |
| 标记全部已读 | `/api/inbox/read-all` |
| 归档一条消息 | `/api/inbox/archive` |

[Gotone 投递说明](../gotone/)
