> 本文为英文原文 [`RESOURCE-EXHAUSTION-AND-AVAILABILITY.md`](RESOURCE-EXHAUSTION-AND-AVAILABILITY.md) 的简体中文译本。

# 资源耗尽与可用性狩猎

#### 何时使用本文件

当不受信任请求、消息、文件、租户状态或 Agent 工作可消耗 CPU、内存、磁盘、连接、worker 槽、付费 API 或队列容量，或可使共享服务死锁/崩溃时使用本文件。本域区分可源码审查的可用性漏洞与一般性能问题。切勿通过对共享或实时服务加压来验证。

对内存完整性缺陷使用 `MEMORY-SAFETY-AND-BINARY.md`，对 broker 投递逻辑使用 `PROTOCOLS-RPC-AND-MESSAGING.md`。即使底层解析器在别处覆盖，可达致命错误的共享影响也属于此处。

## 核心纪律（包含在本域每个 Agent 提示中）

```
- Require an input-to-cost path, a missing effective bound, and impact on another user, shared service, safety function, or operator-owned spend. Self-limiting work in the requester's own process is not a service vulnerability.
- A missing rate limit is not enough. Check body/message/file caps, concurrency, queues, deadlines, database constraints, upstream gateways, and per-tenant quotas before calling a path unbounded.
- Do not run stress, saturation, or production tests. Use asymptotic analysis, small boundary fixtures, mocked paid calls, strict local resource limits, and deterministic cancellation tests.
- State attacker cost, service work, persistence, scope, and recovery. One bounded input with superlinear or persistent shared effect is materially different from sustained volume.
- Use `confirmed` for source-visible bounds failures demonstrated safely. Use `needs_validation` when upstream caps, deployed topology, autoscaling, paid quota, or recovery behavior is outside the repository.
```

## 计算放大攻击类（subagent_type: `general`）

**超线性解析、匹配或评估**
小的接受输入驱动灾难性正则回溯、嵌套解析、递归验证、符号评估、图遍历、模板展开或对抗排序/哈希行为。派生接受深度/基数与复杂度，然后本地演示有界增长曲线。

**解压与表示放大**
压缩、稀疏、嵌套、别名或编码输入远超检查的传输或文件大小展开。在每次展开后与跨解析器阶段验证限制，包括归档、图像、字体、结构化文档与协议压缩表。

**数据库与下游查询放大**
小请求因查询深度、过滤基数、分页或展开字段无界而创建宽扫描、病态连接、扇出、无界排序/聚合或许多下游调用。确认授权并非有意允许相同资源范围。

## 资源累积攻击类（subagent_type: `general`）

**无界缓冲与基数**
体、乱序流、上传、会话、唯一缓存键、指标标签、日志字段、订阅或挂起作业在无每项与聚合限制下累积。在断开、超时、取消与部分解析上找到清理与过期。

**文件描述符、句柄与临时资源泄漏**
畸形或取消的工作错过清理，并保留 socket、文件、数据库游标、定时器、子进程、临时文件或对象引用。确认泄漏通过有界本地迭代重复并影响共享池。

**取消后的分离工作**
客户端超时、断开、取消作业或失败授权返回控制，但留下数据库、模型、网络或 worker 工作运行。追踪取消与截止时间通过每层的传播。

## 配额与调度攻击类（subagent_type: `general`）

**认证前工作不平衡**
昂贵解析、密钥查找、加密、解压或外部请求在认证与最早大小/速率门之前发生。比较最小请求者努力与共享服务成本，并检查上游限制。

**配额记账范围与重置缺口**
记账使用攻击者可影响的 IP、路由、租户、密钥前缀、任务 ID 或其他维度，允许一个主体的工作逃出其预期预算或消耗另一主体的分配。审查整数溢出、分布式竞态、重试、重连与账户切换。

**Worker、池与优先级饥饿**
低优先级或攻击者控制的作业持有无关用户所需的共享锁、worker、数据库池、事件循环回合或调度器优先级。要求绕过队列/并发公平性或超出截止保留槽的路径。

## 失败与恢复攻击类（subagent_type: `general`）

**可达致命错误或死锁**
不受信任输入在共享进程中到达 `panic`、中止、致命断言、未处理异常、进程退出、锁循环或无限循环。确认监督器范围以及是一个 worker 还是整个服务不可用。重启的隔离 worker 可降低影响但不抹去缺陷。

**重试风暴与 fail-open 放大**
超时、依赖错误、部分处理消息或健康检查失败在无抖动、上限、熔断或去重下触发同步或无界重试。验证一个有界失败源可创建持久聚合工作。

**毒化记录与队头阻塞**
一个畸形记录或消息在共享队列、分区、启动扫描、迁移或恢复循环前端反复失败。审查跳过/隔离策略、偏移，以及其他租户是否共享被阻塞单元。

**不安全恢复与容量回滚**
重启、恢复、回退或清理路径重建无界状态、忽略当前配额，或恢复立即重复失败的输入。恢复正确性是可用性的一部分。

## 通用动作（适用于上述）

- 构建输入到资源表：最早接受大小/基数、认证前工作、下游扇出、持久化、共享池、限制与清理所有者、恢复。
- 比较聚合限制与每对象限制。一万个有效单字节项可能规避每消息上限，同时耗尽租户范围或进程范围状态。
- 仅在带严格 CPU/内存/时间限制与小增长点的隔离夹具中验证。模拟外部与付费调用，一旦可观察缺失边界或取消即停止。

## 验证规则（在此报告任何 finding 前应用）

1. 命名不受信任输入、请求者工作、服务放大或保留资源、共享爆炸半径与恢复。无具体共享影响的缺失限制是加固。
2. 确认无源码可见上游、解析器、队列、租户或框架边界阻止该路径。未知已部署控制需要 `needs_validation`。
3. 对超线性行为，确立接受复杂度与有界本地增长。对泄漏，显示清理应发生后的可重复保留。对致命路径，识别进程/监督器隔离。
4. 按低请求者工作、未认证可达性、跨租户范围、持久化与差恢复优先；不要用可用性影响验证。
5. 仅以安全本地证明与有意义共享效果返回 `confirmed`。以所有者必须检查的精确上游限制、拓扑、配额或恢复观察返回 `needs_validation`。
