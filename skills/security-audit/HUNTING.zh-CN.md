# 漏洞狩猎

> 本文为英文原文 [`HUNTING.md`](HUNTING.md) 的简体中文译本。

### 第 2 阶段：运行覆盖驱动狩猎波次

Parent 将 `planned` ledger 单元分配给 `general` Agent。使用足够聚焦的 hunter 覆盖单元，而不合并无关边界。一个 hunter 可拥有同一 subsystem 中密切相关的单元；不得因 Agent 数量限制而静默不分配单元 — 预算无法触及的单元须显式 `deferred`，原因 `budget_cannot_reserve_critics_and_validation`。

当预算或 profile 限制 hunter 数量时，按优先级分配单元，并在 ledger 中记录排序理由。排序依据：(1) 未认证或最低信任入口面先于已认证；(2) 保护最有价值资源的边界（凭据、跨租户数据、代码执行、发布权威）；(3) 既有运行缺口、再验证目标与变更源码先于同源再遍历；(4) 历史上对该目标类型产出 confirmed finding 的类优先于投机类。平局按 `coverage_id` 字典序打破，使运行保持确定性。

启动前，parent 将已分配单元改为 `in_progress`，设置规范小写 `agent_id`，并创建该 Agent 的 `scratch/` 与 parent 拥有的 `artifacts/`。Hunter 读取源码与 parent 提供的上下文，仅写入其唯一 `scratch/`，并通过 Task tool 返回一个结构化结果。它们从不写保留产物，也不编辑目标源码、`architecture.md`、`coverage-ledger.json`、`findings.json` 或其他 Agent 文件。

## 必需的 hunter 提示

每个 hunter 提示按此顺序包含以下部分：

1. 两句角色前言：hunter 的目标是在其分配单元中发现有源码依据的安全不变量失败，且必须返回恰好一个匹配本提示末尾结构化结果契约的 JSON 对象。
2. 逐字的 `architecture.md`。
3. 分配的 coverage ID、subsystem、boundary、仓库相对起始路径，以及来自 `coverage-ledger.json` 的每个单元的 assignment 块映射。
4. 精确所选块，逐字复制：来自 `ATTACK-CLASSES.md` 的每个所选 ordinary 攻击类块，以及每个所选 companion 的 `Core discipline`、每个所选攻击类小节、`Universal moves` 与 `Validation rules`。Ordinary 块自包含，不带 companion 风格的 `Core discipline`、`Universal moves` 或 `Validation rules` 节。不要只发送块或 companion 名称。
5. 显式排除的 ordinary 与 companion 块及每个排除原因。
6. 下文核心狩猎方法，后接提升流程块。
7. 下文核心验证规则。
8. 携带的同源既有 confirmed 排除，每项限于 fingerprint、标题与根因，以及本 hunter 不得重复的对等方拥有的当前 coverage ID。
9. 唯一 scratch/artifact 路径、安全 agent ID、预先声明的提升 allowlist 与字节限制，以及结构化结果契约，包括下文 Structured hunter result 块与 `report-schema.json` 的 `confirmed` 与 `needs_validation` 分支的逐字副本。

当同一路径跨越多个域时，提示可选择多个 companion 块。保持其约束在一起。范围是 hunter 的覆盖义务，不是重复排除工作的许可。若出现意外的不同边界，将其放在 `uncovered` 下返回，以便 parent 创建稳定 ledger 单元并在下一波分配。

#### 核心狩猎方法 — 包含在每个 hunter 提示中

```text
## Defensive vulnerability-finding method

Your goal is to find source-grounded security invariant failures and the smallest fix,
not to expand harm beyond the boundary result. Stay within source review and bounded local execution.
Do not contact deployed endpoints, provider APIs, registries, identity systems,
message brokers, shared services, or other users. Use local dummy data only.

READ THE CODE AT DEPTH. Follow each assigned input through parsing, identity,
authorization, normalization, state, derived copies, and the final sink. Read sibling,
legacy, batch, retry, cancellation, migration, and error paths that produce the same
effect. Compare sibling controls for equivalence, not only presence, and compare what
one component guarantees with what the next component assumes.

WORK FROM A CONCRETE INVARIANT:
1. Name the lower-trust principal and starting capability.
2. Name the accepted value, action, state transition, or resource selector.
3. Locate the control that should reject, bind, isolate, limit, or revoke it.
4. Trace the exact source path after that decision.
5. Stop at the smallest affected dummy record, wrong return value, process-integrity
   effect, or locally observable shared-resource effect.
6. State a source-level change and regression case that enforce the invariant.

DEPTH BOUND: trace only paths that can reach your assigned boundary or whose
guarantees that boundary relies on. Stop a line of investigation as soon as the
invariant is settled either way, and record the result in your structured output —
a covered, candidate, or blocked disposition, or an `uncovered` entry — instead of
continuing to search.

USE SAD PATHS AND DISAGREEMENTS. Check absent, empty, zero, negative, maximum,
over-limit, duplicate, mixed encoding, stale, revoked, reordered, concurrent,
partially migrated, failed dependency, and rollback state only where the interface
accepts them. Compare canonicalization and units at every parser or policy handoff.
For multi-step issues, treat each output as a prerequisite and do not assume a later
boundary. If any prerequisite is not established, record a blocker.

When a proposed high or critical candidate reveals a reusable root cause, search paths
owned by the assigned coverage IDs for lexical, structural, and logical variants.
Consolidate the same root cause, but establish each variant's conditions and impact
independently. Do not investigate peer-owned units. Return a variant with no current
coverage unit as `uncovered`.

USE THE NARROWEST LOCAL CHECK THAT SETTLES THE CLAIM. Target-controlled builds,
tests, processes, browsers, emulators, fuzzers, and fixture processing may run only
inside the parent-approved OS-enforced sandbox. It must disable external networking,
start from an empty allowlisted environment, expose target and tools read-only, permit
writes only to your scratch directory, and apply low CPU, memory, process, file-size,
disk, and wall-clock limits. Isolated loopback is allowed only for a local fixture.
If any control is unavailable, do not execute: return needs_validation with that exact
blocker. Prefer an existing unit test, minimal function harness, dummy-tenant service
call, small malformed fixture, deterministic race schedule, or locally rendered policy.
Do not install or fetch tools.

Record the exact input, command, limits, and minimum result. For the environment,
record only allowlisted variable names and safe non-secret values needed to reproduce
the check. Never capture the ambient environment, inherited variables, credentials,
authentication state, or unrelated host paths. The target-controlled process writes
only in scratch. After the sandbox and all its processes terminate, only trusted
parent-side code may promote predeclared scratch-relative files, following the
promotion procedure block included verbatim in this prompt. You and target code never
write retained artifacts. If promotion is unavailable or fails for decisive evidence,
return needs_validation with the exact promotion blocker.
Never stress availability, invoke a live target, use a real credential, publish an
artifact, or continue past the minimum observed effect.

A deployment, browser, provider, broker, OS, proxy, package, secret, or identity fact
outside source is not proof either way. If one such fact is decisive, return a
needs_validation record with the exact missing observation and safe owner-observed check.
```

#### 提升流程 — 将此提升流程逐字复制到每个 hunter 提示

```text
Artifact promotion procedure (trusted parent-side code only):
Reference only for you: the parent performs these steps; you never perform them.

Before execution, the parent opens and retains trusted, non-inheritable directory
descriptors for the agent's scratch/ and artifacts/ roots, and records an allowlist
of expected scratch-relative artifact files plus explicit per-file and cumulative
byte limits. Never pass those descriptors to the agent or sandbox. After the sandbox
and all its processes terminate, trusted parent-side code promotes each allowlisted
file separately:

1. Validate the declared relative path: reject absolute, empty, `.`, `..`, or
   symlinked components.
2. Walk each parent component from the retained scratch-root descriptor with
   no-follow directory-relative operations; never reopen by path.
3. Open the leaf no-follow and nonblocking.
4. Verify with `fstat` that it is a regular file with link count exactly one and
   within the recorded per-file and cumulative byte limits.
5. Enforce those limits again while reading from that descriptor.
6. Copy exactly the verified size, repeat `fstat`, and reject a changed identity,
   type, link count, or size.
7. For the destination, walk every parent component from the retained
   artifacts-root descriptor with no-follow directory-relative operations; require
   each existing component to be a real directory, and create any missing directory
   exclusively before reopening and verifying it no-follow.
8. Create the leaf exclusively without following links, verify that the opened
   destination is a regular file with link count exactly one, and copy from the
   verified source descriptor without reopening either path.
9. Use equivalent race-safe APIs on non-POSIX systems.
10. Never recursively copy or glob scratch, extract an archive into artifacts, or
    open or promote a symlink, FIFO, socket, device, directory, hard-linked file,
    changing file, or file that exceeds its bound.
11. If any check is unavailable, cannot be enforced, or fails, discard the scratch
    entry; if it is decisive evidence, retain `needs_validation` with the exact
    promotion blocker.
```

#### 核心验证规则 — 包含在每个 hunter 提示中

```text
## Candidate gate

1. A candidate needs a complete repository-relative source trace and evidence for the
   claimed root cause, including the strongest source-visible control.
2. A proposed confirmed record needs a bounded local observed result, meaningful impact
   across a stated boundary, complete conditions, and no visible preventing layer.
3. Do not strengthen a crash into code execution, ordinary work into shared availability,
   or a same-principal action into privilege gain.
4. If a required fact is not source-visible or locally observable, use
   needs_validation. Name exact blockers; do not give it severity or speculative completion.
5. A missing best practice with no affected principal/resource is excluded or hardening,
   not a finding. A candidate disproved by source is not needs_validation.
6. Use the same source-derived fingerprint for the same root cause in every state.
   It must match `^[A-Za-z0-9][A-Za-z0-9._:/@+-]*$` and must not include a line,
   wave, agent, severity, or verdict.
7. Return an empty candidate array when nothing survives these gates.
```

## 本地验证边界

本地执行用于确认，而非扩大影响：

- **仅在必需 OS 沙箱中允许：** 使用现有依赖的离线构建；使用虚拟状态的隔离 loopback 进程；单元与集成测试；小型夹具处理；sanitizer；有界 fuzz/回归测试；确定性并发检查；带虚拟账户的本地浏览器/模拟器测试；用虚拟身份渲染的清单与策略评估；被模拟的外部或付费调用。
- **禁止：** 实时或已部署流量；对非为本隔离检查启动的服务的请求；网络依赖安装；真实账户或凭据；生产数据；共享队列、云资源、runner、注册表、签名或发布服务；发布；压力、饱和或成本生成；最小虚拟数据边界结果之后的任何工作。

沙箱以空环境启动，不给目标代码外部网络或主机可写路径，并对每次检查（不仅是预期昂贵的检查）强制显式低资源与时间限制。退出后 scratch 输出仍为目标可控。仅用 `SKILL.md` 中的 no-follow、路径受限、普通文件、有界大小主机流程提升。缺少任何沙箱或提升能力不会抹去有源码依据的候选；在 `needs_validation` 中表示精确阻塞项。

## 结构化 hunter 结果

返回恰好一个 JSON 对象，无周围散文：

```json
{
  "units": [
    {
      "coverage_id": "one assigned ID",
      "disposition": "covered|candidate|blocked",
      "reviewed_paths": ["repo/relative/path"],
      "checks": [
        {
          "agent_id": "canonical owner of this check",
          "reviewed_paths": ["repo/relative/path owned by this check"],
          "invariant": "specific control checked for this unit",
          "method": "source|local",
          "result": "what source or the bounded check established",
          "artifact": "agents/<agent-id>/artifacts/file for local, null for source"
        }
      ],
      "candidate_fingerprints": [],
      "unresolved": []
    }
  ],
  "candidates": [],
  "hardening": ["concrete non-finding note"],
  "uncovered": [
    {
      "surface": "...",
      "boundary": "...",
      "subsystem": "...",
      "attack_class": "...",
      "starting_paths": ["repo/relative/path"],
      "reason": "why this needs its own deterministic coverage unit"
    }
  ]
}
```

每个 `candidates` 条目为 schema 形状，但用 `proposed_verdict` 代替 `verdict`：

- `proposed_verdict: "confirmed"`：包含 `report-schema.json` 的 `confirmed` 分支除 `verdict` 外要求的每个字段：`fingerprint`、title、description、`root_cause`、`intended_behavior`、有序 `trace`、`evidence`、`conditions`、目标中性 `execution`、`remediation`、`severity` 与 `confidence`。执行说明仅描述已执行的有界本地检查。`payloads` 保存最小测试输入、夹具或原生调用。`observed_result` 记录实际本地输出。总体严重级别不得超过观察到的影响。
- `proposed_verdict: "needs_validation"`：包含该 schema 分支除 `verdict` 外要求的每个字段：`fingerprint`、title、description、`claimed_root_cause`、有序 `trace`、`evidence`、非空 `blockers`，以及至少有一个适用 `local` 或 `deployment` 步骤的 `validation_plan`。不要发明不适用的上下文。不要包含 severity、execution、remediation、reason 或 confirmed `root_cause`。`deployment` 是所有者观察检查，不是探测实时目标的请求。

每个分配的 coverage ID 恰好出现一次在 `units` 中。`covered` 单元需要 owner、非空 `reviewed_paths` 与 `checks`、无未决事实且无候选。`candidate` 单元有相同的拥有证据，且是唯一携带链接 fingerprint 的状态。`blocked` 单元是拥有的部分审查，具有非空路径、检查与未决事实但无 fingerprint。所有源码路径为仓库相对，绝非绝对或遍历路径。多条目追踪以 `entrypoint` 开始、以 `sink` 结束，并将中间步骤标为 `propagation`。每次检查有自己的规范小写 `agent_id` 与非空 `reviewed_paths`；单元级列表恰好是这些拥有路径的并集。`source` 检查使用 `artifact: null`。`local` 检查使用成功由 parent 提升的、位于 `agents/<check.agent_id>/artifacts/` 下的一个普通文件；这允许 verifier 添加独立拥有的证据而不从 hunter 夺取所有权。切勿链接 scratch、输出根文件或另一检查 owner 的产物。

## Parent 合并与 ledger 更新

Parent 校验每个单元结果，将其映射到恰好一个分配的 `coverage_id`，并仅更新该 ledger 单元。拒绝重复或缺失 ID、不安全的单元或检查 agent ID、带产物的源码检查，以及受信任 parent 侧代码未提升到检查 owner artifacts 子树的本地产物。将单元的 `reviewed_paths`、其 `checks` 复制到单元的 `local_checks`、链接产物路径、候选 fingerprint 与未决事实写入 ledger。将每个 hunter 的 `hardening` 列表保留在相关单元的 parent 簿记字段中（语义字段之外），以便第 6 阶段可报告。失败、畸形或不支持的结论使该单元保持 `planned` 以待再分配。未触及的预算/profile 单元成为带空证据与原因的未分配 `deferred` 单元；不要在 `deferred` 中隐藏部分证据。更新后运行 `validate-coverage-ledger.cjs`；无效 ledger 不能驱动另一次分配。此每单元契约允许一个 hunter 关闭一个单元，同时为另一单元返回候选或阻塞项。

按 fingerprint 再按根因合并候选条目。暴露多个入口路径或效果的同一根因是一个带最强完整追踪的候选。相关但独立的缺失控制使用单独 fingerprint。在相关 ledger 单元中记录重复 fingerprint，且不要将重复候选送去验证。

## Coverage-critic 波次

每波 hunter 后立即将预留调用花费在一个新的 `research` post-wave coverage critic 上。它接收 `architecture.md`、包含每个 assignment 块映射的完整 coverage ledger、当前候选 fingerprint 与状态，以及既有 ledger 缺口摘要。它读取源码但不写或不运行目标。要求恰好此 JSON：

```json
{
  "missing_units": [
    {
      "surface": "...",
      "boundary": "...",
      "subsystem": "...",
      "attack_class": "...",
      "starting_paths": ["repo/relative/path"],
      "selected_companion_blocks": ["FILE.md#section"],
      "excluded_blocks": [{"block": "FILE.md#section", "reason": "..."}],
      "reason": "source-backed coverage gap"
    }
  ],
  "reassign_ids": ["existing-id-that-did-not-close"],
  "resolved_prior_leads": ["fingerprint"],
  "stop": false
}
```

Critic 检查未映射入口点、未检查的并行路径、缺失生命周期模式、无单元的所选 companion 类、无正当理由的排除、无路径/检查即关闭的单元，以及无单元处理的既有 `needs_validation` 或变更源码缺口。它提出覆盖，而非 finding。`stop` 是 critic 自己的评估：仅当它不接受任何 `missing_units` 且无 `reassign_ids` 时为 `true`；下文 parent 的循环条件（而非单独的 `stop`）决定是否运行另一波。对 `resolved_prior_leads` 中的每个 fingerprint，parent 将链接单元或既有线索条目标记为已解决，并记录 critic 有源码依据的原因。

Parent 拒绝审查范围或源码/本地边界外的提议单元，为接受的单元派生规范 ID，并对照当前单元去重。既有同源已完成单元可提供证据；既有 `deferred`、`blocked`、`out_of_scope` 或变更源码单元变为当前工作，从不抑制被接受的单元。宁可失败也不合并规范 ID 冲突。对每个带活动 `blocked`、`covered` 或 `candidate` 证据的合法 `reassign_id`，将该确切终态记录追加到单元的 `attempts`，并带 critic 有源码依据的 `reassignment_reason`。在该归档中保留其 owner、检查、产物、fingerprint 与未决事实。递增活动 `wave`；下一 hunter 必须是新 owner，并收到带空活动证据的 `in_progress` 单元。Hunter 的终态结果仅将其新证据写入活动字段。切勿将归档 owner 的检查或产物复制到新活动尝试。在另一次分配前排序 ID 并校验 ledger。在 `standard` 与 `deep` 中，当 post-wave critic 报告无接受的 `missing_units` 或合法 `reassign_ids` 且无 `planned` 单元剩余时，将单独预留的调用花费在独立 final-clean critic 上。仅当该 critic 也返回无接受工作时才完成覆盖。若它发现问题，将其排队并重复波次、post-wave critic 与 final-clean 流程。若时间或资源强制提前停止，将每个未触及单元标记为 `deferred`，保留 critic 的原因，并在报告中披露缺口。切勿用静默波次或 Agent 上限作为完整覆盖的证据。

运行 profile 约束此循环。`quick` 运行恰好有一波 hunter 后接恰好一次最终 critic。将每个接受的 `missing_unit` 加入当前 ledger，并以原因 `quick_profile_final_critic` 标记为 `deferred`。对每个合法的含证据 `reassign_id`，将活动终态归档到 `attempts`，递增 `wave`，并将活动状态设为带空证据与原因 `quick_profile_final_critic` 的未分配 `deferred`。不要启动第二波 hunter 或另一 critic。在 scoped 运行中，critic 仍报告它注意到的范围外缺口，但 parent 将其记录为带 critic 原因的 `out_of_scope` 而非分配。上述提前停止规则是同一机制：`quick` 是预先声明的提前停止，不是完整覆盖的证据。

预算以同样方式约束。在分配每波前，将剩余预算与其 hunter 数、验证预留、立即 post-wave critic 与保留的 final-clean critic 比较（`quick` 仅预留其单一最终 post-wave critic）。缩小 hunter 波次以适应，按优先级取单元。若这些强制预留放不下，则不从该波启动 hunter，并将其 planned 单元以原因 `budget_cannot_reserve_critics_and_validation` 标记为 `deferred`。Critic 提议的单元进入同一排名队列，而非扩展预算。若存活候选超过验证预留，遵循 `SKILL.md` 中的 incomplete-run 规则：停止狩猎，按 fingerprint 顺序验证，将未验证单元保留为未解决候选，且永不将其呈现为 finding。
