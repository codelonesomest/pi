# Oh My Pi 全路线图 Feature Portability Matrix

- 状态：**一手 source 对照完成；不授权 production 实现**
- 日期：2026-08-01
- OMP evidence pin：[`can1357/oh-my-pi@09a7c865636457c50ed75fc3b1a7cc21ef72c105`](https://github.com/can1357/oh-my-pi/tree/09a7c865636457c50ed75fc3b1a7cc21ef72c105)
- 范围：将 OMP 作为持续维护的产品、行为和 source-code reference，逐项审计 Harness Evolution 的 01–16 个专题；判断每个机制应直接拒绝、仅作行为参考，还是可经过本项目 seam 改造后移植。
- 非范围：不把 OMP 当作任何 production gate 的替代证明；不 wholesale copy OMP；不创建 package、公开 API 或生产实现。

## 1. 结论

[OBSERVED] OMP 并非只有 extension examples：其 pinned `pi-coding-agent` source 中已有 native `task`、`hub`、`irc`、agent registry/lifecycle、async jobs、persistent eval backends、MCP manager、internal-URL router、plan-mode、goal runtime、todo、retry/fallback 与 compaction modules。其 package manifest 同时表明这些实现依赖 OMP 的 Bun runtime、workspace catalog 和 fork-owned package graph。[S01](#s01-package-and-license) [S05](#s05-native-task-and-agent-definitions) [S06](#s06-native-coordination-runtime) [S07](#s07-eval) [S10](#s10-mcp) [S12](#s12-plan-mode) [S13](#s13-native-goal-and-todo-no-scoped-kanban-ledger-counterpart)

[INFERENCE] OMP 应作为本项目的**主要产品和机制参考**，而不是只作为 fork strategy 或 URI 安全的有限参考；但它不是可直接引入的 library，也不应成为本项目 durable truth 的 owner。每一项移植都必须先经过本项目的 [Implementation 前研究门禁](implementation-research-gate.md)：用 OMP behavior 做 oracle，围绕本项目的 ownership、runtime、durability、authorization 与 package seam 重建实现。

[OBSERVED] OMP 的根许可证是 MIT，允许复制、修改和分发，但要求保留 copyright/license notice。[S01](#s01-package-and-license)

[INFERENCE] 许可证允许并不等于 architecture admission。OMP 的 `@oh-my-pi/pi-coding-agent` 是 Bun-first、workspace-catalog product package；直接依赖其 private source、把其 process-global singleton 当 shared durable state、或把 OMP package topology 复制到本项目，都会同时增加 runtime coupling、upstream conflict 和替换成本。[S01](#s01-package-and-license)

### 1.1 分类语义

| 分类 | 含义 |
|---|---|
| **Adapt** | 已观察到可复用的机制，但必须在本项目自己的 seam、types、policy 和 tests 下重建；不得直接把 OMP implementation 当 public dependency。 |
| **Behavior reference** | 可作为 UX、状态机或 failure behavior 的 oracle；当前 evidence 不足以支持代码/ownership 复用。 |
| **No verified counterpart** | 在已审计 OMP scope 内没有能满足本专题核心 domain contract 的实现；不得把相邻功能误写成 counterpart。 |
| **Reject** | 机制与本项目已冻结的不变量冲突，不能进入目标设计。 |

没有任何完整专题是“直接 port”。最小可移植对象是一个经过 source pin、license attribution、dependency review、failure review 和 observable compatibility test 的机制，而不是一个目录或 package。

### 1.2 跨专题结论

1. **OMP 的 native runtime 很有价值，但大多是 process-local。** `AsyncJobManager`、`AgentRegistry`、`IrcBus` 与 lifecycle manager 已有 stable IDs、owner scoping、delivery retry、idle/park/revive 和 cancellation；它们仍以 process-global `Map`、timer 和 live session 为中心。它们能启发 Runtime/Team adapter，不能充当 durable team ledger、cross-host lease 或 canonical workflow state。[S06](#s06-native-coordination-runtime)
2. **OMP 已经证明 product features 可以在 fork 中深度演化。** tool presentation、editor, settings, plan, task, eval, MCP 和 workflow affordance 都不是“只留 Pi 底层”的简单 extension；这强化本项目“upstream-compatible foundations + fork-owned deep modules + thin composition adapters”的路线，而非全仓重写或一 feature 一 package。[S01](#s01-package-and-license) [S15](#s15-upstream-port-discipline)
3. **保留三个 orchestration owners。** OMP native task/hub/IRC 可以提供 Subagent/Runtime behavior；它不能把 Agent Team、Goal/Kanban 和 Dynamic Workflow 合并成一套 process-local tool state。Team 仍拥有 durable roster/mailbox/lease；Workflow 仍拥有 program/graph/checkpoint；Work ledger 仍拥有 guarded task transitions。[Agent Teams](../05-agent-teams/research.md) · [Goal/Kanban](../12-goal-kanban/research.md) · [Dynamic Workflows](../16-dynamic-workflows/research.md)

## 2. 适用的移植门禁

每次从下表选择一个机制前，work-package receipt 必须额外回答：

1. **机制，不是目录。** 精确记录 OMP pin、symbol、observed behavior、测试/文档证据，以及本项目要保留和故意不保留的行为。
2. **单一 owner。** 指明 target domain 是否为 TUI/coding-agent composition、future Runtime、future Work、Session composition 或 session-agnostic artifacts；不得让 OMP-like global manager 与本项目 canonical Session/ledger 双写同一事实。
3. **runtime 与 dependency separation。** 先确认 Node/Bun、native binary、provider SDK、filesystem、OAuth、TUI and package dependencies；不得以 private-source import 或 workspace link 伪装为 portable seam。
4. **failure admission。** 对 cancellation、timeout、process loss、duplicate delivery、stale state、untrusted project configuration、path traversal、missing artifact 与 unsupported platform 写出可见 outcome；side effect 不得因“OMP 会重试”而自动安全重放。
5. **reference test first。** 选择一个小机制，通过本项目行为测试或 disposable spike 对齐 OMP observed behavior；任何 durable result 仍必须通过本项目 Session/receipt gates。

### 2.1 通用 stop conditions

以下机制一律不得直接带入目标设计：

- session-local numeric artifact ID 充当 durable/provenance identity；
- resolver 在 caller binding 缺失时以 global registry / first match 继续搜索并把结果视为 authorized；
- JSONL sidecar write、best-effort artifact copy 或 `EPERM` direct overwrite 充当 canonical physical commit；
- process-global `Map`、PID、in-memory Promise、TUI component state 或 kernel variable 充当 restart-safe domain truth；
- mutable `local://` plan file、tool text、Todo snapshot 或 chat message 充当 approved plan/task receipt/board state；
- OMP UI confirmation、frontmatter source precedence 或 prompt instruction 充当 durable authorization；
- unbounded `Promise.all`, generic retry, or reconnect 充当 non-idempotent side-effect recovery。

## 3. 全路线图矩阵

| # / 专题 | 已观察的 OMP counterpart | 分类与可移植 seam | 本项目 target、前置条件与最小 admission | 不得照搬 / stop condition |
|---|---|---|---|---|
| **01 Tool UI/UX** | OMP custom tools expose `renderCall`/`renderResult`; `ToolExecutionComponent` carries per-call expanded/partial state, catches renderer failures and falls back; the default renderer selects pending/running/error/done presentation plus collapsed/expanded views.[S02](#s02-tool-ui) | **Adapt**：保留 ToolRun lifecycle、partial/result/error and compact/expanded behavior；不复用 OMP `Component`/ANSI renderer。 | 保留在 coding-agent + TUI composition，先定义 UI-neutral `ToolExecutionViewModel` for one existing read-like tool. Test partial → complete/error, expansion, renderer failure fallback, and custom/built-in parity. Structured attachment work can enrich it later but is not a reason to delay a non-durable card. | TUI `Component`, `Theme`, terminal image and ANSI-to-HTML output are adapters, not shared web data. Stop if the proposed model treats UI text as tool outcome truth or silently loses framework-owned details. |
| **02 IDE 级聊天编辑器** | OMP terminal editor has grapheme/word segmentation, visual width/CJK wrapping, paste markers, history, undo and autocomplete; coding-agent owns app keybinding interception, external-editor flow and host APIs.[S03](#s03-editor) | **Adapt**：复用 text-command and host-bridge lessons，不能把 terminal editor implementation 当 IDE component。 | Keep terminal editing in TUI; prove one Pi-owned host-editor bridge (`set text`, multiline edit request, cancel) against one host. Validate Unicode/grapheme cursor behavior, paste, cancellation and external-editor round trip. | Stop if browser/ACP/IDE code imports terminal layout state or if a custom editor is claimed to be a stable cross-host contract without a second host. |
| **04 Subagents / Frontmatter** | Native `task` parses bundled/user/project/plugin Markdown definitions, exposes typed output schema/mode, tool restrictions, model/effort and isolated execution; discovery has explicit precedence.[S05](#s05-native-task-and-agent-definitions) | **Adapt**：frontmatter parser/precedence and typed delegation input are strong references; OMP definition source is not the policy owner. | Initially a coding-agent deep module; future Work/Runtime seam only after a second real consumer. First slice returns `{ definition, source, capabilities }`, validates required fields and duplicate precedence, then performs separate caller-owned admission. | Do not treat project-local Markdown or interactive confirmation as authorization; do not preserve OMP tool-name allowlist as an unreviewed policy contract. |
| **05 Agent Teams** | Native task + async jobs + hub + IRC + registry/lifecycle provide stable IDs, owner-scoped jobs, process supervision, mailbox delivery, idle/park/revive and subagent session reopening.[S06](#s06-native-coordination-runtime) | **Adapt, runtime-only**：these are valuable Subagent/Runtime primitives; they are not a durable Team domain. | Future fork-owned Work + Runtime deep modules after Session/operation receipts and A3 sole-writer ownership are closed. First proof may cover one bounded agent run with lifecycle/progress/cancel and explicit parent result receipt. | Stop before Team implementation if the only state is process-global registry/mailbox/job `Map`; no direct port without durable team/member/task/mailbox events, CAS lease, redelivery dedupe and restart recovery. |
| **06 Execution Eval** | Native EvalTool supplies persistent JS/Python (and optional Ruby/Julia) backends, per-language session namespacing/reset, streaming output, display/status, artifact sink, cancellation and idle timeout pause/resume around bridges.[S07](#s07-eval) | **Adapt**：`ExecutorBackend` is a deep adapter seam; persistent-kernel behavior is a useful oracle. | Begin in coding-agent with a Node-compatible single-language cell only after runtime/dependency research; extract future Eval module only after two adapters. Test state reuse, reset isolation, cancel/timeout, output cap/spill and explicit kernel cleanup. | OMP’s Bun/runtime/session wiring must not become public contract. Do not claim host-restart kernel recovery, canonical artifact durability or arbitrary-language support from persistent in-process state. |
| **07 Settings UI/UX** | OMP has typed configuration with global/project/override merge, a searchable `SettingsList`, curated selector and preview/cancel behavior.[S08](#s08-settings) | **Adapt**：effective-value/source and searchable selector patterns are portable. | Retain current coding-agent config + TUI ownership. First read-only `effective setting` query returns value and source across default/global/project; test precedence and a preview cancellation path. | Do not turn OMP JSON persistence/deep merge into a new canonical policy store; stop if source provenance, transactional edit or headless behavior is not explicit. |
| **08 Tool / Skill / Device Discovery** | OMP router mounts `xd://` only through caller session context; the protocol rejects malformed targets and unmounted sessions. It supports a unified route surface and explicit handler registration.[S09](#s09-tool-skill-and-device-discovery) | **Adapt**：descriptor discovery separated from execution is useful; `xd://` is an ephemeral device mechanism, not a finalized universal URI contract. | Coding-agent capability/catalog seam first. Admit one read-only optional descriptor that only materializes through existing approval at a turn boundary; validate unknown device, unmounted session and no discovery-implies-approval path. | Do not freeze `xd://` as a public Bundle scheme before URI registry research; do not let a discovery catalog activate a tool, MCP server or skill without policy. |
| **09 Built-in MCP** | OMP native manager covers multi-source config, stdio/HTTP/SSE, OAuth material, deterministic tool naming/sorting, cache/deferred tools, resource subscriptions, reconnect epochs/circuit breaker and one-call connection retry.[S10](#s10-mcp) | **Adapt**：manager/transport/tool-bridge separation is a strong source reference. | Future fork-owned MCP module, with coding-agent adapter. First admission: explicit read-only stdio lifecycle `start → initialize → list tools → close`, cancellation and structured error. Persisted operations wait on Runtime/Session receipt gates. | No automatic replay of side-effecting MCP calls; no process-global manager as durable authority; no OAuth/token or local-path forwarding port without security review. |
| **10 Context Compaction** | `compact-modes.ts` is a dependency-light parser/table; one-off mode overrides do not mutate settings. OMP stores compaction entries and offers soft/remote/snapcompact strategies.[S11](#s11-context-compaction) | **Behavior reference + narrow Adapt**：the parser/override separation is portable; OMP compression is not proof of this project’s continuous compartments or retention policy. | Future Context deep module after Session semantics. First slice: pure mode-selection API plus one persisted compaction receipt retaining a verified tail boundary; test override non-mutation. | Do not port snapcompact as generic context truth, or represent image archive/model behavior as search/expand/memory equivalence. |
| **11 `/plan` / Web Review** | OMP has plan-mode state, safe plan-title/path resolution, `local://` plan-file lookup and interactive plan-approval plumbing; its session HTML export is a separate static projection.[S12](#s12-plan-mode) | **Adapt** for local plan-file hardening and runtime read-only-mode behavior; **Behavior reference** for approval interaction. | Future Work plan domain + TUI/browser adapters, after durable plan artifact/approval receipt seam. First test: path traversal rejection, a fixed plan artifact digest, runtime-enforced read-only mode and approval-to-execution binding. | A mutable `local://` file, UI popup, static HTML export or title string must not be the approval authority. Stop if approval is not actor/digest/version bound. |
| **12 GoalBuddy / Kanban** | OMP has a native, session-backed one-active-goal runtime with ID/objective/status, token/time accounting, and create/pause/resume/complete/drop behavior; it also has Todo phased text/status snapshots. No scoped Kanban/task-ledger/claim-lease/review-receipt counterpart was verified.[S13](#s13-native-goal-and-todo-no-scoped-kanban-ledger-counterpart) | **Adapt, bounded Goal primitive only**：objective/status/budget/continuation mechanics are a behavior reference; neither OMP Goal nor Todo is the target Work ledger. | Future Work domain after canonical Session/operation contracts. A separately admitted Goal behavior slice may cover one objective plus pause/resume/budget accounting; the Goal/Kanban slice must still independently define durable task records, guarded transitions and a read-only projection. | Do not treat OMP one-active-goal state or Todo snapshot as Goal/Task Ledger truth; stop if board drag mutates status without command guards/receipts. |
| **13 Reliability / Unattended** | OMP has bounded assistant retry/fallback, rate-limit handling, persisted retry annotations, MCP reconnect/backoff/circuit breaking, owner-scoped async cancellation and retrying delivery sinks.[S06](#s06-native-coordination-runtime) [S10](#s10-mcp) [S14](#s14-turn-recovery) | **Adapt** for failure taxonomy and bounded in-process recovery; not proof of unattended/restart-safe supervision. | Future Runtime/Reliability deep module. First read-only operation slice records an interrupted/unknown outcome and explicitly reconciles after reopen; test no blind replay, retry limits and user diagnostics. | Process-local delivery retry, timer, provider session or recovered chat turn cannot imply host-restart operation receipt; stop if any non-idempotent effect is replayed merely because it is retriable. |
| **14 Package / Upstream Sync** | OMP maintains a focused-port guide, historical upstream marker, explicit package-manifest merge/replacement guidance, stale-concept search and regression traps.[S15](#s15-upstream-port-discipline) | **Adapt**：this is a process pattern, not package topology. | Fork maintenance/release ownership; no Session dependency. First slice is a scoped-port receipt recording source range, public seams, deliberate omissions, dependency diff and focused regression proof. | Do not copy OMP Bun, package scope, workspace catalog or historical marker mechanically; stop if a port has no affected-caller map or local divergence review. |
| **15 Session Undo / Redo** | OMP normalizes checkpoint/rewind tool outcomes and reloads a completed rewind report from persisted custom messages; its session tree preserves historic entries and leaf navigation.[S16](#s16-checkpoint-and-rewind) | **Behavior reference**：tool-result normalization is adaptable, but it is not a multi-step cursor/redo/workspace checkpoint model. | Session/history only after v4 canonical semantics. First slice persists/reloads typed checkpoint observation and proves cursor-only undo/redo retains history; workspace restore remains separate. | Do not promise arbitrary shell/MCP reversibility, represent an external rewind report as canonical cursor state, or introduce workspace byte restore without digest/conflict contract. |
| **16 Dynamic Workflows** | OMP has bounded task fan-out, async jobs, structured child output, IRC/hub coordination, eval bridge primitives and a `workflowz` prompt notice; no verified durable versioned program/graph/checkpoint engine is present.[S05](#s05-native-task-and-agent-definitions) [S06](#s06-native-coordination-runtime) [S07](#s07-eval) | **Adapt, bounded primitives only**：use task dispatch/supervision behavior as reference; no direct workflow runtime port. | Future Work + Runtime deep module after operation receipts, policy/budget and canonical Session gates. First proof: typed three-step sequential program with bounded concurrency, stop-on-failure and cancellation; no persistence/retry/graph claim. | `workflowz` prompting, eval variables, process jobs or task batch cannot be durable WorkflowDefinition/Run/NodeAttempt truth. Stop if program digest, checkpoint, budget, approval and recovery are absent. |

## 4. Ownership synthesis

```text
OMP observed product mechanism
  → retain behavior / external source evidence
  → Pi-owned deep module at the smallest true seam
  → adapter into coding-agent + TUI/browser
  → canonical Session / receipt only where durable facts are required
```

| Target owner | OMP inputs that may inform it | Explicit non-owner |
|---|---|---|
| Existing coding-agent + TUI composition | tool view model, editor host bridge, settings selector, frontmatter reader, device catalog | session truth, work ledger, browser state |
| Future Runtime/Reliability deep module | task execution, async jobs, hub process supervision, retry/fallback, MCP reconnect | Team task ownership, Workflow graph, provider/TUI singleton state |
| Future Work deep module | plan lifecycle, task vocabulary, bounded workflow dispatch behavior | Todo snapshot, IRC mailbox, task output prose as authoritative ledger |
| Session composition | caller-bound resource bindings, plan/checkpoint links, durable operation receipts | OMP JSONL sidecar, registry fallback, numeric artifact handles |
| Future session-agnostic artifacts module | immutable byte/CAS lessons, output spill/read policy | session ID, leaf, authorization, artifact numeric allocation |
| Future MCP / Eval deep modules | OMP transport/backend interfaces and lifecycle behavior | coding-agent-private imports, OMP Bun runtime, current OMP global singletons |

This keeps the existing package conclusion intact: prove modules and two real adapters first; only then decide whether an independent package reduces upstream touch points and has one canonical owner.

## 5. What this changes now

1. **OMP is now a roadmap-wide reference corpus.** Future receipts for every listed topic should check the corresponding row before doing generic competitor research.
2. **The source-first rule is tighter.** OMP README feature claims or a source-code search snippet are triage only; a selected mechanism must be read at the pinned file/test/documentation level before code changes.
3. **The current production ordering does not change.** Session canonical semantics, importer equivalence, SQLite append/receipt/concurrency proof, A3 ownership, structured ToolResult attachments and artifact package admission remain gates for all durable feature work.
4. **Product-track work remains possible only through its own receipt.** Tool presentation, editor host bridge, settings provenance, frontmatter parsing and catalog discovery may have smaller non-durable slices, but this matrix itself is not their authorization.
5. **No OMP compatibility layer.** We port selected semantics to Pi-owned interfaces; we do not emulate OMP numeric URI forms, runtime globals, package graph or legacy persistence merely to claim compatibility.

## 6. Evidence ledger

All OMP links below are pinned to `09a7c865636457c50ed75fc3b1a7cc21ef72c105`.

### S01 Package and license

- [MIT license](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/LICENSE)
- [coding-agent manifest](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/package.json) — Bun engine, workspace/catalog dependencies, exports and package ownership.

### S02 Tool UI

- [custom-tool call/result renderer contract](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/extensibility/custom-tools/types.ts)
- [tool execution component and guarded renderer fallback](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/modes/components/tool-execution.ts)
- [default tool lifecycle renderer](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/tools/default-renderer.ts)

### S03 Editor

- [TUI editor](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/tui/src/components/editor.ts)
- [interactive custom editor](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/modes/components/custom-editor.ts)
- [external editor path](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/modes/interactive/external-editor.ts)

### S04 Session, resource and URIs

- [internal URL types / ResolveContext](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/internal-urls/types.ts)
- [router](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/internal-urls/router.ts)
- [`local://` handler](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/internal-urls/local-protocol.ts)
- [`artifact://` handler](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/internal-urls/artifact-protocol.ts)
- [artifact manager](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/session/artifacts.ts)
- [blob/artifact architecture](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/docs/blob-artifact-architecture.md)

### S05 Native task and agent definitions

- [task tool](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/task/index.ts)
- [task contracts](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/task/types.ts)
- [agent discovery](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/task/discovery.ts)
- [agent parser/bundled definitions](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/task/agents.ts)
- [structured subagent execution](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/task/executor.ts)

### S06 Native coordination runtime

- [async job manager](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/async/job-manager.ts)
- [hub tool](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/tools/hub/index.ts)
- [IRC bus](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/irc/bus.ts)
- [agent registry](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/registry/agent-registry.ts)
- [agent lifecycle/park/revive](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/registry/agent-lifecycle.ts)

### S07 Eval

- [EvalTool](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/tools/eval.ts)
- [ExecutorBackend contract](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/eval/backend.ts)
- [JavaScript backend](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/eval/js/index.ts)
- [Python backend](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/eval/py/index.ts)

### S08 Settings

- [settings configuration and merge](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/config/settings.ts)
- [settings selector](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/modes/components/settings-selector.ts)
- [TUI settings list](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/tui/src/components/settings-list.ts)

### S09 Tool, skill and device discovery

- [`xd://` protocol](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/internal-urls/xd-protocol.ts)
- [internal URL router](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/internal-urls/router.ts)

### S10 MCP

- [MCP manager](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/mcp/manager.ts)
- [MCP tool bridge](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/mcp/tool-bridge.ts)
- [MCP configuration loader](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/mcp/config.ts)

### S11 Context compaction

- [compaction mode parser/metadata](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/session/compact-modes.ts)
- [session compaction format](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/docs/session.md)
- [semantic-compression skill](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/.omp/skills/semantic-compression/SKILL.md)

### S12 Plan mode

- [plan mode state](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/plan-mode/state.ts)
- [approved plan resolution](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/plan-mode/approved-plan.ts)
- [plan file reader/listing](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/plan-mode/plan-files.ts)

### S13 Native Goal and Todo no scoped Kanban ledger counterpart

- [Goal state](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/goals/state.ts)
- [Goal runtime](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/goals/runtime.ts)
- [Goal tool](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/goals/tools/goal-tool.ts)
- [Goal runtime behavior coverage](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/test/goals/goal-runtime.test.ts)
- [Todo tool](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/tools/todo.ts)
- Scoped pinned-source searches found no `kanban` match under `packages/coding-agent/src` and no verified Goal/task-ledger claim/lease/review-receipt counterpart in the audited scope. This is evidence only for the searched source scope, not a claim about every OMP package or integration.

### S14 Turn recovery

- [turn recovery](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/session/turn-recovery.ts)
- [retry/fallback chains](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/session/retry-fallback-chains.ts)

### S15 Upstream port discipline

- [porting from pi-mono](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/docs/porting-from-pi-mono.md)
- [OMP root manifest](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/package.json)

### S16 Checkpoint and rewind

- [checkpoint-entry normalization](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/packages/coding-agent/src/session/checkpoint-entries.ts)
- [session operations: export/share/fork/resume](https://github.com/can1357/oh-my-pi/blob/09a7c865636457c50ed75fc3b1a7cc21ef72c105/docs/session-operations-export-share-fork-resume.md)

## 7. Open evidence that still blocks a port

Before an individual item can leave research, collect its exact target-Pi caller map, actual local types/dependency versions, target OS/runtime support, current test conventions and a failure matrix. In particular:

- UI rows need a second host before a general UI-neutral public contract is extracted.
- Definition/discovery rows need an explicit trust and policy owner before project-local content can affect tools/models.
- Team, Goal/Kanban, Reliability and Workflow rows need durable operation/task/mailbox receipts and restart evidence.
- Eval and MCP rows need dependency, credential, process, cancellation and output-retention review.
- Undo and Compaction rows need explicit persistence and recovery evidence.

This matrix narrows where to look and what not to copy. It does not satisfy any of those implementation gates by itself.
