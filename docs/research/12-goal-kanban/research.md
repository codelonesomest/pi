# GoalBuddy + Kanban 执行反馈研究

- 状态：产品与领域架构方向建议
- 更新日期：2026-07-31
- 范围：Goal intake、任务图、Kanban projection、agent/team execution、comments、blocked/failed feedback 与状态回流
- 核心结论：Goal/Task Ledger 是 durable source of truth；Kanban 是人机协作视图和命令面，不是另一套 task database

## 1. 产品目标

用户希望把 GoalBuddy 的严谨执行方法与 Cline Kanban 的可视化任务管理结合：

1. 用户定义 broad goal；
2. 系统通过 intake/research/judgement 形成可执行目标；
3. 任务按依赖与阶段进入 Kanban；
4. agents/teams 执行并实时更新；
5. 用户能在 card、blocker、failure、artifact、plan 上评论；
6. feedback 可把 failed/blocked/review task 拉回 todo/ready；
7. task completion 有 proof，而不是 agent 自报；
8. session/process 重启后继续执行。

## 2. GoalBuddy 可借鉴的执行纪律

GoalBuddy 强调：

- broad/vague work 先结构化 intake；
- durable board 与 receipts；
- Scout/Judge/Worker role separation；
- 一次只有一个 active task（在需要严格串行的 goal 模式）；
- Worker 是有界、可逆、allowed-files、带 verify 的 work package；
- Judge 在歧义、风险、phase transition 与 completion 做 skeptical gate；
- PM/lead 拥有 rolling board，而不是 worker 自己决定下一个 scope。

这能避免“看板有很多卡，但 agent 任意抓取并悄悄缩 scope”。不过不是所有 goal 都应强制单活跃任务；Agent Team 模式可允许多个互不依赖、ownership 清楚的 ready tasks 并行。

## 3. Cline Kanban 可借鉴的能力

Cline Kanban 是 agent orchestration workspace，公开说明包含：

- projects、task board、task details；
- 多 coding agents/CLI integrations；
- Git worktree isolation；
- real-time terminal streaming；
- changes/diff review；
- MCP/API integrations；
- task status 与 agent execution；
- comments/mentions 等协作方向。

值得借鉴的是 visual workspace + worktree/terminal/diff 聚合。我们的实现不能假设每张 card 等于一个独立 terminal process，也不能让 UI 直接用 Git/PTY 状态推测任务真相。

## 4. 三层领域模型

### 4.1 Goal

```ts
interface Goal {
  id: GoalId;
  title: string;
  outcome: string;
  context: string;
  constraints: Constraint[];
  acceptance: AcceptanceCriterion[];
  nonGoals: string[];
  risks: Risk[];
  status: "intake" | "research" | "planned" | "executing" | "blocked" | "review" | "completed" | "abandoned";
  activePlan?: PlanRef;
  policy: GoalExecutionPolicy;
}
```

### 4.2 Task

沿用 Agent Team Task Ledger 的 task contract：dependency、assignee/lease、allowed files、verification、artifacts、receipt、feedback。

### 4.3 Board

```ts
interface BoardView {
  id: string;
  goalId: GoalId;
  columns: BoardColumn[];
  filters: BoardFilter[];
  sort: BoardSort;
  wipPolicy: WipPolicy;
}
```

Board column 由 task status 投影；拖卡产生 domain command，例如 `request_transition`，不能直接把 database status 改成任意字符串。

## 5. 建议状态机

```text
backlog
  → ready
  → claimed
  → in_progress
  → verification
  → review
  → done
```

异常路径：

```text
claimed/in_progress → blocked
claimed/in_progress/verification/review → failed
any open → cancelled
blocked/failed/review_changes → ready (new generation)
```

UI 列建议：

- Backlog；
- Ready；
- In Progress；
- Verification；
- Review；
- Blocked/Failed（可分 swimlane）；
- Done。

`failed` 不等于 blocked：failed 有已发生的失败 attempt/receipt；blocked 是缺少前置条件或人工输入。

## 6. Transition Guards

| Transition | 必要条件 |
|---|---|
| backlog→ready | goal/plan scope明确、dependencies满足 |
| ready→claimed | atomic lease、assignee capacity |
| claimed→in_progress | session/worktree/runtime ready |
| in_progress→verification | deliverables reported、artifacts durable |
| verification→review | verify contract通过 |
| review→done | required judge/human approvals |
| blocked→ready | blocker resolved + feedback/action recorded |
| failed→ready | retry decision、generation增加、必要时策略修改 |
| done→ready | explicit reopen reason；保留原 receipt |

拖拽不满足 guard 时，UI 应显示原因与可执行动作，而不是静默弹回。

## 7. Intake 与 Goal Prep

Goal 新建不应立刻拆成大量 implementation cards。流程：

1. Capture：原始目标；
2. Clarify：只询问工具/代码无法回答且有实质 tradeoff 的问题；
3. Scout：建立 repo/system evidence；
4. Judge：检查 scope、acceptance、风险与可执行性；
5. Plan：生成 plan artifact；
6. Approve：用户或 policy gate；
7. Materialize tasks；
8. Execute。

Board 在 intake/research 阶段可显示“Discovery” lane，避免假装实施任务已经 ready。

## 8. 单活跃与并行模式

### Focused Goal 模式

- WIP limit = 1；
- 适合未知依赖强、风险高、长链工作；
- 最早 open task自动 active；
- phase gate 由 Judge/PM；
- worker 完成后才能推进。

### Team Parallel 模式

- 每个 independent lane 有 WIP；
- task 必须声明 dependency、ownership、cross-task contract；
- shared prerequisite 先 inline/lead 完成；
- 并行宽度受 team capacity/budget；
- same files 的 task不因 UI方便就并行。

模式是 goal policy，不是看板全局固定。

## 9. Card 设计

折叠 card 显示：

- title + task ID；
- status、priority；
- assignee/agent role；
- dependency/blocker count；
- elapsed/age；
- comments/open feedback；
- verify/review badges；
- worktree/diff summary；
- retry generation。

详情页 tabs：

1. Objective & acceptance；
2. Activity/event timeline；
3. Agent transcript；
4. Changes/diff；
5. Verification；
6. Comments & feedback；
7. Artifacts；
8. Dependencies；
9. Attempts/retries。

不要在卡片默认显示长 agent prose。

## 10. Comments 与 Feedback

```ts
interface FeedbackThread {
  id: string;
  subject: GoalRef | TaskRef | ArtifactAnchor | DiffAnchor | PlanAnchor | FailureRef;
  status: "open" | "resolved" | "outdated";
  messages: FeedbackMessage[];
  requestedAction?: "clarify" | "revise" | "retry" | "reassign" | "replan" | "cancel";
}
```

来源必须区分：user、lead、judge、worker、system。评论不会直接成为模型 authority；domain service 根据 actor permission 转成 command。

### Blocked feedback flow

1. worker 创建 structured blocker；
2. task→blocked；
3. user/peer comment；
4. blocker owner 确认 resolved 或提供新的 action；
5. task generation++；
6. 回 ready；
7. 新 assignee/run 获取 blocker + feedback + previous artifact refs。

### Failed feedback flow

1. failure 保存 error class、attempt、partial side effects、logs；
2. UI 提供 Retry same、Retry with feedback、Reassign、Replan、Cancel；
3. feedback 不修改旧 attempt；
4. 新 attempt 链接 parent attempt；
5. 已产生副作用需 reconciliation 后才 retry。

## 11. Completion Proof

Task receipt：

```ts
interface TaskReceipt {
  taskId: TaskId;
  generation: number;
  outcome: string;
  changedArtifacts: string[];
  verification: VerificationReceipt[];
  unresolved: string[];
  worker: ActorRef;
  model?: ModelReceipt;
  startedAt: string;
  finishedAt: string;
}
```

`done` 必须满足 acceptance oracle。不同任务可要求：

- command/test result；
- browser interaction proof；
- diff review；
- judge verdict；
- human approval；
- external deployment observation。

Worker “我完成了”不是 proof。

## 12. Worker/Judge/Scout Role 集成

- Scout：read-only evidence receipt；不能改 board scope；
- Worker：只执行一个 coherent task generation；
- Judge：read-only gate，可以 approve/request changes/mark ambiguity；
- Lead/PM：拥有 goal/plan/task creation、priority 与 phase transition；
- user：最终 scope、high-risk decision 与 override authority。

角色由 unified agent definition 加载；权限 ceiling 由 runtime enforcement。

## 13. 实时事件与 UI 同步

```ts
interface BoardSnapshot {
  generation: number;
  goal: Goal;
  tasks: TeamTask[];
  members: TeamMember[];
}
```

之后推 ephemeral progress/events；client generation gap 时请求新 snapshot。`packages/protocol` 的 authoritative snapshot + transient progress 原则适合复用。

UI 不能以 WebSocket message 到达顺序当作 durable final order。server append domain event 后才发布 authoritative change。

## 14. TUI + Browser 分工

### TUI

- 当前 goal/task摘要；
- list/column compact view；
- comments quick reply；
- block/retry/reassign；
- jump transcript；
- pause/resume；
- keyboard-first。

### Browser

- 大型 Kanban；
- dependencies graph；
- diff/plan/artifact anchored review；
- multi-agent timeline；
- drag/drop guarded transitions；
- filters/analytics。

二者共享 API 与 domain commands。TUI 不是 degraded truth；无 browser 时仍能完成关键控制。

## 15. Session 与存储

建议：

```text
durable records/
  goals/<goal-id>/events.jsonl
  tasks/<task-id>/events.jsonl
  reviews/
  artifacts/
  indexes/session.sqlite
```

或统一 session journal + projections。重要的是：

- append-only events 为真相；
- SQLite board views可重建；
- task/member session refs稳定；
- comments/diff anchors durable；
- process/job handles只在 runtime registry；
- board resume 不依赖浏览器仍开着。

## 16. Automation 与 24h 运行

Board scheduler 可：

- 自动 claim ready tasks；
- 遵守 dependency/WIP/budget；
- crashed run reconcile；
- recoverable failure按 policy retry；
- blocked/input_required 通知；
- no-progress detector；
- idle team scale-down；
- nightly/long tasks durable resume。

但不能自动越过 human decision、permission approval 或 unresolved side-effect ambiguity。

## 17. 安全

- browser local service auth/CSRF 与 plan review相同；
- comments/agent output是 untrusted text；
- diff/HTML sanitize；
- user identity/actor source不可由 agent伪造；
- drag/drop/retry commands需要 optimistic generation；
- team permission ceiling；
- worktree/filesystem boundary；
- secrets不进 card/transcript/export；
- external integrations（GitHub/Jira）单独 permission/auth。

## 18. 包归属建议

建议未来：

- `@own/pi-goals`：goal/intake/acceptance/phase policy；
- `@own/pi-work`：task ledger/status/receipt/feedback/lease；
- `@own/pi-team`：member/mailbox/scheduler；
- `@own/pi-board-web`：browser projection/adapters；
- plan contract独立。

但首期避免一次建四个空包。可先在 `packages/agent/src/work/` 建 goal+task domain，等 team/browser 成为第二消费者后按真实依赖提取。Kanban rendering 不应成为 core package dependency。

## 19. 分阶段建议

1. Goal/Task/Receipt/Feedback domain + event store；
2. TUI task list与 guarded transitions；
3. Goal intake/Scout/Judge/Worker workflow；
4. browser Kanban read-only；
5. comments/retry/reopen/diff review；
6. Agent Team scheduler/worktrees；
7. plan materialization/change request；
8. long-running automation与external integrations。

## 20. 验收场景

1. dependency 未完成不能进 ready；
2. focused mode 只允许一个 active task；
3. parallel mode只调度 independent ownership；
4. blocked card feedback→new generation→ready；
5. failed attempt保留并可带 feedback retry；
6. verify fail不能 done；
7. reviewer request changes回 work queue；
8. browser/TUI 同步 generation；
9. lead/server restart后 board重建；
10. agent不能伪造 user comment/approval；
11. worktree conflict产生 integration task；
12. done task reopen保留旧 receipt/audit。

## 21. 资料来源

- Goal Prep skill：`skill://goal-prep`
- Cline Kanban：https://github.com/cline/kanban
- Agent Team 研究：`../05-agent-teams/research.md`
- Plan review 研究：`../11-plan-web-review/research.md`
- Protocol package：`packages/protocol/README.md`
- Durable harness：`packages/agent/docs/durable-harness.md`
