# security-audit

> 本文为英文原文 [`README.md`](README.md) 的简体中文译本。

[English](README.md) · [最佳实践手册](docs/最佳实践手册.md)

面向编码 Agent 的安全审计技能：将 Agent 编排为安全审计员，通过侦察、覆盖驱动狩猎、候选验证、结构化输出、独立记录核验与目标中性报告等阶段协作完成审计。

本技能是 Cloudflare 漏洞发现 harness 的起点，见 [Build your own vulnerability harness](https://blog.cloudflare.com/build-your-own-vulnerability-harness)。该 harness 后来扩展为多阶段、舰队级系统；本仓库是其演进前的单仓库起点。

## 它做什么

该技能按六个阶段运行结构化审计：

1. **侦察（Reconnaissance）** — 在 `architecture.md` 与 `coverage-ledger.json` 中映射架构、信任边界、输入面、既有证据与确定性覆盖。
2. **覆盖驱动狩猎（Coverage-led hunting）** — 按 ledger 单元分配隔离的 hunter，记录检查，并用 coverage critic 发现缺口。
3. **候选验证（Candidate validation）** — 将每个唯一候选交给新的 verifier，由其尝试证伪。
4. **结构化输出（Structured output）** — 将 `confirmed`、`needs_validation`、`rejected` 记录写入 `findings.json`，并对照 `report-schema.json` 校验。
5. **独立记录核验（Independent record verification）** — 新 Agent 核验最终源码主张；实质性替换需再经独立 verifier。
6. **目标中性报告（Target-neutral reporting）** — 由已核验记录与 coverage ledger 派生 `REPORT.md`、`FINDINGS-DETAIL.md`、`NEEDS-VALIDATION.md`。

父 Agent（parent）在创建 ledger 之后以及每次后续更新后运行 `validate-coverage-ledger.cjs`；在第 4 阶段以及第 5 阶段每次替换后运行 `validate-findings.cjs`。

判定彼此独立：`confirmed` 需完整源码追踪与有界可观察结果；`needs_validation` 需精确未决事实且无严重级别；`rejected` 记录已被证伪的候选。

对同一仓库的多次运行是累加的。技能利用既有 ledger 与 findings 定位缺口、对变更源码再验证，并在不把过时或未解决工作当作已覆盖的前提下继承当前源码证据。

## 文件

| 文件 | 用途 |
|------|------|
| `SKILL.md` | 设置、核心原则、平台术语、工作流概览与审计反模式 |
| `RECONNAISSANCE.md` | 第 1 阶段侦察提示与综合说明 |
| `HUNTING.md` | 第 2 阶段编排、狩猎方法与验证规则 |
| `ATTACK-CLASSES.md` | 核心、wildcard 与 obvious-things 攻击类提示 |
| `MEMORY-SAFETY-AND-BINARY.md` | 面向原生目标的内存安全、二进制与内核狩猎类 |
| `AI-AND-LLM.md` | 面向 LLM 目标的提示注入、Agent/工具与输出处理狩猎类 |
| `WEB-PROTOCOL-AND-AUTH.md` | 面向 HTTP 协议与认证目标的请求分帧、缓存与认证协议狩猎类 |
| `CLIENT-SIDE.md` | 面向客户端/浏览器目标的 DOM 注入、消息信任、UI 欺骗与原型污染狩猎类 |
| `SUPPLY-CHAIN-AND-RELEASE.md` | 依赖、CI、发布、签名、更新、插件与扩展狩猎类 |
| `CLOUD-AND-DEPLOYMENT.md` | IAM、基础设施即代码、容器、无服务器、入口与运行时配置狩猎类 |
| `PROTOCOLS-RPC-AND-MESSAGING.md` | RPC、序列化、队列、broker、webhook 与流式协议狩猎类 |
| `RESOURCE-EXHAUSTION-AND-AVAILABILITY.md` | 共享资源、配额、队列、worker 与运营方成本狩猎类 |
| `DATA-ISOLATION-AND-LIFECYCLE.md` | 租户隔离、缓存、搜索、导出、备份、迁移、删除与恢复狩猎类 |
| `DESKTOP-MOBILE-AND-LOCAL-IPC.md` | 原生应用、深度链接、webview、导出组件、helper、daemon 与本地 IPC 狩猎类 |
| `VALIDATION-AND-REPORTING.md` | 第 3–6 阶段：候选验证、结构化输出、记录核验与报告 |
| `report-schema.json` | `findings.json` 三种判定的 JSON schema |
| `validate-findings.cjs` | 第 4、5 阶段零依赖的 `findings.json` 校验器 |
| `validate-findings.test.cjs` | findings 校验器测试与生产者兼容夹具检查 |
| `validate-coverage-ledger.cjs` | 第 1–5 阶段零依赖的 `coverage-ledger.json` 校验器 |
| `validate-coverage-ledger.test.cjs` | coverage-ledger 校验器测试 |

上述 Markdown 文档的简体中文译本为同目录 `*.zh-CN.md` 旁路文件。合成的中文最佳实践见 [`docs/最佳实践手册.md`](docs/最佳实践手册.md)。

## 安装

使用 [Skills CLI](https://skills.sh) 安装本技能：

```bash
npx skills add https://github.com/cloudflare/security-audit-skill \
  --skill security-audit
```

用户级安装使用 `--global`：

```bash
npx skills add https://github.com/cloudflare/security-audit-skill \
  --skill security-audit \
  --global
```

运行 `npx skills --help` 查看 Agent 选择与非交互选项。

## 用法

在目标代码库中（或指向该代码库）启动编码 Agent，然后请求安全审计：

```
security audit this codebase
```

```
find security vulnerabilities in ./src
```

```
do a security review, output to ~/audits/my-project
```

当请求匹配触发条件（安全审计、查找漏洞、对代码做 pen-test 等）时技能会自动激活。直接的代码库审计或 pen-test 请求使用完整审计模式（full audit mode）。安全问题与聚焦漏洞工作使用指导模式（guidance mode），除非你明确要求报告产物。完整审计模式下，未指定输出目录时默认写入 `~/security-audit-skill/<repo-name>/run-<N>`。仅当你明确选择版本控制忽略的目录时，工作流才会写入目标仓库内部。

## 要求

- 支持工具调用与并行子 Agent 的编码 Agent 与模型
- 用于零依赖 findings / coverage-ledger 校验器的 Node.js
- 用于目标可控构建、测试、进程、浏览器、模拟器、fuzzer 与夹具的 OS 强制沙箱。必须禁用外部网络、使用经净化的 allowlist 环境、强制资源限制，且仅允许写入分配的 scratch 路径。缺少这些控制时，工作流将线索保留为 `needs_validation`，而非执行目标代码。

## 设计原则

- **仅确认已确立的边界失败。** 有源码依据但受阻的线索保留为带精确未决事实的 `needs_validation`。
- **对抗式验证。** 检查 finding 的 Agent 绝不是发现它的 Agent。
- **严重级别需要影响。** 可能性 × 影响，而非偏离检查清单。
- **纵深防御缺口不是漏洞。** 若 Layer A 已阻止攻击，缺少 Layer B 只是 hardening note。
- **多次运行提升覆盖。** 在测试运行中，单次运行大约只发现重复运行合计漏洞的一半。

## 联系

关于 AI 驱动安全工具的问题、反馈或交流：security-ai-research@cloudflare.com

## 许可证

MIT — 见 [LICENSE](LICENSE)。
