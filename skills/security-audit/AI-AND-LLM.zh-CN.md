> 本文为英文原文 [`AI-AND-LLM.md`](AI-AND-LLM.md) 的简体中文译本。

# AI、LLM 与 Agent 狩猎

#### 何时使用本文件

当语言模型参与信任敏感决策时使用本文件：聊天机器人与助手、RAG 管道、持久 Agent 记忆、Agent/工具调用循环、MCP 服务器与客户端、从不受信任输入构建提示的代码，或消费模型输出并据其行动的代码。重要数据流是 *不受信任内容 → 模型或记忆 → 能力、权威或汇点*。

与 `ATTACK-CLASSES.md` 一起使用，而非替代。传输、访问控制、查询构造、文件系统使用与输出渲染仍是普通信任边界。本文件覆盖模型特定委托层。按检索、记忆、工具分发、MCP 与输出处理拆分大型目标。

## 核心纪律（包含在本域每个 Agent 提示中）

```
- Prompt injection alone is not a finding. Require a code-level boundary failure: content reaches another principal's context, invokes authority the requester lacks, discloses data they cannot read, or drives a sink they cannot reach directly.
- Model output, memory, tool descriptions, and MCP responses are untrusted inputs. Point to the code that grants authority, trusts output, writes durable state, or feeds a sink.
- A guardrail prompt is not a security boundary. Count only deterministic checks, resource-scoped authorization, isolation, binding, and constrained credentials.
- State the attacker, affected principal, effective execution identity, resource, exact action, authority used, and observable impact. An intentional direct request to use the requester's existing authority is not a delegation defect merely because a model executes it.
- Authorization and action binding are separate controls. Attacker-controlled content that causes an action under an affected principal's valid authority is an action-binding failure when that principal did not intentionally request or approve the exact action.
- Classify every candidate as `confirmed` only after source evidence and bounded local validation establish the boundary and result. Use `needs_validation` when a required provider, deployment, model, renderer, or identity behavior is not observable locally.
```

## 上下文、检索与记忆攻击类（subagent_type: `general`）

**通过检索或摄取内容的间接注入**
攻击者可写入进入不同主体模型上下文的 RAG 文档、索引页、文件、邮件、议题正文、工具响应或元数据。追踪谁可写每个源、检索如何限定范围、谁的会话消费它，以及那里启用了什么能力。分别检查隔离、资源授权与到消费主体意图的绑定。缺陷是缺失确定性控制，而非说服性文本本身。

**跨会话或跨租户上下文渗漏**
对话历史、嵌入、检索结果或提示缓存键控过宽。在查询本身与每个缓存键中验证租户与 ACL 过滤器。对象上存储租户字段若替代查询、共享缓存或批处理路径省略它，则不是强制。

**持久记忆投毒**
攻击者控制的内容或模型摘要被写入随后塑造另一任务、用户或特权会话的记忆。审查谁可创建、更新、合并与删除记忆；其来源与租户范围；低信任观察是否成为持久指令或事实；以及检索是否区分用户偏好与工具策略。用户有意保存且仅用于该用户有意、允许请求的记忆不是跨边界 finding。

**提示角色与来源混淆**
提示组装允许不受信任文本冒充系统消息、先前回合、工具结果、策略或记忆记录。寻找字符串拼接、无类型历史、调用方控制的角色字段，以及丢失源标签的序列化往返。确认伪造来源改变确定性信任决策或到达有意义能力。

## 工具与动作攻击类（subagent_type: `general`）

**工具参数注入到下游汇点**
模型产生的参数在无处理程序侧验证下到达 SQL、shell、文件、URL 获取或特权 API。将工具 schema 视为输入解析，然后跟随每个字段从解码调用到汇点。结构化输出收窄形状；它不确立授权、安全路径、安全 URL 或查询语义。

**过度代理与混淆代理权威**
Agent 使用服务身份或宽凭据，而工具处理程序不重新检查请求主体对命名资源的权限。验证有效身份以及调用方能否通过正常产品接口执行该确切操作。带强制每用户查询范围的共享凭据不是缺陷。

**动作确认与批准绑定**
用户批准一个描述的动作，但执行可使用变更的参数、不同资源、不同主体或后续模型回合。当攻击者控制内容在受害者有效权威下导致副作用且受害者未有意请求或批准时，也存在动作绑定缺陷，即使通用授权允许受害者执行它。审查意图或确认是否绑定规范化工具名、完整参数对象、请求者、目标、金额、过期与批合成员。检查重试与恢复会话：批准不得授权变更或重复副作用。

**工具 schema 与分发器不一致**
Schema 接受分发器或处理程序不同解释的别名、额外字段、重复键、强制、嵌套自由形式对象或超范围值。比较 schema 验证、规范化、生成绑定与处理程序默认值。在值成为资源选择器或安全相关选项处再次验证。

**无界委托动作循环**
有界请求可在无每请求预算、每动作授权、取消或幂等控制下排队重复支出、发送、变更或外部 API 工作。确认对共享成本、配额、其他用户或持久状态的影响。不要通过耗尽服务测试；使用代码级记账与本地有界循环。

## MCP 与子 Agent 信任类（subagent_type: `general`）

**子 Agent 与 MCP 信任继承**
委托任务接收完整会话、凭据、记忆或能力，而非所需最少权威。检查带入每次调用的主体与租户、能力收窄、凭据受众，以及委托结果返回时是否视为不受信任。

**MCP 服务器与工具身份混淆**
调用或结果按攻击者可影响的服务器名、工具名、请求 ID、资源 URI 或模型选择别名路由，而非按已认证连接与未决请求。检查两个服务器是否可声称同一工具或资源身份、重连是否改变绑定，以及一个服务器的响应是否可满足另一服务器的挂起调用。

**将 MCP 元数据与 schema 当作策略**
MCP 对等方提供的工具描述、资源元数据、提示、完成提示或 schema 被当作策略或授权信任。这些字段可引导模型但不能授予能力。找到在元数据冲突时仍权威的确定性 allowlist、服务器身份检查与处理程序授权。

## 输出与披露攻击类（subagent_type: `general`）

**不安全输出渲染**
模型输出在无汇点所需编码与策略下到达执行 HTML、Markdown、模板、URL 或命令汇点。对浏览器渲染，在 `CLIENT-SIDE.md` 中验证自动加载资源与 CSP 或净化；仓库外的渲染器行为使候选为 `needs_validation`。

**敏感上下文提取**
组装的上下文包含凭据、另一用户数据、私有源码或本身授予访问的策略值，且用户影响的输出暴露它们。阅读提示组装与数据获取代码。不跨数据边界的通用指令或行为披露不是 finding。

## 通用动作（适用于上述）

- 先画四张图：每个执行身份、每个能力、每个可写上下文或记忆源，以及每个输出目的地。然后将起点主体连接到终点权威。
- 从有副作用的工具向后穿过分发器、schema、确认、模型上下文、检索与摄取。从持久记忆读开始追踪每个写入者。
- 比较同一动作的直接、排队、重试、恢复、批处理与委托路径。最强门必须在参数最终化后、每个副作用前应用。

## 验证规则（在此报告任何 finding 前应用）

1. 命名穿越的边界与可观察结果：攻击者、受影响主体或共享资源、执行身份、目标，以及未授权或未请求的动作或披露。
2. 对混淆代理权威主张，证明工具缺少请求者与资源授权，且攻击者不能正常执行相同动作。对动作绑定主张，改为证明攻击者控制内容在受影响主体权威下导致主体未有意请求或批准的动作。有效通用授权不确立该意图。
3. 对记忆或检索主张，同时引用攻击者控制的写入与后续跨主体读或特权决策。无可达消费者的共享记录不够。
4. 对动作绑定，确立有意请求或规范化批准对象（若有），并与处理程序使用的对象比较。在不将测试扩展到有害执行的情况下，确认本地可观察的未请求动作、变更、重复或权威变更。对 schema 不一致，比较规范化验证对象与处理程序对象。
5. 对 MCP 身份主张，验证已认证连接、请求关联、工具命名空间与有效凭据。若需要外部服务器身份或部署路由，标记 `needs_validation`。
6. 仅以完整源码追踪与有意义结果返回 `confirmed` finding。对特定未决边界事实返回 `needs_validation`，并陈述解决它所需的有界本地或所有者观察检查。
