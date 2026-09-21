> 本文为英文原文 [`SUPPLY-CHAIN-AND-RELEASE.md`](SUPPLY-CHAIN-AND-RELEASE.md) 的简体中文译本。

# 供应链与发布狩猎

#### 何时使用本文件

当目标解析依赖、从未受信任贡献构建、运行 CI、创建发布产物、签名或晋升构建、加载插件或更新已部署软件时使用本文件。本域覆盖从源码与依赖到用户运行产物的信任交接。对本地二进制加载器内的缺陷使用 `MEMORY-SAFETY-AND-BINARY.md`，对运行时工作负载权威使用 `CLOUD-AND-DEPLOYMENT.md`。

将大型目标拆分为依赖解析、CI 隔离、产物来源、发布授权与更新器/插件信任。

## 核心纪律（包含在本域每个 Agent 提示中）

```
- A mutable or known-vulnerable dependency is not a finding by itself. Show who can influence resolution, which build consumes it, and what execution or release boundary follows.
- Follow integrity across every handoff: source identity, resolved inputs, build worker, artifact identity, test result, signature/attestation, promotion, and update consumer.
- CI configuration is authorization code. Establish which event triggered a workflow, whose code runs, which secrets and tokens exist, and what it may publish or mutate.
- A checksum fetched from the same untrusted location as the artifact does not establish independent integrity. Identify the trusted root and failure behavior.
- Use `confirmed` for in-repo control-flow failures with bounded local validation. Use `needs_validation` for branch protection, hosted-runner, registry, signing-service, or production promotion facts that are not observable.
```

## 依赖与构建输入攻击类（subagent_type: `general`）

**依赖源与命名空间混淆**
解析器配置可选择非预期的公共/私有命名空间、回退注册表、镜像、仓库或源 URL。审查包名、源优先级、lockfile 与校验和使用、替代构建文件、平台特定解析，以及首次安装相对更新行为。

**可变与未绑定构建输入**
构建消费分支、标签、未验证子模块、下载的工具、生成资产、远程包含、浮动 CI action 或内容可在无源码审查下变更的容器标签。要求较低信任写入者与进入受信任构建输出的路径；可复现性本身不证明真实性。

**生成源码与 codegen 来源缺口**
Schema、vendored 归档、生成客户端、本地化、文档示例或二进制 blob 在无与源码相同的审查与完整性门下产出可执行或发布内容。比较本地再生成与已提交输出，并验证谁控制输入与生成器。

**构建上下文包含**
密钥、本地配置、仓库元数据、测试夹具或开发者产物因构建上下文与忽略规则超出预期发布输入而进入包或镜像。确认结果产物暴露真实凭据、私有数据或特权配置。

## CI 与自动化攻击类（subagent_type: `general`）

**特权工作流中的不受信任代码**
拉取请求、议题评论、fork、依赖更新或外部事件以受保护密钥、写令牌、部署权威或受信任 runner 运行贡献者控制的代码。比较触发类型、checkout ref、批准门、环境保护与权限收窄。不要假设源码中不存在的仓库主机默认。

**工作流命令与表达式混淆**
攻击者控制的分支名、提交消息、议题字段、产物名、矩阵值或生成输出在无规范验证下进入 shell 命令、模板表达式、路径或特权工作流输入。

**缓存、产物与工作区信任混合**
较低信任作业可填充较高信任作业随后恢复并执行或发布的缓存、产物、共享工作区或输出。审查缓存键与命名空间、产物生产者身份、摘要绑定、保留，以及晋升是否按可变名重新解析。

**自动化身份越权**
CI 作业获得超出操作、仓库、环境或所需时长的权限，且不受信任作业输入可选择受影响资源。仅缺少最小权限是加固；要求可达的特权动作。

## 发布与更新攻击类（subagent_type: `general`）

**构建到晋升替换**
测试、审查、签名与发布引用可变标签、文件名、通道或产物 ID，而非同一不可变摘要。检查构建与发布之间的每次复制、重打包、架构合并与来源步骤。

**发布授权与签名策略缺口**
发布或签名被从错误工作流、仓库、分支、环境、密钥角色或阈值接受。审查证明内的身份声明，并验证消费者验证它们，而不只是有效签名。轮换、过期与撤销必须在策略要求处 fail closed。

**更新元数据与回滚混淆**
更新器认证载荷字节但不认证版本、产品、平台、通道、目标路径、过期或回滚状态，或它接受来自不同授权事务的元数据与载荷。验证原子安装与恢复行为。无策略绑定的签名 API 调用不完整。

**插件与扩展信任扩展**
扩展包获得超出其声明范围的主机权威，较低信任发布者可替换另一发布者身份，或安装/更新钩子在真实性与能力检查前运行。有意安装任意同用户插件不是权限边界。

## 通用动作（适用于上述）

- 从已发布摘要或已安装更新向后走到每个源、生成输入、凭据、worker、缓存、测试结果与授权决策。
- 并排比较不受信任与受保护工作流事件。标记它们之间每个持久通道交叉，并要求不可变身份加生产者信任。
- 审查撤销密钥、失败下载、缺失证明、部分平台发布、回滚与注册表中断路径。失败策略是发布完整性的一部分。

## 验证规则（在此报告任何 finding 前应用）

1. 命名较低信任参与者、可控源/缓存/产物/元数据、消费的受信任作业或更新器，以及结果未授权发布、代码包含、密钥披露或特权执行。
2. 证明跨破损交接的产物身份。不同可变名或未绑定摘要必须到达真实消费者。
3. 验证固定版本的内置包管理器、仓库主机、注册表与签名默认。未知托管控制需要 `needs_validation`。
4. 保持本地验证有界：使用无害夹具仓库、虚拟凭据标记、本地注册表/配置与非生产产物命名空间。不要发布或改动真实发布。
5. 仅以完整源码可见交接与有意义结果返回 `confirmed`。以所有者必须观察的精确分支、runner、注册表、签名或部署事实返回 `needs_validation`。
