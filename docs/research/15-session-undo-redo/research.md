# 多步会话与文件 Undo/Redo 研究

- 状态：架构方向建议
- 更新日期：2026-07-31
- 范围：OpenCode 式多步 `/undo`、`/redo`，conversation boundary、workspace snapshots、redo invalidation、branching、crash recovery 与非文件副作用
- 核心结论：采用“durable session cursor + workspace checkpoint chain”，而不是删除消息或调用 `git reset`；undo/redo 是可逆的视图/工作区切换，开始新对话或产生新 mutation 时才 commit 为分支

## 1. 目标体验

用户应能：

1. `/undo` 一次，撤回最近一轮 user message、后续 assistant/tool messages，以及这轮造成的文件变化；
2. 连续 `/undo` 多次，逐个回到更早 user-turn boundary；
3. `/redo` 多次，按原顺序恢复消息和文件变化；
4. undo 后修改 prompt 并重新提交，形成新分支；
5. session 或进程重启后仍保留当前 undo cursor/redo path；
6. 清楚看到将撤销/恢复多少 turns、messages、files 与不可撤销副作用。

这里的“undo”与编辑器输入框 `Ctrl+Z` 完全不同。应分别命名为 editor undo 与 session undo，避免 keybinding/mental model 冲突。

## 2. OpenCode 参考实现

OpenCode 当前公开文档说明：

- `/undo` 撤销最后 user message、后续 response 及 file changes；
- 可多次 undo；
- `/redo` 仅在 undo 后可用，并恢复 file changes；
- TUI 默认快捷键为 leader + `u/r`；
- 官方文档称依赖 Git repository。

截至 2026-07-31 的实现进一步表明：

### App command

`use-session-commands.tsx`：

- session 保存一个 `revert.messageID` cursor；
- undo 找到 cursor 之前最后一个 user message；
- 若 session 正在工作，先 interrupt；
- 调 `session.revert.stage(sessionID, messageID)`；
- 把被撤 message 的 prompt parts 放回 composer；
- UI 只展示 cursor 之前的 user messages；
- redo 找 cursor 之后下一 user message并重新 stage；到末尾则 clear revert；
- 新 prompt 执行时由 core commit staged revert，从而截断/分叉后续路径。

### Core revert

`packages/core/src/session/revert.ts`：

- 根据目标 message sequence 找之后的 assistant messages；
- 每个 assistant message 带 snapshot start/files 信息；
- 第一次 stage capture 当前 workspace 作为 `original` snapshot；
- 连续 stage 时在 original 与目标 checkpoint 间 restore；
- session revert state 保存 boundary message、original snapshot、diff/files；
- `clear` 恢复 original workspace，即 redo 到最前端；
- `commit` 发布 committed event，后续原 history不再是 active continuation。

值得借鉴的关键不是 API 名称，而是：**undo 不立即删除历史，redo 通过移动一个 boundary cursor 实现，文件 restore 始终相对于第一次 undo 时保存的 original state。**

## 3. 当前 Pi 基线

Pi 已有：

- append-only JSONL session tree；
- 每个 entry 有 `id` / `parentId`，可形成 branches；
- `/tree` 导航与 branch summary；
- `/fork` 创建 branched session；
- `/edit` 从旧 user message 开新分支；
- editor 自身 undo keybinding；
- compaction/branch summaries；
- file-operation tracking 可记录 read/modified paths。

缺口：

- 没有 session-level undo/redo command；
- session entry 与 workspace checkpoint 没有统一 boundary；
- branch navigation 默认不恢复文件；
- redo path/cursor 没有 durable state；
- shell/MCP/外部 side effects 没有 undo 语义。

Pi 的 session tree 是优势：undo 后新提交天然可成为 sibling branch，不必物理删除原 entries。

## 4. 语义边界

建议首版一单位 undo = 一个 **completed user turn group**：

```text
user message
  + assistant turn(s)
  + tool calls/results
  + follow-up/steering messages consumed in same operation
  + compaction/config entries caused by the operation
  + workspace mutation set
```

Boundary 由 durable `turn_started/turn_finished` 或 user entry + operation receipt定义，不用单纯比较 timestamp。

以下不默认成为独立 undo step：

- model/thinking 设置切换；
- UI toggle；
- read-only tool；
- background index更新；
- retry attempts（属于同一 logical turn）。

可在高级 history UI 允许回到任意 safe checkpoint，但 `/undo` 默认按 user turn，符合用户预期。

## 5. Domain Model

```ts
interface SessionRevertState {
  id: string;
  sessionId: string;
  originalLeafId: string;
  originalWorkspaceCheckpoint: CheckpointId;
  cursorBoundaryId: string;
  cursorLeafId: string | null;
  stagedAt: string;
  affectedFiles: WorkspaceFileDelta[];
  warnings: RevertWarning[];
  generation: number;
}

interface TurnCheckpoint {
  turnId: string;
  beforeLeafId: string | null;
  afterLeafId: string;
  workspaceBefore: CheckpointId;
  workspaceAfter: CheckpointId;
  mutatedFiles: string[];
  sideEffects: SideEffectReceipt[];
  status: "completed" | "failed" | "interrupted";
}
```

`original*` 在第一次 undo 时冻结；连续 undo/redo 只移动 cursor。redo 不依赖重放 tools，而是恢复既有 message path + workspace checkpoint。

## 6. Session Cursor

Pi session tree 不应删除 entries。prompt/session view 应接收 active cursor：

```ts
interface ActiveSessionView {
  leafId: string | null;
  stagedRevert?: SessionRevertState;
}
```

- 无 staged revert：active branch 到 current leaf；
- staged revert：active view 到目标 boundary前 leaf；
- undo：cursor 移向 parent completed user turn；
- redo：沿 `originalLeafId` 的已保存 ancestry 向前；
- clear redo：cursor 返回 original leaf；
- commit：把 cursor 设为正式 active leaf，新消息成为 sibling branch；
- old path 保留在 tree，可导航/恢复。

不能依赖“message ID 字符串可比较即时间顺序”；Pi 应使用 tree ancestry/entry sequence。

## 7. Workspace Checkpoint Engine

### 7.1 不能直接依赖用户 Git working tree

禁止用 `git reset --hard` / `git checkout .`：

- 会覆盖用户在 agent 外的改动；
- 会碰 staging/index/branch；
- 多 session/worktree 下不安全；
- untracked files与ignored files语义复杂；
- 用户 repo可能没有 Git。

OpenCode 可借鉴 snapshot approach，但在 Pi 中实现为独立 content-addressed workspace snapshots，不操纵用户 Git history/index。

### 7.2 Capture

每个 mutating operation/turn：

1. tool execution前记录候选 paths或 workspace observation；
2. 捕获 touched file 的 before blob/mode/existence；
3. tool完成后捕获 after blob/mode/existence；
4. 生成 content-addressed blob与 delta；
5. durable checkpoint receipt后才标 turn complete；
6. large/binary/symlink 用专属 policy；
7. store 可 deduplicate/compress。

首版可只支持 built-in edit/write tools准确报告的 touched paths。Shell 等任意 mutation默认无法完整捕获，必须警告；后续可用 filesystem watcher + pre/post snapshot扩大覆盖。

### 7.3 Restore

restore 前做 three-way safety check：

```text
expected current blob (checkpoint source)
actual workspace blob
requested target blob
```

- actual == expected：安全 restore；
- actual 已等于 target：视为 already applied；
- actual 两者都不是：发生 out-of-band change，不能静默覆盖；
-可选择 preview/skip/stash-to-artifact/force（force需明确用户批准）。

每次 restore 使用 atomic temp+rename、保留 mode/symlink、校验 digest，并产 receipt。

## 8. 多步 Undo 算法

```text
undo:
  interrupt/reconcile active operation
  choose previous completed turn boundary
  on first undo capture original leaf + workspace checkpoint
  compute target workspace checkpoint
  preflight conflicts and irreversible effects
  restore target
  append revert_staged event with new cursor/generation
  refill composer from undone user prompt (optional setting)
```

连续 undo：

- original leaf/checkpoint保持不变；
- target 向 ancestry 更早移动；
- restore 只处理 current→target 的 delta；
- UI显示 `Undo 3 · 7 messages hidden · 5 files changed`。

## 9. 多步 Redo 算法

```text
redo:
  require staged revert
  choose next turn on saved original path
  preflight workspace conflicts
  restore next checkpoint
  move cursor forward
  if cursor reaches original leaf:
    clear staged revert
```

Redo **不重新调用 model/tools**；它恢复此前存在的 history/file state。若希望重新生成，应 undo 后 resubmit prompt，形成分支，而不是 redo。

## 10. Redo Invalidation 与分支

以下动作会 commit staged revert并终止 shortcut redo path：

- 提交新 user prompt；
- 执行 mutating tool/command；
- 导航到另一个 branch并修改；
- 显式 `/undo commit`；
- session migration使原 checkpoint不兼容。

但“终止 shortcut redo”不等于销毁旧 history；旧 original path仍在 session tree，可通过 `/tree` 或“Redo branch”恢复。

只读操作、查看 diff、改变 UI、复制 prompt 不应清除 redo。

## 11. 与 `/tree`、`/edit`、`/fork` 的关系

统一成 cursor/branch primitives：

- `/undo`：沿当前 original path回退一 turn并同步 workspace；
- `/redo`：沿同一路径前进；
- `/edit`：回到旧 user boundary，prefill composer，新提交建 sibling branch；
- `/tree`：任意 branch导航；可选是否 restore workspace；
- `/fork`：把选中 cursor/path复制成新 session；
- branch summary：离开 branch时可生成，但不是 undo correctness前提。

未来最好让 `/tree` 显示 workspace checkpoint availability 与 side-effect warnings。

## 12. Side Effects

文件恢复不等于整个 turn 被撤销。以下通常不可自动 undo：

- shell启动/kill process；
-数据库 mutation；
- Git commit/push；
- GitHub issue/PR/comment；
- package install/lifecycle；
- deploy/cloud mutation；
- MCP remote side effects；
- emails/payments/webhooks。

每 tool 声明：

```ts
interface ToolReversibility {
  effect: "none" | "workspace" | "external" | "mixed";
  reversible: "automatic" | "compensatable" | "manual" | "unknown";
  compensation?: string;
}
```

Undo preflight 显示：

```text
This will restore 4 files.
Not automatically reversible:
- GitHub comment #123
- process `web` is still running
[Continue file/session undo] [Inspect] [Cancel]
```

不能声称整个 turn已撤销；receipt中明确 partial undo。

## 13. Concurrent 与 Out-of-band Changes

- active agent stream先 interrupt并等待 durable settle；
- running tool若不可取消，进入 reconciliation；
- other Pi session可能修改同一 workspace；
- user editor修改需 conflict check；
- team worktree中的 undo限定该 worktree；
- shared main workspace不允许静默跨 session restore；
- watcher只是辅助，digest才是 authority。

推荐 team agents用 isolated worktree，减少 undo冲突。

## 14. 崩溃恢复

Restore 是 transaction-like operation：

```text
revert_restore_started
file_restore_applied(path, beforeDigest, afterDigest)
revert_restore_finished
revert_staged
```

崩溃后：

1. reducer发现 unfinished restore；
2.逐 file检查 actual digest；
3. already target则补 receipt；
4. still source则继续；
5. unknown digest则 `uncertain` 并阻塞；
6.只有 workspace与session cursor一致后发布 staged/cleared。

session cursor event必须在文件 restore完成后提交，避免 UI history和磁盘不一致。

## 15. Compaction 与 Context

- raw session tree保留，因此 compaction不妨碍 undo；
- cursor回到 compaction前 branch时可重新计算 prompt view/使用已有 summary；
- summary entry属于 branch，不应复制到不匹配 sibling；
- context compartments按 raw range/branch引用；
- undo 不删除 memory，但若 memory来自被撤 branch，需 provenance并在该 branch superseded 后不自动当事实。

## 16. UI/UX

### Commands

```text
/undo                  undo one completed user turn
/undo 3                preview and undo three turns
/undo preview [n]      show messages/files/effects
/redo                  redo one
/redo 3                redo three
/undo commit           keep current cursor as active branch
/undo abort            redo all to original state
```

`/undo abort` 比 `/redo all`更清楚地表示取消 staged undo；UI仍可提供 Redo all。

### TUI

- footer/status：`Undo 2/5 staged`；
- transcript中被撤部分折叠而非删除；
- diff preview 显示 file additions/deletions/modes；
- composer默认恢复最近被撤 user prompt；连续 undo时设置可决定是否覆盖已有草稿；
- new prompt前提示将从当前 cursor 建立新 branch；
- conflict dialog逐 file选择；
- session tree标 original/cursor/redo path。

### Keybindings

不能占用 editor `undo`。新增 configurable app bindings到 `DEFAULT_APP_KEYBINDINGS`，例如 leader sequence；不硬编码 `ctrl+x` 检查。

## 18. 隐私与安全

Snapshot可能保存已被用户删除的 secrets：

- storage目录权限；
- optional encryption at rest；
- ignore/sensitive path policy；
- secrets scanning/redaction不能破坏 exact restore，敏感 path可选择不捕获并警告；
- retention/secure delete；
- exports默认不含 workspace blobs；
- symlink/path traversal防护；
- restore不可逃出 workspace root；
- binary/large files有明确上限。

## 19. 包归属建议

建议一个深 `@own/pi-history` 或 `@own/pi-session-history` domain：

- turn boundaries；
- cursor/branch/revert state machine；
- checkpoint/revert contracts；
-不依赖 Node fs/Git/TUI。

Adapters：

- workspace snapshot store（Node）；
- session JSONL/bundle store；
- coding-agent commands/TUI；
- team worktree adapter。

不要建独立 `pi-undo` 与 `pi-redo` 包；它们是同一 state machine。若初期只有 coding-agent消费，可先在 session domain实现，不急着发布。

## 20. 分阶段建议

1. Durable turn boundaries与session cursor，先做 conversation-only undo/redo；
2. built-in edit/write touched-path checkpoints；
3. restore conflict preflight与diff UI；
4. multi-step redo + branch commit；
5. crash-safe restore journal；
6. shell/watcher coverage与side-effect receipts；
7. `/tree`/`/edit`/`/fork`统一；
8. retention/encryption/export。

## 21. 验收场景

1. 连续 undo 3 次后 visible branch正确；
2. 连续 redo 3 次恢复原 messages/files；
3. redo不重跑model/tool；
4. undo后新 prompt形成 sibling branch；
5. old branch仍可在 tree访问；
6. active stream先安全中断；
7. file addition/delete/rename/mode/symlink恢复正确；
8. user out-of-band edit触发 conflict而不覆盖；
9. non-reversible external effect明确警告；
10. restore中途crash可reconcile；
11. staged undo在process restart后恢复；
12. checkpoint GC不删除pinned redo path；
13. compaction前后的turn可undo；
14. editor undo与session undo keybindings不冲突；
15. 非Git目录可使用conversation undo，snapshot engine可用时恢复文件；
16. snapshot缺失不会虚假声称文件已恢复。

## 22. 资料来源

- OpenCode TUI undo/redo 文档：https://opencode.ai/docs/tui/#undo
- OpenCode app command：https://github.com/anomalyco/opencode/blob/dev/packages/app/src/pages/session/use-session-commands.tsx
- OpenCode core revert：https://github.com/anomalyco/opencode/blob/dev/packages/core/src/session/revert.ts
- 当前 Pi session format：`packages/coding-agent/docs/session-format.md`
- 当前 Pi tree/fork/edit：`packages/coding-agent/docs/session.md`
- Reliability研究：`../13-reliability-and-unattended/research.md`
