> 本文为英文原文 [`CLIENT-SIDE.md`](CLIENT-SIDE.md) 的简体中文译本。

# 客户端与浏览器狩猎

#### 何时使用本文件

当有意义的信任决策或不受信任渲染发生在浏览器中时使用本文件：单页应用、浏览器扩展、嵌入式 webview、service worker、离线应用，以及将攻击者可影响内容渲染到 DOM、接收跨窗口消息或使用浏览器存储的代码。这些路径包括服务器从未见到的源，如 URL 片段、`window.name`、`postMessage` 与先前缓存内容。

与 `ATTACK-CLASSES.md` 一起使用。本文件覆盖浏览器源与汇点、源边界、浏览器持久化与跨站状态 oracle。对 webview 桥的原生侧使用 `DESKTOP-MOBILE-AND-LOCAL-IPC.md`，对服务器侧 CSRF、会话与认证回调使用 `WEB-PROTOCOL-AND-AUTH.md`。

## 核心纪律（包含在本域每个 Agent 提示中）

```
- A client-side candidate needs a controllable source and an executing or disclosing sink. Name both and show attacker-influenced data reaching the sink.
- The impact must reach a victim's session, another origin, or shared persistence. Self-injection and disclosure of the attacker's own data are not findings.
- Framework escaping, browser same-origin policy, CSP, COOP/CORP, service-worker scope, and modern noopener defaults are real controls. Verify them before assigning impact.
- Browser storage and caches are shared by origin and may outlive login state. Identify who writes, who reads, and which account, tenant, or worker lifecycle clears each record.
- Use `confirmed` only for complete source evidence plus bounded local browser tests. Use `needs_validation` when renderer, extension permission, deployed header, or browser-policy behavior is required but unavailable.
```

## DOM 与对象状态攻击类（subagent_type: `general`）

**基于 DOM 的 XSS**
将 `location` 字段、`document.referrer`、`window.name`、消息数据、存储与浏览器控制的文档状态追踪到 `innerHTML`、`outerHTML`、`document.write`、字符串求值 API、可执行 URL、jQuery HTML API 或框架逃逸口。被框架转义的插值不是 finding。

**DOM clobbering**
攻击者注入的 `id` 或 `name` 属性遮蔽代码随后信任的全局、表单属性、配置对象或初始化标志。要求同时有保留该属性的标记路径与对 clobber 值的安全相关使用。

**原型污染与 gadget 链**
攻击者控制的键到达深层合并或路径赋值等递归写入并修改原型状态。然后可达 gadget 消费被污染属性以改变授权、执行、导航或渲染。仅有 `JSON.parse`、浅拷贝或无 gadget 的污染不够。

## 跨源消息与网络攻击类（subagent_type: `general`）

**`postMessage` 源与来源信任**
处理程序在无精确源 allowlist（以及多帧共享源时预期的 `event.source`）下用 `event.data` 执行敏感动作。在发送侧，发往 `*` 的敏感数据到达非预期嵌入者。弱子串、前缀、后缀或未锚定正则源匹配不是源检查。

**跨站 WebSocket 请求使用**
WebSocket 升级在无 `Origin` 检查或通道特定令牌下接受来自不受信任源的环境 cookie，允许受害者会话读或变更数据。同时确认升级行为与安全相关消息处理程序。

**带凭据的 CORS 信任**
服务器在允许凭据时反射或弱匹配 `Origin`，并返回敏感响应。带凭据的裸通配被浏览器拒绝；仅报告实际反射/允许的源路径与跨源数据或变更。

## Service worker 与浏览器存储攻击类（subagent_type: `general`）

**Service worker 注册与范围接管**
攻击者可影响内容可成为注册的 worker 脚本、控制接收过宽 `Service-Worker-Allowed` 范围的路径，或在无完整性控制下改变更新导入。验证最终脚本 URL、响应 MIME 类型、源、范围，以及谁控制每个导入脚本。具有预期范围的正常同源 worker 不是缺陷。

**Service worker 缓存与身份混淆**
Worker 在策略中未包含账户、租户、授权状态或请求模式的情况下缓存个性化响应，然后在账户切换或注销后提供它们。审查 fetch 事件路由、缓存名与键、导航回退、缓存清理，以及错误/离线路径是否返回另一用户的先前响应。

**浏览器存储披露与过时授权**
令牌、私有响应、草稿数据或授权决策保留在 `localStorage`、`sessionStorage`、IndexedDB、Cache Storage、扩展存储或客户端状态中，并可被另一账户或较低信任同源组件读取。仅存储令牌不是 finding；要求具有较少权威的现实读者，或撤销/注销后的继续使用。

**跨上下文存储与广播混淆**
`storage` 事件、`BroadcastChannel`、共享 worker 或源范围缓存在标签间携带身份或命令而不绑定到当前会话。检查账户切换、私有/公共窗口、租户变更，以及可覆盖较新认证状态的过时标签。

## 跨站信息泄漏类（subagent_type: `general`）

**XS-Leaks 与跨源状态 oracle**
攻击者页面可通过资源加载/错误事件、帧或窗口状态、重定向行为、时序、缓存状态或响应大小区分受保护跨源状态，同时浏览器附加受害者凭据。要求一个具体含秘密谓词，如私有对象、角色或账户是否存在。通用时序方差或公共资源可用性不是 finding。

**窗口与 opener 状态披露**
跨源窗口的允许元数据或导航结果揭示受保护状态，或保留的 opener/命名窗口关系让攻击者控制页面影响特权导航。检查 COOP、帧保护、`noopener`、精确源，以及可观察状态是否机密。

## UI 欺骗与导航攻击类（subagent_type: `general`）

**点击劫持**
被框定的状态变更动作缺少有效 `frame-ancestors`、`X-Frame-Options` 或等价 UI 隔离。要求敏感动作并确认它可在框定状态下完成；只读内容上缺失头是 hardening note。

**客户端导航混淆**
客户端源在无 scheme 与目的地策略下控制重定向或导航，包括可执行 `javascript:` 或 `data:` 目的地。反向 tabnabbing 仅适用于代码显式保留 `window.opener`、使用无隔离的 `window.open`，或支持无隐式 `noopener` 的浏览器。

## 通用动作（适用于上述）

- 从 DOM、导航、worker、消息与存储汇点开始，然后向后追踪到仅浏览器与服务器控制的源。记录应停止该路径的浏览器策略。
- 用本地测试源与虚拟账户测试账户切换、注销、worker 更新、离线回退与过时标签状态。不要使用生产用户、源或共享服务。
- 对 XS-Leaks，仅列出由源码与本地浏览器行为证明的谓词。然后识别会移除 oracle 的响应头或渲染选择。

## 验证规则（在此报告任何 finding 前应用）

1. 引用源、汇点、浏览器策略、受影响源/会话与可观察变更或披露。
2. 对原型污染，证明递归写入与安全相关 gadget。对 DOM clobbering，证明标记存活且被遮蔽值被使用。
3. 对 service worker 与存储，证明生命周期可达性：攻击者控制的写入或缓存条目必须到达不同账户、租户或后续授权状态。
4. 对消息、CORS、WebSocket 与 XS-Leaks，显示精确源/来源验证与暴露的受保护状态或动作。确认 CSP、COOP/CORP、cookie 与 SameSite 策略尚未阻止它。
5. 仅以完整客户端路径与有界本地证据返回 `confirmed` finding。以所有者必须验证的精确已部署头、扩展权限、浏览器版本或渲染器行为返回 `needs_validation`。
