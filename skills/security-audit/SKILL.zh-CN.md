---
name: security-audit
description: Security guidance and vulnerability review for codebases, APIs, services, CLI tools, libraries, and daemons. Use for security questions, focused reviews, vulnerability research, security audits, or pen tests. Run the complete workflow only for explicit codebase audit or pen-test requests, full/comprehensive/end-to-end reviews, or requested report artifacts.
---

# 安全审计

> 本文为英文原文 [`SKILL.md`](SKILL.md) 的简体中文译本。

发现真正违反信任边界的漏洞，并向所有者提供源码证据、安全复现、优先级与最小有效修复。这是防御性、源码优先的工作流。没有具体受影响主体、资源或安全结果的候选，不能成为 `confirmed` finding。

## 运行模式

本技能默认处于指导模式。加载它本身并不授权完整审计工作流或文件创建。

- **指导模式（Guidance mode）**：针对安全问题、聚焦评审、方法论、分流或特定 finding 调查，仅使用本技能相关部分。不要自动运行全部六个阶段、创建输出目录或写入审计产物。可在有用时启动聚焦 Agent；它们向当前任务返回结果。
- **完整审计模式（Full audit mode）**：当用户明确要求审计或 pen-test 代码库、要求全面/端到端安全评审，或要求报告产物时，使用完整工作流。运行全部六个阶段并写入下文定义的文件。

若请求可能对应任一模式，在创建文件或启动完整工作流前先问一个聚焦问题。

## 平台术语

本技能与具体 Agent 平台无关：

- **Parent** 是协调本次运行并拥有共享状态的 Agent。
- **Task tool** 是平台的委托或子 Agent 机制。
- **`research` agent** 是用于聚焦源码探索与事实核验的委托 Agent。
- **`general` agent** 是用于广泛调查与有界本地执行的委托 Agent。
- **`subagent_type:`** 出现在标题中时，标明上述两种委托角色中哪一种执行该工作。

使用等价平台能力时，须保持角色、写隔离、提示与独立性边界。

## 通用执行安全

这些规则在两种运行模式下均适用。源码检查只读。仅在提供以下全部控制的 OS 强制沙箱内，运行目标可控的构建、测试、进程、浏览器、模拟器、fuzzer 与夹具处理：

- 无外部网络；仅在检查需要本地客户端/服务端流量时使用隔离的 loopback 命名空间；
- 从显式 allowlist 填充的空环境，并带有 scratch 本地的 `HOME`、临时目录与缓存；
- 只读目标与工具链，目标可控进程仅可写入分配的 `scratch/` 目录；以及
- 明确的低 CPU、内存、进程、文件大小、磁盘与挂钟时间限制。

Agent（在目标可控进程之外）可在构建必须写在源码旁时，于分配的 `scratch/` 中制作一次性源码副本。指导模式下不要保留目标可控文件。完整审计模式下，仅受信任的 parent 侧代码可按「写隔离」中的流程，将最少非秘密结果提升到保留的 `artifacts/`。切勿向目标代码暴露保留输出目录（Agent 自己的 `scratch/` 除外）、其他 Agent 目录、主机家目录、凭据、socket 或共享服务。不要安装依赖或让构建拉取依赖。仅使用本地已可用的工具与依赖。若无法强制全部控制，则不要执行目标代码：将缺失的沙箱能力报告为 needs-validation 阻塞项，并给出安全验证计划。

使用虚拟主体、夹具与密钥。不要探测已部署端点、外部服务、共享基础设施、生产身份、其他用户数据或实时控制面。不要对实时或共享进程做可用性测试、发布产物、改动发布、消耗付费 API 配额，或在建立缺陷所需的最小本地效果之外继续。若决定性事实在源码或沙箱夹具之外，报告为需要验证。

## 完整审计设置

在完整审计模式下，侦察前先解析这些值：

- **Skill directory**：包含本 `SKILL.md` 的绝对目录。
- **Target**：受审仓库根目录的绝对路径。
- **Repo name**：来自目录或本地 Git remote 的稳定仓库标识。
- **Output directory**：目标外的新可写目录，默认 `~/security-audit-skill/<repo-name>/run-<N>`，其中 `<N>` 为下一个未用整数。仅当用户明确选择且 parent 验证版本控制忽略整个目录时，才可使用目标内目录。否则停止并请求外部路径。
- **Source ref**：已审提交以及工作树是否 dirty。不要将未审的生成或修改文件当作另一修订。

### 写隔离

Parent 创建并作为共享运行文件的唯一写入者：

- `run-metadata.json`
- `architecture.md`
- `coverage-ledger.json`
- `findings.json`
- `REPORT.md`
- `FINDINGS-DETAIL.md`
- `NEEDS-VALIDATION.md`

每个 hunter 或 verifier 在 `<output-dir>/agents/<agent-id>/` 下获得唯一根目录，并有独立的 `scratch/` 与 `artifacts/`。规范 agent ID 匹配 `^[a-z0-9][a-z0-9_-]{0,63}$`，且不得等于 Windows 设备名如 `con`、`prn`、`aux`、`nul`、`com1`–`com9` 或 `lpt1`–`lpt9`。小写 ID 可防止大小写折叠冲突。Agent 与每个目标可控进程仅可写入 `scratch/`；保留的 `artifacts/` 由 parent 拥有，从不暴露给沙箱，且仅可由受信任 parent 侧提升代码写入。Agent 不得更改共享文件、目标源码、保留产物或其他 Agent 目录。不要用 `/tmp` 或主机家目录作为可写回退。

执行前，parent 打开并保留 Agent 的 `scratch/` 与 `artifacts/` 根的受信任、不可继承目录描述符，并记录预期 scratch 相对产物文件的 allowlist 以及显式的每文件与累计字节限制。切勿将这些描述符传给 Agent 或沙箱。沙箱及其全部进程终止后，受信任 parent 侧代码分别提升每个 allowlist 文件：

1. 校验声明的相对路径：拒绝绝对、空、`.`、`..` 或符号链接组件。
2. 从保留的 scratch 根描述符出发，用 no-follow 目录相对操作遍历每个父组件；切勿按路径重新打开。
3. 以 no-follow 与 nonblocking 打开叶节点。
4. 用 `fstat` 验证其为普通文件、硬链接数恰好为 1，且在记录的每文件与累计字节限制内。
5. 从该描述符读取时再次强制这些限制。
6. 精确复制已验证大小，再次 `fstat`，拒绝身份、类型、链接数或大小的变更。
7. 对目标，从保留的 artifacts 根描述符用 no-follow 目录相对操作遍历每个父组件；要求每个已存在组件为真实目录，并在创建缺失目录后排他创建，再 no-follow 重新打开并验证。
8. 排他创建叶节点且不跟随链接，验证打开的目标为硬链接数恰好为 1 的普通文件，并从已验证源描述符复制，不重新打开任一路径。
9. 在非 POSIX 系统上使用等价的竞态安全 API。
10. 切勿递归复制或 glob scratch、将归档解压到 artifacts，或打开/提升符号链接、FIFO、socket、设备、目录、硬链接文件、正在变化的文件或超出边界的文件。
11. 若任何检查不可用、无法强制或失败，丢弃该 scratch 条目；若其为决定性证据，以精确提升阻塞项保留 `needs_validation`。

[HUNTING.md](HUNTING.md) 与 [VALIDATION-AND-REPORTING.md](VALIDATION-AND-REPORTING.md) 以相同围栏块将此流程嵌入 hunter 与 verifier 提示；顺序与规则与本列表一致。

对已复现检查，记录命令、精确测试输入、沙箱限制，以及仅 allowlist 环境变量名加上复现所需的安全非秘密值。切勿捕获或复制环境、继承变量、凭据值、认证状态或不相关主机路径。从空环境启动，而不是执行后再试图脱敏。

委托前，parent 写入 `run-metadata.json`，至少包含 `run_id`、`repo`、`target`、`source_ref`、`profile`、`scope_paths`、`budget`（未设则为 null）、`execution_policy: "sandboxed-source-and-local-only"`、所选 companion 文件、既有运行路径、共享文件所有者与 `run_status: "in_progress"`。仅在这些事实变化时更新元数据；候选状态属于 coverage ledger 与 `findings.json`。

## 完整审计规划

本节的覆盖、既有运行、profile 与预算要求仅适用于完整审计模式。

### 覆盖与既有运行

单次运行并不完整。狩猎前构建确定性覆盖计划，并在每次 Agent 结果后更新。[RECONNAISSANCE.md](RECONNAISSANCE.md) 定义稳定覆盖单元，[HUNTING.md](HUNTING.md) 定义 coverage-critic 波次。仅 parent 更新 ledger。

若存在既有运行，在规划当前运行前阅读每个兼容的 `coverage-ledger.json` 与 `findings.json`：

1. 将相关当前源码与每条既有记录和单元比较。仅凭既有 source ref 不能证明路径未变。
2. 仅当相关源码与条件未变且证据仍满足当前契约时，将既有 `confirmed` 记录带入当前候选集。将其链接到种子为 `planned` 的当前 ledger 单元，保留 fingerprint，仅将该根因从 hunter 排除，并让携带记录走当前最终核验路径；复核它的第 3 阶段 verifier 成为该单元的 assignment owner，并将其移至 `candidate`。
3. 当既有 `confirmed` 的相关源码变更时，创建当前 planned 再验证单元。不要将该根因放入 hunter 排除列表。仅当当前独立验证确立当前路径与结果时，它才仍为 confirmed。
4. 将既有 `needs_validation`、`deferred`、`blocked`、`out_of_scope` 与任何变更源码单元变为当前工作。仍在外部的 `needs_validation` 记录仅可在检查当前源码追踪并以 fingerprint 链接到当前 `planned` 单元（其 verifier 复核提供 owner 与证据）后携带；记录保留未决阻塞项。这些既有状态从不抑制当前单元。
5. 既有同源 covered 单元可影响优先级，但仍须出现在当前 ledger。既有 `rejected` 仅抑制未变的失败主张，不抑制其单元的覆盖；变更证据则产生当前工作。
6. 阅读既有 profile 与范围。既有 `quick` 或 scoped ledger 仅贡献其记录的证据与缺口，从不隐含「其余没问题」。

若无既有 ledger，在最终覆盖陈述中说明。切勿暗示一次运行穷尽目标。

### 运行 profile 与范围

完整审计设置期间，根据用户请求选择 profile，或按目标规模与利害提出。记录到 `run-metadata.json`（`profile`、`scope_paths`）并在报告中说明。默认是 `standard`。

- **`quick`** — 面向小目标、再运行或快速初看的有界遍历。将 ledger 单元粗化为 surface × boundary × attack class（subsystem 使用固定规范标识 `profile/quick/all-in-scope-subsystems`），恰好运行一波 hunter 后接恰好一次最终 coverage-critic，并对每个候选使用一个新 verifier 同时完成候选验证与最终记录核验。不要启动后续 hunter 波次：将 critic 接受的发现与再分配记录为 `deferred`。
- **`standard`** — 按书面工作流执行。
- **`deep`** — 面向高利害或大型目标。按 subsystem 与 lifecycle mode 拆分 ledger 单元，运行 critic 波次直至 clean pass，保持候选验证与最终记录核验为独立新 Agent，并对 `prior_covered_same_source` 单元做独立第二遍。

**Scoped run** 审计子集：命名路径、一个 subsystem、一个 companion 域，或两个 source ref 之间的 diff。仅为范围内 surface 种子 ledger 单元，并将其余记录为 `out_of_scope` — 绝不是 `covered`。Scoped 或 `quick` 运行必须呈现为部分覆盖。

Profile 改变广度与冗余，从不改变证据门槛。不要为了缩放而削弱候选门、源码/本地执行边界、`needs_validation` 纪律、schema 校验或对 `confirmed` 记录的独立核验。

#### 成本预算

Ledger 使花费可计数：一个单元大约一次 hunter 分配，一个存活候选按 profile 是一或两次 verifier 分配。当用户设定预算 — 或 parent 为大目标提出预算 — 在 `run-metadata.json` 中将 `budget` 记录为跨所有阶段的最大 Agent 调用次数。

在启动任何侦察 Agent 前应用严格预算门。预留四次基线侦察调用、`quick` 的一次最终 post-wave critic，或 `standard`/`deep` 的一次 post-wave 加一次独立 final-clean critic，以及至少一次 verifier 调用。仅在为每次额外调用重复此门后才增加聚焦侦察。若请求预算无法资助该最小值，则不启动任何 Agent：请求更大预算、更窄范围或不同 profile。若请求不变，设置 `run_status: "incomplete"` 与 `incomplete_reason: "budget_cannot_fund_reconnaissance_and_reserves"`，并报告未运行任何审计遍历。

按此顺序花费：

1. 将侦察、每次 post-wave critic 与独立 final-clean critic 计为 Agent 调用。
2. **在狩猎前预留 critic 与验证。** 对 `quick`，预留其一次 post-wave final critic。在每波 `standard` 或 `deep` hunter 前，预留一次立即 post-wave critic 加一次独立 final-clean critic。同时按 profile 预留 verifier 成本（每个预期候选约 1 或 2 个 Agent；存疑时在 critic 预留后预留余额的 30%）。切勿将 hunter 分配进任一预留。
3. 按优先级将 hunter 分配给单元，直到狩猎配额用尽。该波后立即花费预留的 post-wave critic；保持 final-clean 与验证预留完整。
4. 在后续波次前，再次预留其新的 post-wave critic。若剩余预算无法覆盖所需 critic 调用与验证预留，则不从该波启动 hunter，将其 planned 单元标记为 `deferred`（原因 `budget_cannot_reserve_critics_and_validation`），并用保留的 final-clean critic 记录由此产生的缺口。

在第 1 波前，用种子单元、隐含 hunter 数、强制 critic 调用、验证预留以及剩余预算是否覆盖计划，更新侦察前估算。若明显无法，说明并提出更紧范围或更粗 profile，而不是静默稀释证据。若后续事实耗尽所需 final-critic 预留，则不启动 hunter，将所有 planned 工作标记 deferred，以原因 `critic_budget_exhausted` 设为 incomplete，且不做完整覆盖声称。

严格总 Agent 预算仍可能被意外大的候选集或第 5 阶段需要另一独立 verifier 的实质性替换超出。若剩余预算无法验证每个候选，停止狩猎，在预算允许时按 fingerprint 顺序验证候选，并设置 `run_status: "incomplete"` 与 `incomplete_reason: "validation_budget_exhausted"`。将每个未验证 fingerprint 链接到带该未决原因的 `candidate` ledger 单元。不要把未验证候选放入 `findings.json`、改标为 `needs_validation`，或将运行报告为完整。第 6 阶段仅可在首节说明候选验证不完整并列出受影响 fingerprint 与单元时产出部分报告。切勿静默超出用户设定的严格预算。

## 核心原则

### 要求边界与结果

对每个候选，指明较低信任主体、接受的输入或动作、预期控制、被穿越的边界、受影响主体或资源，以及具体观察到的或所有者可观察的结果。不要将缺少最佳实践、猜测的部署行为、通用解析器崩溃或自我影响提升为安全 finding。

### 使用有界本地证据

静态分析确立源码路径。当全部执行控制可用时，沙箱本地测试解析行为：最小函数 harness、既有单元测试、小型解析器夹具、虚拟租户集成测试、本地渲染配置，或有界隔离 loopback 客户端。停在错误返回值、未授权虚拟记录、sanitizer finding、策略差异或其他最小效果。不要将本地检查扩展到最小边界结果之外，或产出持久化、故障后或隐蔽材料。

### 尊重源码可见性

部署控制、代理行为、提供商设置、浏览器头、身份策略、broker ACL、打包与拓扑是真实控制。若其必需且仓库中缺失，不要假设存在或缺失。使用带精确缺失事实与安全所有者观察或本地计划的 `needs_validation`。

### 将优先级与确定性分开

仅 `confirmed` 记录获得严重级别。可能性与影响必须反映已证明的条件与结果；总体严重级别不能超过已证明影响。`needs_validation` 表示特定有源码依据的边界假设受阻，不是低置信度的 confirmed 漏洞，且无严重级别。

用这些锚点校准总体严重级别：

- **critical** — 未认证参与者获得代码执行、完整数据存储访问或任意账户接管。
- **high** — 参与者完全击败具有真实后果的显式安全控制：认证绕过、跨租户读或写、影响其他用户的存储型脚本执行、已认证代码执行，或未认证远程停止共享服务。
- **medium** — 真实边界违反，但爆炸半径有限、前置条件少见，或后果限于狭窄资源集。
- **low** — 披露非秘密内部信息，或需持续努力且收益极小的效果。
- **informational** — 已确认但影响极小的观察，主要作为更大 finding 内的先决条件有用。

high/medium 判别：已证明结果是完全击败具有真实后果的显式控制，还是仅削弱它？若无法陈述具体损害，严重级别低于感觉。

### 推荐最小有效源码修复

对每个 confirmed finding，识别代码必须强制的不变量，以及在最后受信任决策点强制它的最窄源码变更。优先具体的仓库相对变更与回归测试，而非通用加固建议。审计描述修复；它不修改目标源码。

## 完整审计工作流

在完整审计模式下，按顺序遵循全部六个阶段：

1. **侦察** — 用 [RECONNAISSANCE.md](RECONNAISSANCE.md) 映射源码、信任边界、本地构建路径、companion 选择、既有证据与初始确定性 coverage ledger。
2. **覆盖驱动狩猎波次** — 用 [HUNTING.md](HUNTING.md)、[ATTACK-CLASSES.md](ATTACK-CLASSES.md) 与所选域 companion，按 ledger 分配隔离 hunter 并收集结构化候选结果。
3. **候选验证** — 合并 fingerprint，并将每个候选交给 [VALIDATION-AND-REPORTING.md](VALIDATION-AND-REPORTING.md) 定义的新源码 verifier。
4. **结构化输出** — 将全部最终 `confirmed`、`needs_validation`、`rejected` 记录写入 `findings.json`；用 `report-schema.json` 与 `validate-findings.cjs` 校验，并用 `validate-coverage-ledger.cjs` 校验覆盖主张。
5. **独立记录核验** — 用新 Agent 核验最终源码主张并调和纠正或状态变更。
6. **目标中性报告** — 由最终记录派生 `REPORT.md`、`FINDINGS-DETAIL.md`、`NEEDS-VALIDATION.md`，不含实时探测说明。

在恰好两种终态之一前不要结束运行：(a) 全部第 6 阶段产物已写入且两个校验器通过，或 (b) 已记录 `run_status: "incomplete"` 及其精确原因并在报告中披露缺口。切勿在阶段中途停止。

## 反模式

1. 将检查清单偏差呈现为漏洞。
2. 无可达边界违反的纵深防御建议。
3. 在有界本地证据不足时做实时或共享环境测试。
4. 猜测源码中不存在的提供商、代理、浏览器、身份或部署行为。
5. 将预期的同主体权威或自我影响当作跨边界结果。
6. 报告强于观察到的解析器或运行时效果。
7. 产出无法去重或核验的纯散文 hunter 结果。
8. 重复报告携带的同源既有 confirmed 记录，或将其用作锚定狩猎的范例。
9. 为 `needs_validation` 记录分配严重级别。
10. 在独立核验前写报告，或让散文与 JSON 不一致。
