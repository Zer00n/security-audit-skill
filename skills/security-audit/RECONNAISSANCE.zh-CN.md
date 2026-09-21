# 侦察

> 本文为英文原文 [`RECONNAISSANCE.md`](RECONNAISSANCE.md) 的简体中文译本。

### 第 1 阶段：映射源码并规划覆盖

Parent 初始化 `run-metadata.json`，应用 `SKILL.md` 中的严格侦察前预算门，然后在狩猎前创建 Agent scratch 根与共享 ledger。若门失败，在元数据中记录 incomplete 状态且不启动侦察 Agent。侦察仅读取目标与本地可用的构建/配置状态。它不接触已部署端点、外部身份提供商、注册表、broker、云 API 或其他共享服务。

并行启动若干 `research` Agent。它们向 parent 返回结构化事实，不写文件。

**Agent 1a：产品、技术栈与本地运行**

```text
Read the target at <target>. Do not use network access. Return:
1. Product type, users, operators, and ordinary trust-sensitive actions.
2. Languages, frameworks, build system, runtimes, and locally visible deployment models.
3. Repository-relative entry points and subsystem boundaries.
4. Exact build and test commands that could run offline with local dependencies, their expected write locations, and the target-controlled inputs they process. Do not run them during reconnaissance.
5. Comparable software or protocol visible from local documentation and dependencies. If no useful comparison is source-grounded, say so.
6. Missing local toolchains or runtime facts that limit bounded execution.
Return only source facts with repository-relative file:line references.
```

**Agent 1b：主体、权威与控制**

```text
Read all source that establishes identity, authorization, isolation, and privilege. Map:
1. Each lower-trust principal and the actions it has by design.
2. Authentication or peer identity at each entry surface.
3. Per-resource authorization and tenant/owner scope.
4. Process, browser, workload, CI, plugin, model/tool, device, or local-IPC authority.
5. Privilege changes, confirmation, revocation, recovery, and fallback paths.
6. Which controls are source-visible and which depend on an unobserved deployment fact.
Return trust boundaries and control locations with repository-relative file:line references. Do not infer live reachability.
```

**Agent 1c：入口面、副本与汇点**

```text
Inventory every source-visible place external or lower-trust input enters:
- HTTP/browser, RPC/message/protocol, files/archive/document, CLI/env/config, plugins/dependencies/CI, cloud events/IAM selectors, model context/tool arguments, mobile/deep-link/webview, and local IPC.
For each surface, follow major transformations, stored or derived copies, and security-relevant sinks. Record source-visible limits and parallel paths to the same effect.
Return repository-relative paths and line numbers. Be complete, but do not execute or send inputs.
```

**Agent 1d：本地执行与部署可见性**

```text
Read tests, build definitions, manifests, packaging, and maintained environment overlays. Return:
1. Small offline tests or existing fixtures that could validate trust boundaries with dummy data inside the required OS-enforced sandbox.
2. Processes that could use an isolated loopback network namespace without external or shared dependencies.
3. Commands that would fetch dependencies, publish artifacts, contact paid/provider APIs, or affect shared state; mark them prohibited for this run.
4. Deployed controls and attachments that source cannot establish and therefore require needs_validation if decisive.
5. The final active source path for each deployment mode only where the repository selects it deterministically.
6. Whether the local platform can enforce an empty allowlisted environment, no external network, read-only target/toolchain mounts, scratch-only writes, and explicit CPU, memory, process, file-size, disk, and wall-clock limits. Missing controls block target-controlled execution.
7. Whether trusted parent-side code can promote predeclared scratch files with path-confined no-follow descriptor traversal, nonblocking regular-file checks, no-follow traversal of every destination parent, exclusive regular-file destination creation, and explicit per-file and cumulative size bounds. Missing promotion controls block use of scratch files as evidence.
```

为这四个 Agent 未映射的、实质不同的部署模式或 subsystem 增加聚焦侦察 Agent。不要静默省略：若 `SKILL.md` 中的预算门阻止聚焦 Agent，则不为其启动任何东西，将未映射区域种子为带原因 `budget_cannot_reserve_critics_and_validation` 的 `deferred` ledger 单元，并在报告中披露缺口。

## 既有运行输入

在选择工作前，parent 阅读同一仓库每个可用的既有 `coverage-ledger.json` 与 `findings.json`：

- 将每条既有记录与单元的源码位置、控制、条件与源码派生身份与当前源码比较。
- 仅当相关源码、条件与合格证据仍适用时，将未变的既有 `confirmed` 记录以相同 fingerprint 带入当前候选集。将其链接到带 `prior_status: "prior_confirmed_same_source"` 的当前 `planned` 单元，并仅将该根因放入 hunter 排除列表。复核携带记录的第 3 阶段 verifier 成为该单元的 assignment owner；其源码复核是该单元的首次检查，并将单元移至带携带 fingerprint 的 `candidate`。
- 当任何相关源码或条件变更时，构建当前 planned `prior_confirmed_changed_source` 再验证单元。不要将该根因从狩猎排除，或假设既有判定仍适用。
- 为每个既有 `needs_validation`、`deferred`、`blocked`、`out_of_scope` 与变更源码单元构建当前工作单元。这些状态是优先级输入，绝不是去重或抑制键。
- 仅在当前源码支持其追踪后，以相同 fingerprint 携带仍受阻的既有 `needs_validation` 记录。将其链接到带 `prior_status: "prior_needs_validation"` 的当前 `planned` 单元；记录保留未决阻塞项。复核携带记录的第 3 阶段 verifier 成为该单元的 assignment owner；其复核是该单元的首次检查，并将单元移至带携带 fingerprint 的 `candidate`。将记录纳入最终核验。
- 除非当前证据改变失败追踪或缺失条件，否则将既有 rejected 记录视为过时主张。未变的拒绝仅抑制该精确主张，不抑制覆盖单元的审查。
- 记录缺失或不兼容的 ledger，而不是将其当作空覆盖。

在 `run-metadata.json` 中陈述所用路径与 source ref。仅在 `architecture.md` 中总结覆盖后果。

## 架构摘要与 companion 选择

Parent 综合 `<output-dir>/architecture.md`，硬上限约 1,000 词。包含：

1. 产品、主体、正常权威与受保护资源。
2. 来自 Agent 1a 的可比软件基线（当有源码依据时）：可比物接受哪些安全权衡。用它校准工作量与严重级别，从不用于否定已证明 finding；若可比物共享实践中重要的缺陷模式，则加强 finding。无有意义可比物时省略此行。
3. 技术栈、源码可见部署路径与离线构建/测试限制。
4. 入口面与重要的源到汇点或生命周期路径。
5. 信任边界与每个边界上最强的源码可见控制。
6. 仓库相对起始路径。
7. 既有覆盖缺口、变更源码与受阻再验证目标，以及同源 confirmed 排除。
8. 源自 [ATTACK-CLASSES.md](ATTACK-CLASSES.md) 的简短 companion 选择摘要：所选文件与需要它们的源码可见边界。

将分配级 ordinary 块、所选 companion 块与带原因的排除块放在每个 ledger 单元中，而非 `architecture.md`。这使大型运行的架构上限有效，并使精确 hunter 提示映射可机器检查。

不要仅因语言或依赖名出现就选择 companion 文件。因为侦察发现了其「When to use this file」节所描述的信任敏感边界才选择。不要仅因另一 Agent 将审查相关类就排除可见边界。

## 确定性覆盖 ledger

Parent 将 `<output-dir>/coverage-ledger.json` 写为顶层 JSON 数组。在运行 profile 设定的粒度上，为入口面、信任边界、subsystem 与适用 ordinary/companion 攻击类的每个实质组合派生一个单元（`quick` 使用一个全范围 subsystem 身份；`deep` 增加 lifecycle mode）。对 scoped 运行，为分配种子范围内 surface，并将发现的排除 surface 保留为 `out_of_scope` 单元，以便后续完整运行可将其变为当前工作。

每个维度有人类标签与 `canonical_refs` 中的稳定源码派生值。即使显示标签变化，同一源码对象也使用相同规范引用。合适引用包括仓库相对入口路径加导出范围、源码定义的路由或消息身份、定义边界的源码控制、仓库包路径，以及精确攻击类块引用。块引用是 `FILE.md#` 加上该文件中以粗体或标题书写的精确类名 — 对照文件文本匹配的稳定标识符，而非渲染的 HTML 锚点。对 companion 节块，使用任何括号限定前的标题文本（例如 `Core discipline`）。不要通过将显示标签小写或 slug 化来派生引用。

无损失 slug 地派生 `coverage_id`：

1. 要求每个引用为 Unicode NFC、有效标量值、可见内容，无控制、格式、行/段分隔符或默认可忽略码点，且无周围空白。
2. 用 RFC 3986 百分号编码其 UTF-8 字节：仅保留 `A-Z a-z 0-9 - . _ ~` 不转义，其余字节使用大写 `%HH`。
3. 用 `::` 连接编码后的 `surface`、`boundary`、`subsystem` 与 `attack_class` 引用；存在时追加编码的 `lifecycle`。

对 quick profile 的粗化 subsystem 维度使用固定规范值 `profile/quick/all-in-scope-subsystems`。不要在引用或 ID 中包含波次号、Agent、判定、严重级别或行号。每次分配前按 `coverage_id` 字典序排序单元。对每个重复 ID 失败。若重复 ID 有不同语义字段，视为规范身份冲突；切勿合并或静默覆盖。校验器也拒绝由不同规范引用表示的同一语义元组。

每个单元记录：

```json
{
  "coverage_id": "...",
  "canonical_refs": {
    "surface": "src/router.ts#POST /users/:id",
    "boundary": "src/authz.ts#requireOwner",
    "subsystem": "packages/api",
    "attack_class": "ATTACK-CLASSES.md#Access control"
  },
  "surface": "...",
  "boundary": "...",
  "subsystem": "...",
  "attack_class": "...",
  "starting_paths": ["repo/relative/path"],
  "ordinary_attack_class_block": "ATTACK-CLASSES.md#Access control",
  "selected_companion_blocks": ["FILE.md#section"],
  "excluded_blocks": [{"block": "FILE.md#section", "reason": "..."}],
  "prior_status": "new|prior_confirmed_same_source|prior_confirmed_changed_source|prior_needs_validation|prior_deferred|prior_blocked|prior_out_of_scope|prior_covered_same_source|prior_covered_changed_source|prior_rejected_claim_changed|none",
  "attempts": [],
  "wave": 1,
  "status": "planned",
  "agent_id": null,
  "reviewed_paths": [],
  "local_checks": [],
  "result_fingerprints": [],
  "unresolved": []
}
```

当 `lifecycle` 实质时，同时添加 `canonical_refs.lifecycle` 与人类 `lifecycle` 字段。仅当无 ordinary 块适用时 `ordinary_attack_class_block` 为 null。所选 companion 列表包含每个适用类及其 companion 的 `Core discipline`、`Universal moves` 与 `Validation rules`；`excluded_blocks` 记录每个考虑但未选的块及排除它的源码事实。

Parent 可添加簿记字段，但保持上述语义字段稳定。在 `prior_status` 中，`new` 标记在存在兼容既有 ledger 时本运行首次见到的 surface；`none` 标记在无兼容既有 ledger 时种子的单元。既有 `deferred`、`blocked`、`out_of_scope` 与变更源码单元在现已范围内时初始化为当前 `planned` 工作。既有同源 covered 单元仍出现在当前 ledger；优先分配变更源码、重要生命周期路径与精确冲突，然后用 coverage critic 决定是否需要再一遍。

`attempts` 是 coverage critic 重新打开的含证据分配的仅追加归档。再分配前，追加该单元先前的确切 `wave`、`status`、`agent_id`、`reviewed_paths`、`local_checks`、`result_fingerprints` 与 `unresolved`，以及 critic 有源码依据的 `reassignment_reason`。仅 `blocked`、`covered` 与 `candidate` 状态可归档。归档尝试保留与活动单元相同的状态与证据不变量，使用严格递增且低于当前波次的波次，并保留其生产 owner 与产物。下一次分配递增 `wave`，使用新 owner，并以空活动证据开始。若 profile 或预算阻止另一次分配，递增 `wave` 并使用带 null owner、空证据与停止原因的活动 `deferred` 状态。切勿将归档 owner 的检查或产物复制到活动状态。后续活动终态仅包含新尝试的证据；归档保持不变。

严格强制此状态表：

| Status | Unit `agent_id` | `reviewed_paths` / `local_checks` | `result_fingerprints` | `unresolved` |
|---|---|---|---|---|
| `planned` | null | empty | empty | empty |
| `not_applicable`, `out_of_scope`, `deferred` | null | empty | empty | nonempty reason |
| `in_progress` | canonical owner | empty | empty | empty |
| `blocked` | canonical owner | both nonempty owned partial evidence | empty | nonempty blocker |
| `covered` | canonical owner | both nonempty | empty | empty |
| `candidate` | canonical owner | both nonempty | nonempty | optional |

规范 agent ID 匹配 `^[a-z0-9][a-z0-9_-]{0,63}$` 且不是 Windows 设备名。小写强制，因此一个 ledger 不能含大小写折叠别名。单元 `agent_id` 记录 assignment owner。每次检查记录自己的 `agent_id` 与非空 `reviewed_paths`；单元级 `reviewed_paths` 恰好是它们的并集。仅源码检查使用 `artifact: null`。本地检查要求仅由受信任 parent 侧代码提升到恰好 `agents/<check.agent_id>/artifacts/` 下的普通文件。这使 hunter 与 verifier 检查可共存于一个单元。Scratch 路径、输出根文件、符号链接、特殊文件与另一检查 owner 的产物不是证据。

Ledger 即覆盖主张。架构摘要、Agent 数量或通用「已审查 auth」句子不是覆盖证据。第 2 阶段仅从 hunter 结构化结果中的路径与检查关闭单元。

种子后、每次 parent 更新后以及第 6 阶段前运行 `node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json`。校验器拒绝超过 5 MiB、64 层嵌套、10,000 单元、嵌套集合 1,000 条目或 500,000 遍历值的输入，并将报告的校验错误上限为 100。实践中 5 MiB 字节限制大约容纳 2,000–5,000 个现实单元，因此在 10,000 单元上限前绑定。在分配工作或提出覆盖主张前修复每个错误。
