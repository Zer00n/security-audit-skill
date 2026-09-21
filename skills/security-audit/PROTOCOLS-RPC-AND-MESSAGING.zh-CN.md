> 本文为英文原文 [`PROTOCOLS-RPC-AND-MESSAGING.md`](PROTOCOLS-RPC-AND-MESSAGING.md) 的简体中文译本。

# 协议、RPC 与消息狩猎

#### 何时使用本文件

当目标使用 gRPC、GraphQL 传输、Cap'n Proto、Thrift、Protobuf、自定义二进制协议、流式 RPC、webhook、broker、队列、pub/sub 或事件总线时使用本文件。它覆盖对等身份、逻辑消息解释、路由、重放、排序与投递语义。对解析器内存安全使用 `MEMORY-SAFETY-AND-BINARY.md`，对 HTTP 分帧使用 `WEB-PROTOCOL-AND-AUTH.md`，对可用性影响使用 `RESOURCE-EXHAUSTION-AND-AVAILABILITY.md`。

按生产者/消费者对、外部/内部对等角色、同步 RPC、流式与异步消息路径拆分大型系统。

## 核心纪律（包含在本域每个 Agent 提示中）

```
- "Internal" is not authentication. Name the peer identity at every hop and show how it becomes the application principal used for authorization.
- Schema validation proves message shape, not provenance, resource authority, ordering, or safe values. Follow decoded fields to policy and side effects.
- Broker guarantees and application guarantees differ. Write down retry, ordering, acknowledgement, deduplication, and transaction behavior before evaluating state changes.
- Parser disagreement requires two concrete consumers, schema versions, or wire representations and one security-relevant divergent value.
- Use `confirmed` for source-complete paths plus bounded local producer/consumer tests. Use `needs_validation` for broker ACL, service-mesh identity, topic attachment, or compatibility behavior outside the repository.
```

## 分帧、schema 与解释攻击类（subagent_type: `general`）

**消息边界与规范化不一致**
组件在长度、压缩、重复字段、未知字段、编码、数值宽度、规范化或信封/体优先级上不一致。比较生成与自定义解析器、网关、语言绑定与版本转换器。确认解码后哪个主体、资源或操作不同。

**联合、枚举与默认混淆**
未知变体、缺失判别器、零值、默认权限或兼容映射到达假设已验证情况的代码。审查穷尽分发、默认分支，以及旧消费者如何解释新添字段。

**信封与载荷身份不匹配**
授权使用看起来受信任的路由或信封元数据，而处理程序对体中冲突的租户、账户、主体、对象或发送者行动。识别哪个源权威，并确保客户端不能覆盖它。

## RPC 身份与授权攻击类（subagent_type: `general`）

**拦截器与方法路径不一致**
认证/授权拦截器应用于一元方法但不应用于流、反射、健康、网关转码路径、兼容服务或单个流消息。比较每个注册与到同一操作的路由。

**对等身份到应用主体混淆**
mTLS、工作负载身份、bearer 元数据、转发身份或 broker 凭据认证通道，但调用方控制字段选择用户或租户。通道身份与声称主体必须由确定性策略绑定。

**每项与流式授权缺口**
流、订阅、批处理或批量消息被授权一次，然后后续项命名不同资源，或在角色、成员资格或令牌撤销后继续。在范围可变处重新检查，并将订阅绑定到其原始主体。

**回调与回复关联混淆**
可预测、重用或跨租户关联 ID 让响应、webhook、取消或确认满足另一调用方的挂起操作。将每个未决请求绑定到已认证对等方、租户、操作与生命周期。

## Broker 与队列隔离攻击类（subagent_type: `general`）

**主题、路由键与订阅范围缺口**
发布者或订阅者可选择另一租户的主题、通配符、消费者组、分区、回复队列或死信路由。在可见处检查 broker 强制 ACL 与应用侧命名空间构造。载荷内的租户文本不是隔离。

**死信、重试与诊断披露**
路由到死信队列、错误主题、追踪或运营视图的消息包含较低信任消费者可访问的密钥或跨租户载荷。在失败路径审查策略与脱敏，不只是正常投递。

**将不受信任生产者当作控制面**
消息体可在无独立认证生产者与事件类型下声明自己为管理事件、提供商回调、复制记录或迁移指令。在特权处理前验证签名与源/账户/受众绑定。

## 重放、排序与事务攻击类（subagent_type: `general`）

**重复投递与幂等缺口**
重试或再投递重复副作用，因为去重缺失、发生在变更后，或使用跨租户或操作碰撞的键。确认 broker 的投递模型与非自然幂等的副作用。

**乱序与过时消息接受**
较旧状态、已撤销成员资格、已取消工作或升级前授权在较新状态之后到达并覆盖它。审查序列/版本检查、墓碑、分区变更与恢复/重放工作流。

**确认/提交排序缺陷**
确认发生在持久提交前并丢失安全相关工作，或提交发生在不可靠确认前并重复变更。评估事务性 outbox/inbox 行为与失败恢复。

**部分多消费者过渡**
若干消费者联合实现一个授权或业务过渡，但重试与部分失败仅使子集提交。识别必须原子持久化或用当前授权补偿的不变量。

## 通用动作（适用于上述）

- 为每个消息族画生产者 → broker/传输 → 网关 → 消费者 → 存储。在每跳记录已认证对等方、权威租户/资源字段、验证与副作用。
- 向每个仓库内 schema 版本或语言绑定馈送相同小型夹具。在不产生负载的情况下测试重复、缺失、未知、边界、重放与乱序消息。
- 比较正常、重试、死信、重放、迁移、反射、流与网关转码路由。安全策略必须在传输变更后存活。

## 验证规则（在此报告任何 finding 前应用）

1. 命名现实生产者或对等方、接受的消息、已认证通道身份、受影响主体/资源与未授权变更或披露。
2. 对不一致主张，引用两个解析器/消费者与分歧解码值。任一侧的安全拒绝阻止确认。
3. 对重放/顺序主张，确立实际投递保证，并用有界本地/内存传输复现不变量失败。
4. 对授权与隔离，验证源码中可见的所有拦截器、broker ACL、网关与消费者层。外部附加使候选为 `needs_validation`。
5. 仅以完整消息生命周期与观察到的有意义结果返回 `confirmed`。以所需的精确 broker、服务身份、路由或投递事实返回 `needs_validation`。
