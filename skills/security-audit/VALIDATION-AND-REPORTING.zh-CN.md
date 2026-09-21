# 验证、结构化输出、核验与报告

> 本文为英文原文 [`VALIDATION-AND-REPORTING.md`](VALIDATION-AND-REPORTING.md) 的简体中文译本。

### 第 3 阶段：独立验证每个候选

在 clean coverage-critic 遍历或显式记录的提前停止之后，按稳定 fingerprint 与根因合并第 2 阶段候选与携带的同源既有 confirmation。将每个唯一提议的 `confirmed` 与 `needs_validation` 候选交给未狩猎过它的新 `general` verifier。携带的既有 confirmation 走相同的当前核验路径，即使 hunter 排除该未变根因。Verifier 可读取 hunter 或既有产物，但必须重新阅读每个引用的当前源码位置，并独立运行其可安全复现的任何决定性检查。

为每个 verifier 分配规范小写唯一 ID 与 `<output-dir>/agents/<verifier-id>/scratch/` 加 parent 拥有的 `artifacts/`。Verifier 仅写入 `scratch/`，从不写保留产物。它仅接收候选、其链接的 coverage-unit 检查与产物路径、解释路径所需的架构事实、精确相关 companion 验证块、下文提升流程块、源码/本地执行边界、逐字复制的 `report-schema.json` 的 `confirmed`、`needs_validation` 与 `rejected` 分支，以及相同 fingerprint 的既有记录。它不得接收另一 verifier 的结论。

#### 候选 verifier 提示

```text
You did not write this candidate. Try to refute it from repository source and bounded
local evidence. Do not contact deployed endpoints or external/shared services. Run
target-controlled code only inside the approved OS-enforced sandbox: no external
network, empty allowlisted environment, read-only target and tools, scratch-only
writes, and explicit low resource and wall-clock limits. If any control is unavailable,
do not execute; retain the exact missing capability as a needs_validation blocker.
Treat every scratch entry as target-controlled after execution. After the sandbox and
all its processes terminate, only trusted parent-side code may promote a predeclared
scratch-relative file, following the promotion procedure block included verbatim in
this prompt. You and target code never write retained artifacts. If promotion is
unavailable or fails, do not use that file as evidence.

1. Verify every trace and evidence file, positive line number, scope, and description.
   Confirm the first entry is a real lower-trust entrypoint and the last is the
   claimed sink or boundary effect.
2. Reconstruct the strongest source-visible validation, identity, authorization,
   normalization, lifecycle, framework, and containment controls on the path.
   Where the architecture summary names a comparable baseline, note whether it
   shares the pattern — as calibration, never as grounds to dismiss.
3. For a proposed confirmed candidate, independently reproduce the minimum observed
   result when possible. Verify inputs, interface shape, conditions, and affected
   dummy principal/resource. Do not infer a stronger result or continue after it.
4. Verify that likelihood, impact, confidence, and the proposed source fix match only
   what the evidence establishes.
5. For a proposed needs_validation candidate, decide whether the blocker is genuinely
   outside source/local observation. If source refutes the trace, reject it. If the
   missing fact remains decisive, keep needs_validation and make the local and
   owner-observed plans exact and non-destructive.
6. Preserve the fingerprint for the same source-derived root cause across every state.

Return exactly one JSON object and no surrounding prose:
{"decision": "confirmed|needs_validation|rejected", "record": { ... }}
where record exactly matches the decision's verdict branch of the schema included
in this prompt. A corrected record replaces the hunter's wording.
```

将此提升流程逐字复制到每个候选 verifier 提示：

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

Verifier 仅可在独立确立完整路径与有界观察结果后，将 `needs_validation` 提升为 `confirmed`。当特定部署或运行时事实仍未知时，将提议的 confirmation 降为 `needs_validation`。当源码、本地行为、可见控制、缺少有意义影响或不可能的先决条件证伪主张时，使用 `rejected`。`needs_validation` 绝不是投机想法的停放处。

Parent 检查每个 verifier 返回相同 fingerprint，除非它识别出真正不同的根因。合并纠正，在每个链接 coverage 单元中记录判定，并确保每个 fingerprint 有一条最终记录。丢弃畸形或散文包裹的 verifier 结果而不修复；预算允许时用新 verifier 重跑该候选，否则它在 incomplete-run 规则下保持为未验证 ledger 候选。

当 verifier 证据更新 ledger 检查时，将该检查的 `agent_id` 设为 verifier 的规范 ID，并列出其非空仓库相对 `reviewed_paths`。保持单元级 `reviewed_paths` 等于各检查的并集。对仅源码审查使用 `method: "source"` 与 `artifact: null`。仅对成功由受信任 parent 侧代码提升到 `agents/<check.agent_id>/artifacts/` 下的文件使用 `method: "local"`。单元保留其原始 assignment owner，因此独立拥有的 hunter 与 verifier 检查可共存。对携带既有记录的种子 `planned` 单元，无先前 owner：复核它的 verifier 成为该单元的 assignment owner，其复核是该单元的首次检查，并将单元移至带携带 fingerprint 的 `candidate`。

若严格总 Agent 预算无法覆盖每个候选，将运行状态设为 incomplete，并遵循 `SKILL.md` 中的确定性预算规则。未验证候选仅保留在 ledger 中。它不以任何判定进入 `findings.json`。

### 第 4 阶段：写入并校验 `findings.json`

Parent 将所有独立决定的记录写入 `<output-dir>/findings.json`，按 fingerprint 排序。包含：

- `confirmed`：有完整本地执行证据、条件、具体修复、可能性/影响/总体严重级别与置信度的有源码依据漏洞。
- `needs_validation`：带精确未决阻塞项与至少一个适用本地或所有者观察部署计划的有源码依据候选。
- `rejected`：验证期间被证伪的有源码依据候选，保留以便未来运行在证据未变时不重复不受支持的主张。

写入前立即阅读 `report-schema.json`。它使用 `additionalProperties: false`；不要将 hunter 包装字段带入记录。保持这些判定契约彼此独立：

- `confirmed` 记录使用 `root_cause`、`intended_behavior`、`conditions`、`execution`、`remediation`、`severity` 与 `confidence`。它不得使用 `claimed_root_cause`、`blockers`、`validation_plan` 或 `reason`。`execution` 是目标中性的，并使用目标的原生接口：适用时的 API/HTTP 输入、CLI 调用、库调用、消息、文件夹具、浏览器动作、渲染策略或本地 harness。`observed_result` 非空且事实性。
- `needs_validation` 记录使用 `claimed_root_cause`、`trace`、`evidence`、`blockers`，以及至少一个非空 `validation_plan.local` 或 `validation_plan.deployment` 字段。仅当两种上下文可解析不同事实时才同时包含。它不得使用 severity、execution、remediation、reason 或 confirmed root cause。
- `rejected` 记录使用 `claimed_root_cause`、`trace`、`evidence` 与 `reason`。它不得使用 severity、execution、remediation、blockers、validation plan 或 confirmed root cause。

每条记录有稳定 fingerprint、标题、描述与仓库相对源码路径。多步追踪以 `entrypoint` 开始、以 `sink` 结束，并仅在其间使用 `propagation`。单条目追踪使用 `entrypoint` 或 `sink`。总体严重级别不能超过已证明影响。

运行：

```sh
node <skill-dir>/validate-findings.cjs <output-dir>/findings.json
node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json
```

在继续前修复每个结构与语义错误。Findings 校验器拒绝超过 5 MiB、1,000 顶层 finding 或 64 层嵌套的输入，并将报告错误输出上限为 100 条消息。校验器成功仅证明格式与 ledger 一致性。

### 第 5 阶段：用新视角核验最终记录

并行为每条最终 `confirmed` 与 `needs_validation` 记录启动一个新的 `research` verifier。此 verifier 检查结构化记录而非 hunter 文稿，并保持在源码/本地边界内。

在 `quick` 运行中，第 3 与第 5 阶段合并：第 3 阶段 verifier 也执行这些记录检查并返回最终 schema 形状记录，因此每个候选得到一个而非两个新独立审查者。其他每个 profile 保持两遍分离。任何 profile 都切勿跳过对 `confirmed` 记录的独立审查。

对 `confirmed`，要求其检查：

1. 每个仓库相对追踪/证据路径、行、范围与描述的操作。
2. 真实入口接口与精确本地输入形状。
3. 每个条件、解析器/策略步骤、源码可见防止层与观察到的本地结果。
4. 受影响主体/资源与已证明影响。
5. 严重级别分离：现实可能性、已证明影响、总体不大于影响。
6. 修复策略与任何 `code_changes`，包括修复是否强制不变量而不只是移动信任。

对 `needs_validation`，要求其检查：

1. 源码路径真实且仅支持所述 `claimed_root_cause`。
2. 每个列出的阻塞项是决定性的，且本地尚不可回答。
3. 候选指明边界与可能的具体结果，而非通用关切。
4. 至少一个验证计划字段存在且精确。`local` 使用有界夹具；`deployment` 请所有者观察配置、身份、路由、策略或运行时事实。不要为不适用上下文发明计划，且切勿向部署发送审计流量。
5. Fingerprint 对同一根因匹配既有/当前记录。

每个 verifier 返回恰好一个 JSON 对象：`{"decision":"verified","fingerprint":"..."}` 或 `{"decision":"replace","reason":"...","record":{...}}`，无周围散文。替换记录必须匹配其 `confirmed`、`needs_validation` 或 `rejected` schema 分支。将畸形或散文包裹的第 5 阶段结果与第 3 阶段同样处理：丢弃而不修复，预算允许时用新 verifier 重跑。

当第 5 阶段替换将记录提升为更强判定（包括任何提升为 `confirmed`），或实质改变根因、追踪、执行输入或观察结果、已证明影响或严重级别时，不要将其作为最终应用。将该完整替换交给未狩猎、未执行第 3 阶段验证、未提议第 5 阶段替换的新独立 verifier。新 verifier 复核当前源码，并在执行边界下独立复现任何决定性本地结果，然后返回 `verified` 或另一替换。仅在此新核验后应用实质性替换。若产生另一实质性替换，用新 verifier 重复。若预算或独立性不可用，从 `findings.json` 移除有争议记录，将其 ledger 单元保持为未解决候选，并设置带精确 `incomplete_reason` 的 `run_status: "incomplete"`。不改变含义或证据的非实质性措辞或仓库行纠正可直接应用。

每次应用替换后，重跑两个校验器并更新链接 ledger 判定。若最终 verifier 识别出单独根因，分配新 fingerprint 并在纳入前送入独立候选验证。仅当每个 ledger 候选有独立最终处置且每条保留记录通过第 5 阶段时，设置 `run_status: "complete"`。

不要仅核验 `confirmed` 记录。误导性的 `needs_validation` 交接浪费所有者时间，并可保留错误前提。

### 第 6 阶段：由最终记录产出目标中性报告

仅在 `findings.json` 中保留的每条记录通过第 5 阶段后，由最终记录、ledger 与保留在 ledger 簿记中的 hunter `hardening` 笔记派生散文。Incomplete 运行可报告独立核验的记录，但必须识别每个未解决 ledger 候选，且不得将其呈现为 finding。散文文件从不改变判定、严重级别、阻塞项或已证明影响。

#### `REPORT.md`

写：

1. 运行 profile、范围、预算（若设置）及已花费相对计划的 Agent、source ref、沙箱源码与仅本地执行声明、既有运行使用，以及显式 deferred 与 out-of-scope 覆盖。命名携带的同源 confirmation 与变更源码再验证。`quick`、scoped、预算受限或不完整运行须明文说明其为部分遍历。若候选验证耗尽严格预算，说明运行不完整并列出每个未验证 fingerprint 与链接单元；不要将这些候选描述为 finding。若预算阻止强制 critic，说明哪个 critic 未运行，且不做 clean-coverage 声称。
2. 一段简短安全态势摘要。
3. Confirmed finding 表：严重级别、标题、受影响边界与一行观察结果。
4. 每个 confirmed finding：仓库源码位置、较低信任主体、目标原生有界复现、条件、实际结果、影响、优先级理由与最小源码修复。
5. 单独的 `NEEDS VALIDATION` 表。给出每条线索的标题、仓库追踪、精确阻塞项、有界本地下一步与安全所有者观察部署检查。不要分配严重级别或称之为 confirmed 漏洞。
6. 单独的 hardening 笔记与正面源码模式。
7. 来自 ledger 的覆盖摘要：covered、candidate、blocked 与 deferred 计数，以及重要排除与最终 critic 结果。

不要将 rejected 记录描述为 finding。仅在它们解释既有分歧或覆盖决策时提及 fingerprint。

#### `FINDINGS-DETAIL.md`

对每个 confirmed `medium`、`high` 或 `critical` 记录，复制完整源码路径与目标中性本地复现：

- 有序仓库相对追踪与证据；
- 虚拟攻击者/主体与受影响虚拟资源；
- 原生输入、调用或夹具与精确有界说明；
- 观察输出及其证明的安全不变量；
- 条件与遏制；
- 源码级修复与回归用例。

#### `NEEDS-VALIDATION.md`

对每条未解决记录，复制源码追踪、已核验证据、精确阻塞项、受影响边界，以及每个适用的有界本地或所有者观察解决计划。将其作为无严重级别的优先级线索。不要将其变为实时测试指导或假设缺失的部署事实。

HTTP 是一种可能的原生接口，不是默认。库 finding 可能使用函数调用，解析器使用夹具，CLI 使用命令，桌面应用使用 IPC 或文件动作，基础设施使用本地渲染策略。不要要求目标没有的端点、外部账户或实时环境。

保持报告与证据成比例。干净运行可以有零条 confirmed 记录。陈述该结果与剩余覆盖/验证限制，而不发明 LOW finding。
