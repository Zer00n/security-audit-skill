> 本文为英文原文 [`MEMORY-SAFETY-AND-BINARY.md`](MEMORY-SAFETY-AND-BINARY.md) 的简体中文译本。

# 内存安全、二进制与内核狩猎

#### 何时使用本文件

当目标在内存不安全或特权上下文中处理不受信任字节时使用本文件：C/C++/Objective-C、Rust `unsafe`、FFI、内核模块与驱动、解析器与解码器、网络守护进程、固件、二进制加载器、语言运行时与 JIT。对协议授权与状态机逻辑使用 `PROTOCOLS-RPC-AND-MESSAGING.md`，对本文件用于进程完整性、内存安全、ABI 边界与加载器行为。

从第 1 阶段挑选相关类，并按解析器、分配器/生命周期、FFI、并发、加载器、运行时或特权接口拆分大型目标。

## 核心纪律（包含在本域每个 Agent 提示中）

```
- Re-derive every bound and lifetime from attacker-controlled inputs and all callers. Validate against the worst accepted case, not a typical test vector.
- A panic, sanitizer finding, or crash proves a defect only when a realistic untrusted input reaches it. Do not infer memory corruption, code execution, or shared availability impact from a label alone.
- Validate in a local harness with sanitizers, deterministic concurrency tests, existing fuzz targets, and debugger-assisted fault classification. Stop after proving the violated invariant and observable impact; do not develop post-corruption techniques.
- Assembly, JIT code, custom allocators, intra-object accesses, and foreign libraries can escape sanitizer coverage. Identify which relevant instructions are instrumented.
- Classify as `confirmed` only after source evidence and bounded local validation establish the defect and effect. Use `needs_validation` when ABI, allocator, architecture, feature, deployment, or reachability facts remain unknown.
```

## 边界、整数与表示攻击类（subagent_type: `general`）

**越界读或写**
长度、偏移、索引或终止符在无正确边界下到达固定或已分配缓冲区。在前缀、对齐、填充与终止符之后重新计算可用余量。检查源与目标容量，以及在声明长度被信任前是否读取短输入。

**整数溢出、下溢、截断与有符号性**
在分配、复制、循环、索引与指针操作前审查攻击者控制的算术。高命中模式包括 `b > a` 时的 `a - b`、`count * element_size`、接近类型最大值的加法、负值转为无符号、64 位长度收窄到 32 位字段，以及 `-1` 等哨兵变为大尺寸。确认后续使用哪个已检查表示。

**单位与指针深度混淆**
代码混合字节、元素、码元、页、字、线单位或嵌套指针元素大小。比较解析、验证、分配、API 边界与复制处的单位。使用与分配相同错误单位的边界检查仍是错误。

**未初始化或部分初始化数据**
缓冲区、填充、结构字段或向量容量在初始化前被返回、比较、哈希、序列化或跨信任边界传递。要求可观察消费者与现实输出长度；仅栈分配本身不是披露。

## 生命周期、类型与并发攻击类（subagent_type: `general`）

**Use-after-free、过时视图与双重释放**
所有者被释放时，回调、等待队列、定时器、迭代器、借用切片或缓存原始指针仍可使用它们。审查每个错误、取消、关闭与 realloc 路径。对嵌入通知锚点，每个释放路径必须排空或分离所有观察者。

**类型混淆与无效向下转型**
标签、vtable、联合判别器、对象种类或外部句柄的检查方式与随后读取的表示不同。寻找未检查动态转型、重用后过时标签，以及验证元素与消费元素不同的序列化类型。在不将测试扩展到违反不变量之外的情况下，本地确认错误类型读或写。

**引用计数与所有权竞态**
非原子 retain/release、检查后未加锁使用，或跨线程不一致的所有权，可在访问期间释放或变更对象。比较快速、错误、关闭与兼容路径的同一锁与所有权规则。

**共享状态竞态与 TOCTOU**
并发解析器流、全局缓存、惰性初始化、信号处理与资源拆除可使先前确立的边界、策略或指针失效。用可重复本地调度、屏障或 thread sanitizer 验证竞态；无可安全相关状态转换的假设交错保持 `needs_validation`。

**锁序、死锁与饥饿**
外部可达操作以不一致顺序获取锁，或在回调与阻塞 I/O 中持锁。仅当有界输入可停止共享进度时在可用性下报告；否则作为并发缺陷记录以修复。

## FFI 与 ABI 攻击类（subagent_type: `general`）

**指针-长度与所有权契约不匹配**
调用方与被调用方在谁分配、释放、固定或变更缓冲区、指针有效时长，或长度是字节还是元素上不一致。追踪每个 `extern`、CGo/JNI/Python/原生绑定与生成包装两侧。检查 null、零长度、别名与回调保留。

**布局、对齐与枚举不一致**
外部代码接收因架构或构建标志而不同的结构、位域、打包记录、回调签名、整数宽度、枚举或调用约定。验证 `repr`、打包、对齐、端序与 ABI 特定类型。仓库内声明不匹配可本地确认；不透明外部实现需要 `needs_validation`。

**展开、异常与线程亲和违反**
异常或 panic 跨越禁止展开的 ABI，回调在拆除后运行，或要求单运行时线程的 API 在别处被调用。审查错误转换与取消。在分配影响前确认进程是否中止或状态是否损坏。

## 二进制加载与运行时攻击类（subagent_type: `general`）

**库、插件与可执行搜索顺序信任**
特权进程从较低信任主体可写的路径加载库、插件、运行时映像或 helper，或通过攻击者可影响的工作目录或环境解析裸名。比较预期安装所有权与每个回退及兼容搜索路径。用户向自己的进程加载自己的插件不是边界违反。

**缺失产物身份或签名绑定**
加载器验证一个文件或元数据记录，但因路径解析、文件替换、架构切片或嵌入资源未绑定到检查而映射不同映像。供应渠道真实性属于 `SUPPLY-CHAIN-AND-RELEASE.md`；本类覆盖本地验证到映射的缺口。

**畸形二进制元数据与重定位处理**
偏移、计数、节、重定位、符号、字节码或调试元数据在范围、重叠与表示检查前被信任。用有界本地夹具与 sanitizer 测试解析器。将内存损坏与安全拒绝的畸形文件分开。

**JIT 与生成代码一致性**
验证器、解释器、优化器与生成代码在类型、边界、副作用或生命周期上不一致。用相同本地输入对比优化与未优化路径。确认进程完整性效果；保持在语言语义内的输出差异不是 finding。

**卸载、重载与拆除安全**
活动函数指针、回调、工作线程或数据视图在模块卸载或运行时重置后存活。审查关闭与失败加载清理，与启动同等仔细。

## 内核与特权接口攻击类（subagent_type: `general`）

**用户拷贝边界与重复读取**
Syscall、ioctl、驱动或内核解析器从用户内存派生受信任事实，然后再次读取同一可变地址。一次拷贝完整请求或重新验证后续拷贝。同时审计每个用户拷贝原语的大小、方向与访问检查。

**特权对象生命周期与分发一致性**
外部可达对象有不平衡 retain/release、拆除时无观察者排空、未检查选择器/表索引，或省略守卫的重复兼容路径。并排对比每个分发与释放路径。

**授权不足的强大接口**
设备节点、管理 socket、helper 或管理 API 验证形状但不验证调用方对资源的权威。确立实际接口所有权与可达性；仓库外的权限或沙箱策略使此为 `needs_validation`。

## 通用动作（适用于上述）

- 审计修复与重复路径的同一源到汇点形状。一个调用方、架构、协议角色、功能开关或兼容路径中的检查不保护其兄弟。
- 为每个解析器或 FFI 边界建表：接受的长度/类型、检查表示、分配所有者、消费者、线程与拆除。多数原生 finding 是该表中的一个不一致。
- 使用既有语料与小型本地生成的边界夹具。保存精确 sanitizer/运行时输出与触发它的输入属性；避免大资源消耗与任何实时目标。

## 验证规则（在此报告任何 finding 前应用）

1. 确立现实不受信任入口与违反边界、类型、生命周期、ABI、并发、加载器或权威不变量的确切操作。
2. 分类可观察效果：无效读、无效写、过时别名、错误对象、未初始化输出、未授权映像加载、死锁或安全进程终止。不要声称强于观察到的效果。
3. 运行证明效果所需的最窄本地 harness、既有测试、sanitizer 或 fuzzer。验证故障操作的 sanitizer 覆盖并记录架构/构建条件。
4. 对并发，使用确定性调度或 sanitizer 追踪。对二进制加载，证明检查身份与映射身份不同，并命名较低信任写入者。
5. 仅以精确输入、源码追踪与观察结果返回 `confirmed` finding。对特定未决可达性、ABI、构建、部署或运行时事实返回 `needs_validation`，并陈述所需有界检查。
