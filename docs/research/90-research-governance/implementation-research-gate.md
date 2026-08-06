# Implementation 前研究门禁

- 状态：必须执行的工程门禁
- 更新日期：2026-07-31
- 适用范围：`docs/research/` 中所有长期功能、协议、持久化与 package 变更
- 目标：在修改 production code 前取得足够证据，优先选择长期低后悔方案，而不是把未经验证的研究假设直接实现

## 1. “长期最优”的可执行定义

这里的“最优”不等于：

- 一开始建立最多 abstractions；
- 为所有未来场景一次设计完整框架；
- 新建最多 packages；
- 永远不再迁移；
- 在没有数据时声称某个存储、协议或平台实现是最终答案。

长期最优必须同时满足：

1. **正确性**：真相来源、不变量、授权边界、错误语义和恢复行为明确；
2. **可验证性**：关键结论能由测试、fault injection、真实 smoke test或版本匹配的一手资料证明；
3. **可演进性**：稳定wire identity与domain contract不依赖physical layout、UI或provider；
4. **低迁移成本**：保留稳定identity/provenance，迁移有显式cutover与rollback，不靠永久shim；
5. **低upstream冲突**：fork-owned domain通过少量稳定seams接入，不在upstream高churn文件散布逻辑；
6. **足够简单**：没有真实第二consumer、规模数据或失败证据时，不提前引入ledger、pack、distributed lock等复杂机制；
7. **平台诚实**：只声明实际验证过的OS、filesystem、runtime与dependency version；未知组合fail closed或明确best-effort；
8. **用户结果完整**：从用户入口到restart/recovery的observable contract成立，不以unit API完成代替端到端完成。

因此门禁追求的是：

> 冻结高置信、跨实现稳定的不变量；用有限spike消除承重未知；把可逆实现选择延迟到有证据时。

## 2. Implementation 不得开始的条件

任一条件成立，都不得修改production code：

- 当前实现与调用者尚未完整映射；
- 不知道canonical truth属于哪个domain/package；
- 存在两个看似相同但语义不同的现有abstraction，尚未比较或收敛；
- 方案依赖未核实的external API、Node/libuv、filesystem、database或provider语义；
- 没有写出至少一个会失败的observable acceptance scenario；
- crash、cancel、timeout、partial write、duplicate/retry、concurrent writer或corruption中有适用项但尚未定义行为；
- migration/cutover会影响已有durable user data但没有验证策略；
- 新package尚未证明独立ownership与consumer seam；
- 方案需要以“后续再补安全/恢复/授权”为前提才能工作；
- 文档之间仍有wire/API/ownership矛盾；
- 关键选择只能靠偏好解释，不能用criteria与evidence区分。

## 3. 必备 Research Receipt

每个implementation work package开始前必须有一份可审查receipt，内容如下。

### 3.1 Outcome 与边界

- 用户可观察结果；
- 明确非目标；
- 哪些旧行为有意保留、删除或clean cutover；
-完成后如何smoke test；
-若只做spike，说明它回答哪个具体未知，且spike不得伪装为production implementation。

### 3.2 Current-state map

必须从仓库证据说明：

- canonical truth当前在哪里；
- producer、storage、consumer、renderer、migration与tests的完整路径；
- exported symbol修改前的references；
-同概念是否已有第二套implementation；
-现有extension/package seam；
- upstream-owned与fork-owned文件边界；
- unexpected repo changes是否属于其他会话/用户。

宽范围变更必须完整阅读目标文件；不得凭search snippet设计。

### 3.3 External evidence

当方案依赖外部行为时：

- 优先官方规范、runtime/source、version-matched types与release notes；
-记录精确版本、URL、support boundary；
-区分规范保证、当前实现事实与推断；
-第三方库必须检查实际installed types/source；
-不可把POSIX结论自动外推到Windows、network filesystem、container overlay或object store；
-空结果或单一资料不足以关闭未知。

### 3.4 Alternatives

至少比较：

1. 保持现状/最小源头修复；
2. 收敛到现有domain abstraction；
3. 建立新domain/package/protocol；
4. 若适用，先做有限spike再决定。

每个候选按以下维度给证据，不只打主观分数：

- correctness与failure containment；
- canonical truth是否唯一；
- migration与rollback；
- upstream conflict surface；
- dependency direction；
- performance/resource cost；
- security/privacy；
- platform portability；
- implementation/test complexity；
-未来替换成本。

选择必须写清“为什么现在选它”以及“哪些新证据会推翻它”。

### 3.5 Invariants 与 ownership

必须冻结：

- stable identity；
- canonical truth与derived projection；
- write/commit/ack ordering；
- idempotency/deduplication；
- authorization/provenance scope；
- version/unknown-field behavior；
- package owner与dependency direction；
- error/result taxonomy；
- resource lifecycle与retention owner。

同一事实不得由两个可变store同时拥有。

### 3.6 Failure 与 threat matrix

按适用范围覆盖：

- create/write/flush/publish/append/ack每个边界；
- process kill与restart；
- cancellation/timeout/non-zero结果；
- malformed/corrupt/missing/duplicate/conflicting data；
-同进程与跨进程concurrency；
- stale writer/snapshot/credential；
- path traversal、symlink、opaque ID、secret/export；
- branch/fork/clone/delete/compaction；
- dependency unavailable或版本变化。

每一项必须定义：可见状态、是否committed、恢复动作、用户diagnostic、是否允许继续mutation。

### 3.7 Migration、rollout 与 rollback

Durable data或public API变更必须说明：

- source bytes/identity如何保留；
- migration是read adapter、overlay、copy还是rewrite；
-何时cut over canonical truth；
- interruption后如何resume；
-验证等价性的oracle；
-失败后如何回到旧reader而不丢新数据；
-何时删除旧实现；
-不保留永久backward-compatibility shim。

若选择不兼容clean cutover，也必须处理用户已有durable data；“不保留API兼容”不等于可以丢session。

### 3.8 Performance 与 support budget

必须先给预算或测量问题：

- memory bound；
- I/O passes与bytes copied；
-启动/replay latency；
-单对象/总对象数量；
-并发与backpressure；
-磁盘/DB增长；
-支持的OS/runtime/filesystem/provider矩阵。

没有真实测量前，不因想象的规模提前加入pack、cache、index或batching。

### 3.9 Acceptance contract

必须包含：

-第一个会失败的observable test；
-成功、边界、错误与负向授权tests；
- fault/concurrency tests；
- migration/restart equivalence；
-真实入口smoke test；
- support matrix evidence；
- repo-required targeted tests与`npm run check`；
-明确Done gate，禁止用“已知风险”绕过blocking prerequisite。

测试必须保护行为，不得断言source text、mock plumbing或偶然实现细节。

## 4. Spike 规则

Spike只用于消除一个会改变架构选择的承重未知。

每个spike必须：

- 先写具体问题和相互竞争的答案；
-使用真实runtime/platform/dependency；
-有可重复命令、输入和measurement；
-输出原始evidence与结论；
-说明结果如何选择/排除方案；
-默认不进入production package；
-结束后删除临时代码，或明确转为行为测试。

禁止“先实现大半功能，再称为spike”。

## 5. Package 提取门禁

新package至少满足：

- canonical domain owner唯一；
- public API比physical implementation稳定；
- dependency direction不反向依赖composition/UI/provider；
-至少一个完整vertical slice证明边界；
-第二consumer已经存在，或其合同已由同一slice中的两个独立adapter证明；
-移动后upstream touch points可量化减少；
-没有只是把同一change amplification跨目录分散；
- package release/version/export/test ownership清楚。

否则先做fork-owned deep module，不创建空scaffold。

## 6. Research Review Gate

以下变更在实现前必须有独立只读审查：

- durable storage/journal/schema/migration；
- authentication/authorization/secrets/PII；
- multi-process concurrency、lease、queue、retry；
- public API/protocol/wire format；
- package boundary或跨三层以上的architecture cutover；
- platform-specific filesystem/process behavior。

审查只报告blocking/high缺陷，并逐项追踪到：

- 已修正文档；
- 有限spike；
-明确非目标；
-或支持矩阵中的fail-closed。

未经关闭的blocking finding禁止进入implementation。

## 7. Implementation 执行门禁

Research Receipt通过后，implementation按以下顺序：

1. 写最小失败测试；
2.实现一个coherent、可回滚的vertical work package；
3.运行针对性测试；
4.从真实用户入口smoke test；
5.仅在行为成立后做cleanup、changelog与完整`npm run check`；
6.独立review introduced patch的correctness；
7.验证所有callers、durable data与docs均已迁移；
8.提交只包含该work package的明确文件。

一个work package不得同时把“研究未知、协议选择、migration与完整feature”混在一起。

