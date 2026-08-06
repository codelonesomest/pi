# Research: Session-scoped Resource Ownership and Lifecycle

- 状态：研究结论与实现前门禁建议
- 更新日期：2026-08-06
- 研究对象标题：**Session-scoped Resource Ownership and Lifecycle**
- 中文名称：**Session 资源所有权与生命周期**
- 范围：研究如何把一个交互式 session 及其机器创建的子 session、`local://` 文件、`artifact://` 输出和相关运行资源组织为可发现、可恢复、可整体清理的 ownership group；重点比较 OMP 式物理目录分组与 Pi 自有的逻辑资源所有权模型。
- 非范围：本研究不实现 `drop` API、不迁移生产数据、不冻结最终物理目录布局，也不把 OMP 的 Bun-only 实现直接复制进 Pi。
- OMP 证据边界：主要证据来自本机 `../oh-my-pi-main/` 文件系统快照。该目录不是 Git repository，无法取得 commit pin；其 `packages/coding-agent/package.json` 声明版本 `17.2.1`、Bun `>=1.3.14`。用户已确认该快照不是 OMP 最新代码，因此 OMP 结论仅代表已读取快照的 observed behavior，不代表最新版本保证。

## Question

Pi 是否应把 session 视为一组资源的 ownership boundary，使主 session、Subagent、`local://`、`artifact://` 以及必要的相关资源处在同一个 session folder 或 namespace 中，未来可以通过一次 `drop` 安全地删除或回收整组内容？如果应当这样做，canonical truth 应是物理 folder、session lineage，还是独立的 session group/resource binding？

### Decision context

当前仓库同时存在两条 session 相关实现路径：

1. `packages/agent/src/harness/session/` 的 v4 `Session` / `SessionRepo` 契约，以及 JSONL 实现。
2. `packages/session-backends/sqlite-node/` 的 SQLite `SessionRepo` adapter。
3. `packages/coding-agent/src/core/session-manager.ts` 仍保留较早的 coding-agent JSONL session manager 与 session selector 入口。

这意味着实现前必须先确认哪一条是目标 production seam；不能在两个 session abstraction 上分别添加一套 group 语义。

## Conclusion

**建议建立 Pi-owned 的 `SessionGroup` 逻辑 ownership 概念，并把 folder 作为 JSONL/filesystem adapter 的 materialized layout，而不是 canonical truth。**

`SessionGroup` 应代表一个交互式 root session 对其机器创建的 descendants 和 session-scoped resources 的拥有范围。每个 child session 仍保留自己的 `sessionId`、parent lineage 和独立生命周期，但通过稳定的 `sessionGroupId` 归属于 root group。`parentSessionId` 只能表示 lineage，不能单独承担 group membership；它无法表达 group 查询、显式脱离 lineage、跨 backend 的资源清单或删除状态。

对用户来说，推荐的结果仍然是 OMP 式体验：主 transcript、child transcripts、工具输出和 `local://` 文件可以在同一个 group directory 下被发现，并且未来可以从一个 group 入口执行 drop。对实现来说，membership、authorization、drop state 和 cleanup receipts 必须由 session domain 的 durable store 管理；不能靠扫描 sibling files 或依赖一个可变 folder 是否存在来推断归属。

这是一项**架构方向建议，不是 production implementation authorization**。当前还缺少对目标 session seam、跨 backend group transaction、旧数据迁移和 filesystem failure semantics 的验证。

## Findings

### 1. 当前 Pi 的 SessionRepo 删除粒度是单 session

**[OBSERVED]** `packages/agent/src/harness/session/types.ts` 的 `SessionRepo` 暴露 `create`, `open`, `list`, `delete` 和 `fork`；`SessionCreateOptions` 有 `parentSessionId`，但没有 group/resource ownership 字段。

**[OBSERVED]** `packages/agent/src/harness/session/jsonl/repo.ts` 的 `delete(metadata)` 只调用 filesystem adapter 删除 `metadata.path`。JSONL repository 的 `createDirect()` 为每个 session 生成独立的 `<timestamp>_<id>.jsonl`，并把 `parentSessionId` 写进 header；`fork()` 继承 parent relation，但没有共同资源组删除操作。

**[OBSERVED]** `packages/session-backends/sqlite-node/src/sqlite/repo.ts` 的 `delete()` 会在 SQLite transaction 中删除单个 session 的 branch cache、facts、lanes、records、entries、lease、stats、sequence 和 session row。该 transaction 保证单 session 的数据库内部清理，但 schema 没有 group table、resource table 或跨 session membership。

**[INFERENCE]** 未来的“删除整个 session”若只循环调用 `SessionRepo.delete()`，会把 group deletion 的正确性分散到调用者：调用者必须自行发现 descendants、清理文件、处理并发和恢复，形成浅而危险的组合 API。group cleanup 应位于拥有 session/resource truth 的深 module seam。

### 2. OMP 已采用“父 session artifact root 收纳 child transcript”的行为

**[OBSERVED]** 本地 OMP `packages/coding-agent/src/session/session-manager.ts` 中，`artifactsDirectoryFor(sessionFile)` 通过去除 `.jsonl` 后缀得到 artifact directory。源码注释明确描述：主 session 为 `<parent>.jsonl`，subagent session 写入 `<parent>/<agentId>.jsonl`。

**[OBSERVED]** 本地 OMP `packages/coding-agent/src/task/executor.ts` 在提供 `artifactsDir` 时创建 `path.join(options.artifactsDir, `${id}.jsonl`)` 作为 child transcript。它同时接受 `parentArtifactManager`，由 child `SessionManager` 采用 parent 的 `ArtifactManager`；因此整个 agent tree 共用一个 artifact directory 和 numeric artifact ID space，而不是每个 subagent 独立建一套 artifact root。

**[OBSERVED]** OMP `packages/coding-agent/src/internal-urls/local-protocol.ts` 的 `resolveLocalRoot()` 优先把 `getArtifactsDir()` 下的 `local/` 作为 persistent local root；没有 artifact dir 时回退到 `os.tmpdir()/omp-local/<safeSessionId>`。`resolveLocalTarget()` 对 root、parent 和 target 做 realpath/containment checks，阻止 path traversal 与 symlink escape。

**[OBSERVED]** 本地 OMP `packages/coding-agent/src/session/session-storage.ts` 的 `deleteSessionWithArtifacts()` 删除主 `.jsonl` 后，再递归删除同名 artifact directory。OMP 的 `IndexedSessionStorage` 会把该 directory 前缀下已索引的 child paths 一并从 backend 删除，并在 backend 失败时恢复内存 index。

**[INFERENCE]** OMP 的物理布局很好地证明了“grouped session artifacts”这一用户体验，但它不是充分的 membership proof：报告打包代码仍通过扫描 parent artifact directory 下的 `.jsonl` 文件推断 subagents，且只保留最近 10 个。这种扫描适合诊断/导出 projection，不适合成为未来 drop 的唯一授权依据。

### 3. OMP 的 artifact group 不是 global blob group

**[OBSERVED]** OMP `docs/blob-artifact-architecture.md` 将 content-addressed blobs 与 session-scoped artifacts 明确分开：blob 位于共享 global blob directory，引用是 `blob:sha256:<hash>`；文本 artifacts 位于 `<sessionFile-without-.jsonl>/`，供 `artifact://` 和 `agent://` 读取。

**INFERENCE]** `drop SessionGroup` 不应默认删除 global blobs。删除一个 session 只能移除该 group 的 ownership/reference；如果未来要做 blob garbage collection，应由独立的 reference-count/reachability/retention owner 负责。否则两个 session 共享同一个 image blob 时，drop 一个 session 会破坏另一个 session 的历史。

### 4. 相关运行资源不等于 session-owned bytes

**[OBSERVED]** OMP task isolation 使用 `~/.omp/wt/` 下的 sandbox，并写入 `.omp-isolation-owner.json`，记录创建进程的 PID、task id 和可选 process start token；cleanup 会区分 live owner 与 crashed leftover。

**INFERENCE]** worktree、PTY、MCP connection、eval kernel、process 和 provider handle 不能因为“由 session 创建”就被当作可直接删除的文件。它们需要各自的 lifecycle/reconcile policy，并且只有在 resource binding 明确声明 session group owns cleanup 时，`drop` 才能请求终止或回收。PID、in-memory registry 和 live Promise 可以用于 runtime liveness 判断，但不能作为 restart-safe canonical truth。

### 5. `parentSessionId` 与 `SessionGroup` 语义不同

**OBSERVED]** 当前 v4 metadata 已有 `parentSessionId`，而 OMP child session 也通过路径和 parent context 形成层级。

**INFERENCE]** lineage graph 与 ownership group 应分开：

- lineage 回答“这个 session 从哪个 session fork/spawn”；
- group 回答“哪个 owner 对这组资源负责 retention/drop”；
- resource binding 回答“这个具体 file/blob/worktree/process 是否属于 group，以及如何清理”。

单纯沿 `parentSessionId` 递归删除会误把用户可见 fork、导入 session 或显式 detach 的 session 纳入删除，也不能处理没有 parent link 但共享 resource root 的情况。

## Domain model proposal

```text
SessionGroup (durable owner)
├── rootSessionId
├── sessionGroupId
├── member sessions
│   ├── root interactive session
│   └── machine-created child sessions
├── resource bindings
│   ├── transcript
│   ├── artifact files / artifact directory
│   ├── local:// root
│   ├── optional eval/kernel state
│   └── optional process/worktree lease
└── lifecycle / cleanup receipts
```

建议术语：

- **Session**：一个可恢复的 conversation/event tree。
- **SessionGroup**：一个 session owner 负责的可清理资源集合；通常以 root session 为用户入口。
- **Session lineage**：session 与 parent/child 的历史关系；不是 ownership 的替代品。
- **Session resource**：可被 group 引用的 transcript、artifact、local root 或 runtime handle。
- **Resource binding**：`resourceId`, `groupId`, `kind`, `locator`, `createdByOperationId`, `cleanupPolicy`, `status` 的 durable association。
- **Global blob**：content-addressed、可被多个 group 引用的共享 bytes；不属于单个 group 的独占资源。

建议冻结的 identity 不变量：

1. `sessionGroupId` 与 `sessionId` 独立且稳定；不能由 filename、numeric artifact ID 或 process ID 充当 group identity。
2. 每个 persistent session 恰好属于一个 group；in-memory session 可没有 durable group，但不能伪装成可恢复 group。
3. 每个 group 有一个 root owner session；child session 可以有 `parentSessionId`，但 group membership 必须显式持久化。
4. resource locator 不等于 authorization；删除前必须验证 binding 的 group、kind、provenance 和当前 lifecycle state。
5. physical paths 是 projection/adapter detail；目录扫描不能自动提升为 membership truth。
6. group drop 不删除共享 global blobs、用户 workspace 或未声明归属的 external side effects。

## Options and trade-offs

### Option A — 保持每个 session 独立删除

- 做法：保留现有 `SessionRepo.delete(metadata)`；subagent、local 和 artifacts 由各自 caller 自行清理。
- 正确性：实现最小，但 caller 很容易遗漏 child/resource；当前 SQLite 只保证单 session 内部表清理。
- 迁移：零迁移成本。
- upstream conflict：低。
- 安全/失败：较差；并发和 partial cleanup 会散落在多个 caller。
- 结论：只能作为现状基线，不满足用户希望的一次性 group drop。

### Option B — 直接采用 OMP 式物理 folder grouping

- 做法：主 session 为 `<root>.jsonl`，同名目录收纳 child `.jsonl`、artifact files 和 `local/`；删除 root file 加递归目录。
- 优点：用户体验直观；filesystem discovery、导出和人工诊断简单；与现有 OMP behavior 接近。
- 成本：物理布局成为事实来源的诱惑；child membership 依赖路径；JSONL 多文件删除不是一个事务；Windows/symlink/partial write 需要额外处理；迁移到 SQLite/remote/object store 时重新建模。
- 安全/失败：OMP 的顺序删除可出现“主文件已删、artifact cleanup 失败”；目录扫描不能可靠区分 orphan、shared resource 和 stale child。
- 结论：适合作为 JSONL adapter 的 projection，不能单独作为跨 backend domain contract。

### Option C — `SessionGroup` + durable resource bindings，folder 只是 adapter（推荐）

- 做法：在 session domain 增加稳定 `sessionGroupId`；为 child sessions 与 file/runtime resources 保存 group membership、provenance 和 cleanup policy；JSONL/SQLite 各自把它 materialize 到适合自己的 layout。
- 正确性：membership 与 lifecycle 有单一 owner；可表达 root、child、detach、shared blob 和 cleanup receipt；可在 SQLite transaction 中原子更新 group state。
- 迁移：需要为旧 sessions 分配 group identity，并处理无法证明归属的 orphan resources；但 physical layout 可以后移或改变，wire identity/provenance 可保留。
- upstream conflict：新增逻辑集中在 fork-owned Session/Resource seam；不必把 OMP runtime singleton 或 Bun APIs 带入 agent-core。
- 性能/复杂度：比 Option A/B 初始复杂；必须先证明第二 backend/consumer 和 group cleanup contract，不能先创建空 package、manifest 或 distributed lock。
- 安全/失败：可以将 `active → dropping → dropped/cleanup_required` 变成 durable lifecycle；删除 global/shared/unowned resource 时可 fail closed。
- 结论：最符合长期可演进性，但当前只授权研究和有限 spike，不授权 production schema 或 API。

### Option D — 先做 filesystem spike，再决定 A/B/C

- 做法：用真实 Node filesystem，在临时目录中模拟 root/child/local/artifact 创建、crash、symlink、并发 drop 和 cleanup retry；不导入 production package。
- 优点：能消除 physical layout、rename/rm、Windows path 和 failure ordering 的承重未知。
- 成本：不能回答 canonical ownership 或 SQLite cross-process semantics；spike 结果不会自动成为 production design。
- 结论：应作为 C 之前的有限验证手段，而不是最终模型。

## Recommended direction

### 先冻结 domain contract，不冻结 folder API

首个设计应定义一个小而深的 SessionGroup/Resource ownership seam，例如：

```text
createGroup(rootSessionId)
attachSession(groupId, sessionId, relation)
attachResource(groupId, resourceDescriptor)
listGroup(groupId)
requestDrop(groupId, policy)
reconcileDrop(groupId)
```

这只是研究中的 interface shape，不是批准的 public API。实际命名与归属 owner 必须在 target session seam 确认后冻结。调用者不应自己拼路径、遍历 child files 或分别删除 artifacts。

### 推荐的 physical projection

对于 file-backed adapter，建议采用显式 group root，而不是依靠 sibling 扫描：

```text
<session-root>/<root-session>.jsonl
<session-root>/<root-session>/
├── sessions/<child-session>.jsonl
├── artifacts/<artifact-file>
└── local/<user-file>
```

这是对 OMP 行为的结构化改写，不是要求照搬 OMP layout。`groupId` 应成为逻辑 identity；root-session basename 只是可读路径名。若兼容旧布局，读取器可支持旧 sibling layout，写入应 clean-cutover 到明确的 group projection，并保留迁移 receipt。

### Drop 的语义

未来 `drop` 至少应满足：

1. 先拒绝或协调仍 active 的 child writers、subagents、kernels 和 processes；默认不能静默杀掉 live work。
2. 在 durable store 中记录 drop intent/state，再执行物理 cleanup；不能仅以目录不存在表示成功。
3. 每个 resource cleanup 有 idempotent receipt；进程中断后可从 `dropping` 恢复到 `dropped` 或 `cleanup_required`。
4. 主 session、child transcripts、session-owned artifacts 和 group-local files 只在 binding 明确属于该 group 时删除。
5. 共享 global blob、用户 workspace、external worktree 和未绑定资源不随 group drop 删除。
6. 失败要报告已清理与未清理资源，而不是返回一个模糊的成功/失败布尔值。

## Migration and rollout questions

**[OBSERVED]** 当前 v4 JSONL/SQLite data 没有 `sessionGroupId` 或 resource registry；旧 coding-agent JSONL session 也以独立 `.jsonl` 文件为中心。

建议迁移策略：

1. 先为新 root session 生成 group identity；旧 session 默认“一 session 一 group”，避免根据相邻文件猜测关系。
2. 只有 header/metadata、validated parent relation 和明确 path ownership 同时成立时，才允许把旧 child resource 归入已有 group；不确定的 orphan 保持 unowned 并报告。
3. 旧 artifact/local 布局先由 read adapter 兼容；写入和新 group 通过明确 projection 产生新布局。
4. migration interruption 必须可重试；在 group membership receipt 完成前，旧 reader 仍应可读取 source bytes。
5. 删除永久旧 reader/shim 前，验证 root/child/resource listing、resume、fork、export 和 drop-equivalent behavior。

## Failure and threat matrix

| 场景 | 可见结果 | committed 状态 | 恢复/限制 |
|---|---|---|---|
| child 在 drop 开始后继续写 | drop 被拒绝或进入 waiting | group 仍 `active`/`dropping` | 通过 writer lease/operation reconciliation 收敛，禁止静默覆盖 |
| 主 transcript 删除成功，artifact 删除失败 | 明确显示 partial cleanup | `cleanup_required` | 重试剩余资源；不把主文件缺失误报为完整删除 |
| process 在 group state commit 后退出 | group 保留 `dropping` | durable | reopen 时扫描 binding receipt 并继续 cleanup |
| symlink 指向 group 外 | 拒绝 resource attach/read/delete | 不 committed | realpath + containment，fail closed |
| 两个 group 引用同一 blob | 当前 group drop 成功但 blob 保留 | binding removed | 独立 blob GC 根据 references/retention 决定 |
| child parent link 缺失或损坏 | child 显示 orphan/unowned | 不自动纳入 group | 人工 repair/import，不按文件名猜测 |
| 同时两个 drop 请求 | 一个成功，另一个 idempotent no-op/明确 already-dropping | 单一 state transition | group-level CAS/lease；禁止重复物理删除竞态 |
| active isolated worktree | drop 不直接 rm 用户/共享 worktree | resource remains or cleanup requested | 仅按 owned isolation lease 终止/回收 |
| incomplete migration | source sessions 仍可读取 | group migration pending | resume migration；不要删除 source bytes |

## What evidence would change the recommendation

以下证据可能使建议退回 Option A/B 或调整 C 的边界：

- 目标 production 只会永久支持单一 file backend，且明确接受 non-atomic best-effort cleanup；
- 第二 backend/consumer 证明不需要 group-level listing、retention 或 restart recovery；
- 真实数据量显示 resource registry 的 I/O/replay 成本超过可接受预算，而 filesystem projection 足以满足授权与恢复；
- 现有 session migration 证明所有 child/resource 都有稳定、可验证的 group identity；
- OMP 最新源码与本地快照在 membership/drop semantics 上有重大差异，需要重新分类 OMP 为 behavior reference。

## Open questions

1. 用户 `/fork`、`clone`、`newSession(parentSessionId)` 是否默认加入 parent group，还是创建新 group？建议 machine-created subagent child 默认共享；用户显式 fork 默认先视为新 group，待产品语义确认。
2. `local://` 是否一定是 group-local，还是可显式 `shared` / `workspace` scope？
3. remote server session、ACP session 和 SDK caller 如何携带 group identity？
4. SQLite 是否成为唯一 canonical store，还是 JSONL 也必须支持完整 group lifecycle？
5. group drop 是否支持 archive/trash/recover，还是只支持 permanent cleanup？
6. group resource 是否需要 retention class、sensitivity/secret classification 与 export policy？
7. active subagent 被 drop 时默认 cancel、wait、detach 还是 require confirmation？
8. global blobs 何时做独立 garbage collection，如何跨 session references 计数或标记？

## Implementation gate status

**[OBSERVED]** 研究门禁要求在 durable storage/schema/migration、跨进程 concurrency、filesystem behavior 或 package boundary 变化前，先完成 current-state map、external evidence、alternatives、invariants、failure matrix、migration/rollback 和 observable acceptance contract。

**[INFERENCE]** 本研究已完成 domain framing、当前实现初步 map、OMP local snapshot 对照、候选模型比较和 failure/open-question 列表；它尚未完成：

- 目标 production session owner 的收敛；
- group schema/wire contract 的审查；
- filesystem/SQLite crash and concurrency spike；
- 旧 JSONL/SQLite data equivalence proof；
- implementation 前独立 read-only review。

因此当前 recommendation 是：**可以进入有限 filesystem spike 和 design review，不可以直接实现 production `SessionGroup` schema 或 `drop` API。**

## Sources

### Current repository

- `packages/agent/src/harness/session/types.ts` — `SessionRepo`, `SessionCreateOptions`, `parentSessionId`, and current error contract.
- `packages/agent/src/harness/session/jsonl/repo.ts` — JSONL session path, header creation, parent lineage, single-file delete, list and fork behavior.
- `packages/session-backends/sqlite-node/src/sqlite/migrations/001_initial.sql` — current SQLite tables, per-session keys and derived branch cache.
- `packages/session-backends/sqlite-node/src/sqlite/repo.ts` — transactional single-session deletion and writer lease behavior.
- `packages/coding-agent/src/core/session-manager.ts` — older coding-agent JSONL session manager and session selector integration; confirms an additional session path still exists in the checkout.
- `docs/research/README.md` and `docs/research/90-research-governance/implementation-research-gate.md` — research archive and implementation gate conventions.

### Local OMP snapshot (primary OMP evidence)

- `../oh-my-pi-main/packages/coding-agent/package.json` — package version `17.2.1`, Bun engine requirement and package identity; local snapshot, not latest, no Git metadata available.
- `../oh-my-pi-main/packages/coding-agent/src/session/session-paths.ts` — managed session root and legacy directory migration.
- `../oh-my-pi-main/packages/coding-agent/src/session/session-manager.ts` — artifact-root derivation, child-session root semantics and artifact manager adoption.
- `../oh-my-pi-main/packages/coding-agent/src/session/artifacts.ts` — lazy artifact directory and shared parent/child artifact manager behavior.
- `../oh-my-pi-main/packages/coding-agent/src/internal-urls/local-protocol.ts` — session-scoped local root, fallback root, realpath and containment checks.
- `../oh-my-pi-main/packages/coding-agent/src/task/executor.ts` — child `<id>.jsonl` creation under `artifactsDir`, parent artifact manager adoption and lifecycle setup.
- `../oh-my-pi-main/packages/coding-agent/src/session/session-storage.ts` — deletion of session file plus same-stem artifacts directory.
- `../oh-my-pi-main/packages/coding-agent/src/session/indexed-session-storage.ts` — indexed descendant-path cleanup and in-memory rollback on backend failure.
- `../oh-my-pi-main/docs/blob-artifact-architecture.md` — distinction between global content-addressed blobs and session-scoped artifacts.
- `../oh-my-pi-main/packages/coding-agent/src/debug/report-bundle.ts` — diagnostic bundle discovery of artifacts and child JSONL files.
- `../oh-my-pi-main/packages/coding-agent/src/task/isolation-ownership.ts` — live owner marker for isolated worktree cleanup.

### Public OMP reference (secondary only)

- [OMP feature portability matrix](../90-research-governance/omp-feature-portability-matrix.md) — previously pinned public source references and the rule that OMP behavior is a reference, not a direct production dependency.
- [OMP repository](https://github.com/can1357/oh-my-pi) — public repository metadata; not used as evidence for the local snapshot's exact source revision.
