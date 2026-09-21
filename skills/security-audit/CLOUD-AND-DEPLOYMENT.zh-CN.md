> 本文为英文原文 [`CLOUD-AND-DEPLOYMENT.md`](CLOUD-AND-DEPLOYMENT.md) 的简体中文译本。

# 云与部署狩猎

#### 何时使用本文件

当仓库定义云身份、基础设施、容器、Kubernetes、服务网格、无服务器函数、边缘 worker、入口、对象存储、托管服务或环境特定配置时使用本文件。本域问已部署组件是否获得预期身份、隔离、网络可达性、密钥与策略。源码常表达意图而非实时事实，因此将源码确认缺陷与部署验证需求分开。

对构建与晋升信任使用 `SUPPLY-CHAIN-AND-RELEASE.md`，对 HTTP 代理语义使用 `WEB-PROTOCOL-AND-AUTH.md`，对数据存储租户范围使用 `DATA-ISOLATION-AND-LIFECYCLE.md`。

## 核心纪律（包含在本域每个 Agent 提示中）

```
- Do not infer a live exposure from a manifest alone. Establish which environment consumes it, what defaults or overlays modify it, and whether the source path is active.
- Map each workload's identity to specific operations and resources. Broad policy is a finding only when lower-trust input can reach an unauthorized action.
- Ingress, proxies, service mesh, metadata services, and admission policy are real boundaries, but only count a control when its configuration and attachment are visible.
- Secret references are not secret disclosure. Require a lower-trust reader, output, artifact, log path, or unsafe fallback.
- Use `confirmed` for active in-repo configurations and local rendering/policy validation. Use `needs_validation` for account policy, network attachment, runtime admission, hosted metadata, or drift that needs owner observation.
```

## 工作负载身份与 IAM 攻击类（subagent_type: `general`）

**工作负载身份越权**
工作负载、pod、函数、边缘 worker 或节点身份可对超出其角色的租户、账户、资源或 API 行动，且不受信任请求或作业输入选择该目标。审查云策略条件、资源模式、服务账户附加、命名空间映射与回退凭据。

**跨账户或跨租户角色混淆**
角色假设、外部 ID、令牌交换、工作负载联邦或资源策略接受未绑定到预期源账户、受众、仓库、命名空间或工作负载的身份声明。同时确立信任策略与调用方控制的声明。

**委托给云元数据的应用授权**
应用信任调用方提供的身份头、标签、label、账户 ID 或资源元数据，而不验证它们来自云控制面或受信任代理。云 IAM 与应用授权是分开的检查。

## 入口、网络与控制面攻击类（subagent_type: `general`）

**意外服务或管理面可达性**
入口、服务、监听器、安全组、负载均衡器注解、端口映射或服务器绑定将管理、调试、指标、节点、控制面或内部 API 暴露给较低信任网络。仅缺失网络控制是 `needs_validation`；到敏感处理程序的仓库控制公共路由可为 `confirmed`。

**受信任代理与网格身份绕过**
后端从预期入口/sidecar 外的对等方接受转发身份、mTLS 主体或授权元数据，或替代端口与健康/遗留路径绕过网格。验证头剥离、对等可达性，以及代理缺失时的 fail-open 行为。

**元数据与内部服务可达性**
不受信任 URL、目的地或协议选择以工作负载凭据到达实例/容器元数据、控制面 socket 或内部 API。在 `ATTACK-CLASSES.md` 下追踪 URL 解析与重定向处理；此处确立已部署网络、元数据版本与身份边界。

## 容器与编排攻击类（subagent_type: `general`）

**主机或控制面能力暴露**
较低信任工作负载可选择特权模式、能力、主机命名空间、主机路径、设备挂载、容器运行时 socket 或跨入节点/控制面权威的服务账户令牌。仅缺少 seccomp 或只读文件系统是加固，除非可达操作穿越该边界。

**准入与策略路径不一致**
一条部署路由强制镜像身份、命名空间、资源、密钥或权限策略，而另一控制器、作业、升级、恢复或兼容路径不强制。确认替代路由与结果部署对象。

**命名空间与标签信任混淆**
网络、准入、密钥或工作负载身份策略依赖较低信任主体可设置的标签、注解、名称或命名空间。比较谁控制选择器与匹配授予什么权威。

## 配置与密钥生命周期攻击类（subagent_type: `general`）

**安全控制优先级漂移**
开发值、chart 默认、环境变量、命令行标志、功能门、sidecar 注入或每区域覆盖在已部署环境中禁用认证、传输安全、租户范围或审计策略。为每个维护的部署渲染最终配置，不只是基础文件。

**跨工作负载边界的密钥暴露**
密钥进入另一工作负载或租户可访问的日志、崩溃报告、进程参数、共享环境、宽卷、构建输出、服务发现或读 API。检查密钥类型与权威；公共端点或密钥 ID 不是凭据。

**凭据续期与中断回退**
未能挂载、刷新、轮换或撤销工作负载凭据导致过时凭据保持活动，或应用接受较低信任身份模式。审查启动、就绪、重连与缓存客户端行为。

## 托管存储、事件与边缘攻击类（subagent_type: `general`）

**对象与签名 URL 策略混淆**
桶/容器策略、对象键、CDN 源或签名 URL 未能绑定主体、操作、对象命名空间、受众或过期。审查列表/版本操作与写路径以及读。

**事件源身份混淆**
函数或 worker 在无验证提供商签名信封、订阅/主题、账户、区域与重放状态下将事件体字段信任为源身份。比较推送、拉取、重试与死信路径。

**边缘/运行时边界不匹配**
边缘或无服务器运行时假设与源运行时不同的密钥、API、文件系统、隔离或租户策略，且回退到源改变权威或缓存行为。确认哪个配置选择每条路径。

## 通用动作（适用于上述）

- 渲染每个维护环境，并制作外部端口、工作负载身份、网络对等方、挂载密钥与云资源矩阵。差异需要所有者或策略解释。
- 跟随较低信任请求、对象、标签或事件进入云策略。显示哪个工作负载凭据执行最终操作，以及什么条件应限定它。
- 对比正常部署、迁移、恢复、节点维护、故障转移与本地/模拟器路径。审查网格、准入、身份、密钥或策略服务不可用时的行为。

## 验证规则（在此报告任何 finding 前应用）

1. 确立活动源码路径与有效部署对象；否则使用 `needs_validation` 并陈述缺失哪个渲染清单或所有者观察附加。
2. 命名较低信任调用方/工作负载、云或应用身份、可控选择器、受影响资源与未授权操作或披露。
3. 验证固定版本的提供商与编排器默认。不要假设公共 IP、可达元数据服务、宽松防火墙或缺失准入附加。
4. 本地验证可渲染模板、评估策略、在隔离夹具中检查容器/用户命名空间，或用虚拟身份运行模拟器。不要探测实时端点或改动共享云资源。
5. 仅以完整活动源码追踪与具体边界结果返回 `confirmed`。以所需的精确已部署策略、身份附加、覆盖、网络或漂移观察返回 `needs_validation`。
