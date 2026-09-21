> 本文为英文原文 [`DATA-ISOLATION-AND-LIFECYCLE.md`](DATA-ISOLATION-AND-LIFECYCLE.md) 的简体中文译本。

# 数据隔离与生命周期狩猎

#### 何时使用本文件

当目标存储多租户或访问控制数据、派生搜索/索引/缓存/分析副本、签发对象链接、导出或恢复记录、迁移 schema，或承诺删除、撤销与保留行为时使用本文件。本域跟随一个数据项通过每个副本与状态转换。对端点级访问控制使用 `ATTACK-CLASSES.md`，对提供商级存储策略使用 `CLOUD-AND-DEPLOYMENT.md`。

按主存储、缓存/搜索、对象/blob 存储、分析/日志、导出/备份、删除/撤销与迁移拆分大型目标。

## 核心纪律（包含在本域每个 Agent 提示中）

```
- A tenant or owner field on a record is not isolation. Find the query, key, path, policy, or row-level control that enforces it for each read and write path.
- Trace derived copies. Sanitized primary data can become unsafe in search, cache, analytics, export, previews, logs, replicas, and backups with different ACL and retention rules.
- Deletion and revocation are lifecycle contracts. Check current, historical, cached, indexed, exported, restored, and queued copies within the product's stated boundary.
- Privacy or retention preference is not automatically a security vulnerability. Require an explicit data-access boundary or deletion/revocation guarantee and an unauthorized reader or later operation.
- Use `confirmed` for complete source-visible lineage and bounded dummy-tenant tests. Use `needs_validation` when external storage policy, retention, CDN behavior, replica lag, or backup access is unavailable.
```

## 租户与对象隔离攻击类（subagent_type: `general`）

**缺失租户或所有者强制**
读、更新、删除、列表、计数或批量查询在未绑定到已认证租户/所有者的情况下标识对象，或信任体字段提供该身份。比较直接查找、嵌套关系、后台、管理、导入与遗留路径。

**复合键与命名空间碰撞**
缓存键、对象路径、数据库唯一性、搜索文档 ID、临时文件或去重键省略租户或环境。两个主体可覆盖或检索同一逻辑键，即使应用记录携带分开所有者。

**策略与查询不一致**
行级策略、ORM 默认范围、授权过滤器与原始/绕过客户端应用不同谓词。检查连接、聚合、别名、视图、事务、`unscoped` 或服务客户端，以及上下文缺失的错误路径。

**Blob 与签名引用越权**
对象键、附件 ID、版本 ID、共享链接或签名 URL 允许超出签发主体访问的操作或命名空间，或在底层 ACL 变更后仍有效。绑定操作、确切对象/版本、受众、过期与租户。

## 派生数据与披露攻击类（subagent_type: `general`）

**搜索、缓存与索引 ACL 漂移**
主记录的 ACL 或生命周期变更而不使可搜索、缓存、嵌入、缩略图、RSS、预览或索引副本失效。在检索时以及文档摄取与失效时验证过滤。

**分析、日志、追踪与诊断作为替代读者**
私有内容或凭据被发射到具有更广访问、更长保留或租户混合的系统。确认数据类与现实读者；字段名、公开标识符与预期策略下的仅运营内容不够。

**枚举与聚合 oracle**
计数、过滤、排序、错误、唯一约束、时序、通知行为或存在检查披露受保护对象或账户状态。要求具体机密谓词与可观察区别，而非一般响应方差。

## 导出、备份、恢复与迁移攻击类（subagent_type: `general`）

**导出与备份范围扩展**
导出、快照、可移植包、报告或备份包含其他租户、不可访问对象字段、软删除数据、密钥值或超出请求者访问的历史。在选择后检查每项授权以及下载最终产物的授权。

**导入与恢复权威扩展**
恢复/导入绕过所有者、schema、ACL、唯一性或验证规则，覆盖现有资源，或在请求者不能写的租户中重建记录。将归档内容验证为不受信任，并授权结果操作而非信任先前来源。

**迁移默认与所有权混淆**
旧记录缺少租户/ACL/生命周期字段、不兼容 ID 碰撞，或部分推出使新旧读者应用不同默认。审查回填、双读/写、兼容、回滚与恢复迁移路径。

**备份与复制边界漂移**
加密密钥、存储账户、跨区域副本、恢复环境或支持快照具有比主数据更广的身份或租户范围。源码仅可确认仓库内策略；托管访问与保留需要 `needs_validation`。

## 删除、撤销与生命周期攻击类（subagent_type: `general`）

**软删除与墓碑绕过**
直接查找、搜索、关系遍历、对象链接、后台处理器或恢复忽略生命周期谓词，并返回或对已删除/撤销记录行动。检查软删除标识符能否在所有引用消失前重新注册。

**过时授权与派生副本使用**
成员资格移除、ACL 更新、同意撤回、密钥撤销或角色降级不使继续授权未来操作的会话、缓存、订阅、作业或物化数据失效。

**保留与排队工作越界**
删除在主存储中完成，而排队处理器、重试、导出、分析或生成产物在承诺边界外重建或保留数据。找到幂等删除与墓碑传播。

**恢复重新引入无效状态**
备份、撤销、取消删除或副本恢复恢复当前策略不再允许的数据、凭据、成员资格或权限。重新授权恢复状态，并重新应用快照后做出的生命周期变更。

## 通用动作（适用于上述）

- 选一个受保护记录，画主写、查询、缓存、索引、事件、导出、备份、删除与恢复路径。在每条边上标记主体与租户。
- 通过相同本地服务方法比较两个虚拟租户，然后在 ACL 变更、删除、账户切换与恢复后重复。不要使用真实用户数据。
- 从绕过客户端、后台作业、迁移、全局唯一性与缓存键开始。这些路径常省略交互端点携带的请求范围身份。

## 验证规则（在此报告任何 finding 前应用）

1. 命名攻击者或较低信任主体、受保护数据/状态、受影响所有者/租户、替代副本或操作，以及未授权披露或变更。
2. 同时引用预期事实来源策略与省略或与其不一致的路径。确认另一层不强制相同租户/生命周期条件。
3. 使用本地虚拟租户与非敏感夹具证明跨范围访问或过时生命周期行为。停在最小可观察记录或操作。
4. 若需要外部缓存、对象存储、副本、分析、备份或保留策略，分类为 `needs_validation` 并陈述所有者观察检查。
5. 仅以完整谱系与具体边界影响返回 `confirmed`。以精确未决存储、ACL、失效、保留或恢复事实返回 `needs_validation`。
