> 本文为英文原文 [`DESKTOP-MOBILE-AND-LOCAL-IPC.md`](DESKTOP-MOBILE-AND-LOCAL-IPC.md) 的简体中文译本。

# 桌面、移动与本地 IPC 狩猎

#### 何时使用本文件

当目标是桌面或移动应用、特权 helper、更新器、本地 daemon、webview 主机、深度链接处理程序、浏览器原生消息主机或本地 IPC 客户端/服务器时使用本文件。相关不受信任参与者可能是下载的文档、远程 Web 内容、另一本地应用、另一 OS 用户、沙箱进程或较低权限账户。陈述该起始能力，而非将所有本地用户视为等价。

对浏览器侧 webview 行为使用 `CLIENT-SIDE.md`，对原生内存与加载器安全使用 `MEMORY-SAFETY-AND-BINARY.md`，对更新真实性使用 `SUPPLY-CHAIN-AND-RELEASE.md`。

## 核心纪律（包含在本域每个 Agent 提示中）

```
- Establish the realistic local or remote-content attacker: another app, another OS user, a sandboxed child, an untrusted document, or a remote origin. Self-harm within the same account and authority is not a boundary violation.
- Paths, process names, bundle/package IDs, and claimed sender fields are not peer authentication. Use OS peer credentials, code identity, capability handles, or protected channel state.
- The native bridge or helper must authorize each operation and final resource after parsing. A trusted UI or broker does not make attacker-influenceable arguments trusted.
- OS sandbox, signing, entitlements, permissions, keychain ACLs, exported-component policy, and prompt behavior are real controls when pinned and visible.
- Use `confirmed` for source evidence plus bounded local/emulator tests. Use `needs_validation` when signing, manifest merge, OS version, device policy, installer ACL, or packaging is required but not observable.
```

## 深度链接、回调与导航攻击类（subagent_type: `general`）

**自定义 scheme 与深度链接歧义**
另一应用或页面可在无当前会话与一次性回调绑定下调用变更状态、导入数据、完成认证或选择账户的路由。审查 URI 规范化、重复查询字段、scheme/host/path 匹配、导出活动/处理程序策略与过时/重放链接。

**应用与账户交接混淆**
OAuth、SSO、魔法链接、邀请、设备配对、无密码或支付回调返回到错误已安装应用、配置文件、租户或挂起事务。将状态绑定到发起应用身份、当前会话、账户、提供商、操作与过期。

**文件打开与 intent 权威混淆**
关联文件、分享 intent、拖放项、粘贴板/剪贴板记录、通知动作或打开文件事件在无确认内容类型、适用时的发送者信任、当前用户意图与最终目标下触发特权操作。

## Webview 与原生桥攻击类（subagent_type: `general`）

**导航源到桥混淆**
远程或攻击者控制帧可到达仅意图用于打包内容的 JavaScript/原生桥。在调用时以及每次导航、重定向、子帧创建、弹窗与错误/回退页后验证源。URL 前缀检查与初始加载检查不足。

**过宽原生桥能力**
Web 内容可通过通用桥选择任意文件、命令、IPC 方法、凭据或系统动作。检查方法 allowlist、规范化参数、用户/租户权威、手势/确认要求与返回值披露。

**Webview 文件与通用访问**
远程内容因文件访问、通用访问、混合内容、调试接口或自定义协议处理程序意外连接源而可读应用本地文件、特权自定义 scheme 或内部源。缺失限制性设置但无可达受保护内容是加固。

## 本地 IPC 与导出组件攻击类（subagent_type: `general`）

**IPC 对等认证缺口**
Unix socket、命名管道、XPC、Binder、D-Bus、原生消息、RPC、共享内存或 loopback 监听器在无检查 OS 凭据、代码身份、沙箱令牌或通道所有权下接受较低信任对等方。要求通道后有有意义方法或披露。

**声称主体相对通道身份**
已认证进程/通道属于一个应用或用户，但请求字段选择另一用户、租户、配置文件或能力。将每个方法与资源绑定到对等凭据，而非调用方声明的标识符。

**导出服务、活动、接收器或提供商越权**
移动组件或本地自动化端点可被外部调用，并执行意图仅供应用本身的操作。审查最终合并清单、intent 过滤器、权限/签名级别、路径授予与替代别名。打包后清单状态未知需要 `needs_validation`。

**IPC 生命周期与关联混淆**
可预测请求 ID、重用句柄、过时通道、继承描述符、全局可写 socket 路径或重启行为让一个对等方回答、取消或重用另一对等方的操作。审查 socket 文件、锁、端口与共享映射的创建权限与清理。

## 特权 helper 与本地文件攻击类（subagent_type: `general`）

**特权 helper 作为混淆代理**
低权限调用方可在无每操作授权下选择特权命令、文件、服务、用户或系统设置。审查 sudo/polkit/UAC/XPC helper 规则，并确保 helper 独立验证规范化参数。

**安装、更新与修复路径信任**
特权安装器/helper 在授权后读取较低信任参与者可写的清单、脚本、包、符号链接、工作目录或修复状态。将授权绑定到不可变内容与安全目的地路径。

**本地文件所有权与 TOCTOU**
应用检查文件/路径，然后在特权读/写期间跟随替换、符号链接、挂载或大小写/规范化变更。使用描述符相对操作并验证最终所有权。在文件打开后将解析聚焦到 `MEMORY-SAFETY-AND-BINARY.md`。

**凭据存储与本地密钥边界不匹配**
钥匙串/密钥库项、令牌文件、备份、日志、剪贴板、通知预览或本地配置可被具有较少权威的另一应用/配置文件/用户读取。仅同一预期 OS 账户可读的明文不自动是漏洞；陈述较低信任读者与凭据能力。

## 应用状态与设备生命周期攻击类（subagent_type: `general`）

**账户切换、注销与设备恢复泄漏**
缓存数据、后台任务、小组件、通知、本地数据库、webview 存储或生物识别批准在注销/账户变更后存活，并在后续账户下出现。审查备份/恢复与多配置文件行为。

**挂起动作与用户在场混淆**
通知、小组件、快捷方式、分享表、生物识别提示或延迟操作授权与显示不同的动作、在过期后执行，或使用另一配置文件的挂起状态。将确认绑定到规范化动作、资源、账户与当前前台状态。

## 通用动作（适用于上述）

- 枚举每个进程、应用组件、本地端点、URI scheme、文件关联、webview 源与 helper。记录 OS 身份、运行时权限、调用方与可调用操作。
- 阅读最终打包输入：合并清单、entitlement、安装器规则、原生消息注册、协议处理程序与 ACL 创建。源码声明可在下游被覆盖。
- 在隔离机器/模拟器上用虚拟配置文件与非敏感本地夹具验证。不要与其他用户的应用、凭据或生产服务交互。

## 验证规则（在此报告任何 finding 前应用）

1. 命名攻击者起始能力、穿越的 OS/应用主体、入口通道、接受的参数或状态，以及未授权操作或披露。
2. 确认适用的 OS 沙箱、对等凭据、签名、entitlement、权限、用户同意与安装器控制。未知打包/运行时事实需要 `needs_validation`。
3. 对 webview 桥，同时引用导航/源控制与特权原生汇点。对 IPC，引用对等认证与每资源授权。对 helper，验证最终规范化目的地。
4. 保持本地测试有界，并使用虚拟内容/账户。在证明边界结果后停止；不要将证明扩展到持久化或更广系统修改。
5. 仅以完整源码与本地证据链返回 `confirmed`。以所需的精确 OS、清单、签名、ACL 或设备生命周期事实返回 `needs_validation`。
