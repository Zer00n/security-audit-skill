> 本文为英文原文 [`WEB-PROTOCOL-AND-AUTH.md`](WEB-PROTOCOL-AND-AUTH.md) 的简体中文译本。

# HTTP 协议与认证狩猎

#### 何时使用本文件

当目标在解析、缓存、浏览器认证或身份边界上使用 HTTP 时使用本文件：Web 应用、API、反向代理、CDN、网关、自定义 HTTP 服务器，以及实现会话、JWT、OAuth/OIDC、SAML、密码恢复、MFA、通行密钥、API 密钥或 mTLS 的服务。与 `ATTACK-CLASSES.md` 一起使用：访问控制审查问主体是否可执行操作；本文件问 HTTP 或身份层是否可混淆操作所属的主体、请求、保证级别或令牌。

从第 1 阶段挑选类。将大型目标拆分为请求分帧与缓存策略、浏览器认证、联邦身份、强认证与恢复、服务凭据与会话生命周期。位于未观察托管代理后的单服务器几乎没有可源码确认的走私面；代理或自定义解析器则多得多。

## 核心纪律（包含在本域每个 Agent 提示中）

```
- Framing and cache findings require two interpretations of the same request, response, or key. Name both components and the exact normalized value on each side.
- For every credential, find the signature or secret verification and every binding required for its role: issuer, audience, origin, RP, client, session, principal, resource, assurance, expiry, and one-time state.
- Host, Forwarded, X-Forwarded-*, Origin, Referer, redirect targets, callback state, and request-derived URLs are trust decisions. Trace each to the affected identity or response.
- A missing header, cookie attribute, MFA prompt, or rate limit is not a finding alone. Require an accepted invalid request, cross-principal impact, assurance downgrade, or credential disclosure.
- Classify `confirmed` only from complete source evidence and bounded local request/token tests. Use `needs_validation` when proxy, IdP, browser, certificate, secret, or deployed configuration is required but not visible.
```

## HTTP 分帧与缓存攻击类（subagent_type: `general`）

**请求分帧与失同步**
前端与后端对请求长度或头规范化不一致。审查多个 `Content-Length` 值、`Transfer-Encoding`、HTTP/2 或 HTTP/3 降级、头名规范化、禁止的连接头与 CR/LF 转换。确认一个组件将哪些字节分配给请求，其同伴将哪些字节分配给下一请求。

**通过未键控输入的 Web 缓存投毒**
请求值改变缓存内容或安全相关头，但不在缓存键中。比较缓存键构造与每个响应变体，包括转发的 host/scheme、所选 cookie、查询规范化、语言/设备头与授权状态。

**缓存欺骗与私有响应缓存**
缓存路由将私有动态路径当作公共静态资源，或缓存策略缺失身份与授权输入的响应。比较边缘可缓存性与应用路由解析、后缀/路径参数规范化与响应缓存指令。

**Host 与转发头信任**
不受信任的 host/代理元数据决定绝对 URL、租户路由、回调、重置链接、缓存键或授权使用的客户端地址。确认谁可提供该头，以及受信任入口是否移除客户端提供的副本。

**响应头注入**
不受信任数据以不安全控制字符或规范化到达 `Location`、`Set-Cookie`、CSP 或其他响应头。在报告前验证框架拒绝，并要求安全相关响应变更。

## 浏览器会话攻击类（subagent_type: `general`）

**普通 CSRF**
浏览器向状态变更端点发送环境凭据，该端点在无有效反 CSRF 令牌、同站请求绑定或严格 Origin/Referer 验证下接受跨站请求。清点每个 cookie 认证的变更，包括表单、类 JSON、multipart、方法覆盖与遗留路由。SameSite 仅对实际使用的 cookie 与浏览器上下文有效；登录 CSRF 与跨站子资源请求可有不同要求。

**会话固定与失效**
会话标识符在登录、账户切换、MFA 完成、冒充或其他权限变更时不轮换，或在注销、密码变更、撤销与账户禁用后仍有效。检查服务器会话、刷新令牌、签名 cookie、websocket 状态、缓存副本与回退端点。

**Cookie 范围与传输**
敏感 cookie 有过宽的 `Domain` 或 `Path`、可跨不安全传输，或与另一组件不同选择的兄弟 cookie 冲突。光秃缺失标志保持为 hardening note，除非现实较低信任源、网络位置或浏览器路径可获得或替换凭据。

## 联邦身份攻击类（subagent_type: `general`）

先确立角色。如重定向 allowlist 与代码签发等授权服务器控制不属于依赖方客户端。验证与绑定缺陷属于消费产物的组件。

**JWT 验证与声明绑定**
检查签名验证、服务器固定算法与密钥源，然后 `exp`、`nbf`、`aud` 与 `iss`。审查作为不受信任密钥选择器的 `kid`、`jku` 与 `x5u`、重复/头规范化与解码而不验证路径。另一服务的有效令牌在此无效，即使由受信任签发者签名。

**OAuth/OIDC 请求与回调绑定**
在目标为授权服务器处验证确切 `redirect_uri` 所有权；会话绑定的 `state`；适用时的 PKCE 与授权码绑定；ID 令牌签发者/受众/签名/nonce；以及多提供商流中的所选 IdP 绑定。比较初始回调、重试、移动/深度链接与账户关联路由。

**SAML 签名对象与断言绑定**
确保验证签名的元素是用作身份的元素。审查未签名/回退路径、安全 XML 解析器配置、规范化差异，以及有效窗口、受众/接收者、请求关联与重放状态等新鲜度/绑定字段。

## MFA、通行密钥与账户过渡攻击类（subagent_type: `general`）

**MFA 注册与保证降级**
注册、替换、禁用、恢复码生成、受信任设备创建与回退登录需要预期先前保证。检查有效第一因子能否在无策略要求的新鲜认证下注册或替换第二因子，以及禁用或过时因子是否停止授权会话。

**升级绑定与绕过**
成功挑战升级错误会话、账户、租户、动作或 API 请求，或替代路由省略保证检查。将挑战绑定到主体、当前会话、保证目标、需要时的操作或资源、过期与一次性完成。比较 UI、API、批处理、恢复与恢复流路径。

**WebAuthn 与通行密钥验证**
注册时，将挑战、RP ID、预期源、凭据、用户/userHandle、算法与策略要求的用户验证绑定到发起会话。认证时，验证挑战、RP/源、凭据成员、签名与预期用户在场/验证。检查账户发现与关联流中的 userHandle 或凭据到账户混淆。仅当产品将回归视为克隆信号时，签名计数器处理才有意义。

**账户关联与身份碰撞**
向现有账户添加 IdP、通行密钥、邮箱、电话、设备或外部账户必须要求当前已认证会话、新身份的已验证所有权、策略要求的升级，以及绑定到发起关联账户的回调状态。审查解绑/重绑与邀请接受路径中的已验证标识符或租户碰撞。

**密码重置与更广恢复**
恢复令牌、支持/管理恢复、备份码、设备迁移与邮箱或电话变更常成为最弱认证路径。验证令牌随机性、用户/动作绑定、过期、一次性状态、速率/记账控制、投递 URL 信任，以及先前令牌与会话的失效。仅揭示公开账户存在的不同响应不自动是安全 finding。

## API 密钥与 mTLS 攻击类（subagent_type: `general`）

**API 密钥范围与资源绑定**
密钥认证到超出其服务器侧记录授予的更广租户、资源、动作或环境，或请求参数覆盖这些绑定。审查密钥查找、前缀/完整密钥验证、可发布与密钥类型混淆、范围检查、轮换、撤销缓存与批量端点。

**API 密钥暴露与不安全传输**
密钥出现在客户端包、URL、重定向、日志、错误路径、构建产物或较低信任主体可访问的响应中。称为密钥的公开标识符不是秘密。确认密钥类型与披露获得的权威。

**mTLS 对等与应用身份混淆**
进程信任来自任何网络对等方的客户端证书身份头，验证链但将攻击者可影响的主体文本错误映射到账户，或接受错误信任域、扩展用法、受众或有效性策略的证书。当受信任代理终止 mTLS 时，验证仅该代理可连接、它移除传入身份头，且后端将净化身份绑定到请求。

**证书生命周期回退**
过期、撤销、缺失或续期失败的证书导致静默回退到仅 bearer 或匿名操作，或长生命周期池化连接在撤销后保留授权。缺失部署撤销数据使结果为 `needs_validation`；仓库内 fail-open 分支可源码确认。

## 通用动作（适用于上述）

- 对每个凭据与挑战走签发 → 存储 → 传输 → 消费 → 刷新 → 撤销。比较正常、错误、重试、迁移、遗留与账户切换路径。
- 枚举到同一身份的每扇门与到同一敏感操作的每条路由。有效策略是最弱并行路径，而非最精致 UI。
- 并排对比解析器、代理、路由器、缓存与应用规范化。对本地验证，向每个组件馈送相同有界请求夹具，而非向实时部署发送流量。
- 对恢复与关联，画账户前后图。每条边必须命名当前主体、新身份证明、所需保证、回调/会话绑定与撤销效果。

## 验证规则（在此报告任何 finding 前应用）

1. 应用源码可见性门。代理链、边缘缓存键、IdP 策略、证书信任、浏览器 cookie 行为、密钥与已部署认证模式可能在仓库外。记录精确 `needs_validation` 候选，而非断言缺失基础设施行为。
2. 对分帧与缓存 finding，命名两个组件与分歧解析/键。用有界本地夹具确认跨请求、跨用户或私有响应影响。
3. 对令牌、MFA、通行密钥、账户关联、恢复、API 密钥与 mTLS finding，引用验证行与缺失的主体/会话/资源/源/受众/动作/保证绑定。证明服务器接受无效过渡或凭据。
4. 对 CSRF，命名环境凭据、状态变更路由、接受的跨站请求形状、浏览器 cookie 策略与缺失的有效检查。只读动作与要求非环境 bearer 令牌的路由不符合。
5. 验证框架与库默认。若版本或配置未知，使用 `needs_validation`；不要将未验证的 critical 主张变为较低严重级别的 confirmed finding。
6. 仅以完整源码追踪与可观察未授权身份、状态或披露返回 `confirmed`。对 `needs_validation`，命名缺失事实与解决它的安全本地或所有者观察检查。
