# Agent Team 协作机制研究

- 状态：架构方向建议
- 更新日期：2026-07-31
- 范围：多个长期 agent session 的任务协调、点对点通信、人工介入、共享任务与恢复
- 核心结论：Agent Team 必须是可恢复的协调系统，而不是“并行调用多个 subagent”的 UI 包装

## 1. Agent Team 与 Subagent 的本质区别

| 维度 | Subagent | Agent Team teammate |
|---|---|---|
| 生命周期 | 一次委派，完成后返回 | 长期 session，可多轮接收任务 |
| 通信 | 通常只向 caller 返回 | 可与 lead、peer、用户直接通信 |
| 协调 | parent 集中调度 | 共享 task ledger + 分布式 claim |
| 上下文 | 独立，但目标单一 | 独立且持续积累团队工作上下文 |
| 用户介入 | 主要查看最终输出 | 可进入任一 teammate transcript 干预 |
| 持久化 | run receipt 即可 | roster、mailbox、lease、tasks、session 都需持久化 |
| 适用 | 有界调查、实现、审查 | 需要持续并行与跨角色协作的复杂目标 |

不要让模型根据名称模糊选择两者。runtime 应有两个明确 primitive：`delegate()` 与 `team.spawn()`。

## 2. 当前 Pi 基线

当前仓库已有可组合基础，但没有正式 team runtime：

- subagent example 支持 isolated context、parallel/chain、streaming、abort、usage；
- session tree 与 durable session 适合记录 child run/operation；
- `packages/protocol` 定义 authoritative snapshots 与 transient progress 的区分；
- `packages/server`/`client` 提供远程 session 演进方向；
- TUI alternate-screen layout 能支持固定 team panel；
- durable harness 设计覆盖 queue、operation、turn、unfinished tool reconciliation。

缺少的是共享 task ledger、peer mailbox、member lifecycle、ownership/lease、团队级恢复、直接进入 member session 的 UI 和可验证完成协议。

## 3. Claude Code 参考

Claude Code Agent Teams 提供：

- lead + independent teammates；
- shared task list；
- peer-to-peer mailbox；
- in-process panel 或 tmux/iTerm2 split panes；
- 用户可以进入 teammate transcript、直接发消息或中断；
- task dependency、assignment/self-claim 和 file locking；
- plan approval；
- `TeammateIdle`、`TaskCreated`、`TaskCompleted` hooks；
- teammate role 可复用 subagent definition。

其公开文档也明确标注已知限制：resume、task coordination 和 shutdown 仍有实验性质。可借鉴 interaction model，但我们应把 durable recovery 和 task completion proof 作为首发 contract，而不是后补。

## 4. OMP 参考

OMP 的 `task`、`hub`、`todo` 已体现更通用的协调 primitive：

- agent roster 与 stable name；
- peer messaging；
- shared background job/process supervisor；
- async result delivery；
- task 并行与结构化 output；
- phase-aware todo。

可借鉴的是“协调功能是独立工具 surface”，但团队持久化不应只依赖当前进程内 registry。

## 5. 建议领域模型

```ts
interface AgentTeam {
  id: TeamId;
  goal: GoalRef;
  lead: MemberId;
  members: TeamMember[];
  taskLedger: TaskLedgerRef;
  eventStream: TeamEventStreamRef;
  policy: TeamPolicy;
}

interface TeamMember {
  id: MemberId;
  name: string;
  role: AgentDefinitionRef;
  session: SessionRef;
  state: "starting" | "working" | "idle" | "blocked" | "failed" | "stopping" | "stopped";
  currentTask?: TaskId;
  heartbeatAt?: string;
}
```

必须区分：

- **desired state**：lead 想让 member 做什么；
- **observed state**：member 当前实际状态；
- **transient progress**：TUI 展示；
- **durable event**：恢复与审计真相。

## 6. Task Ledger

建议 task card 至少包含：

```ts
interface TeamTask {
  id: TaskId;
  title: string;
  objective: string;
  status: "queued" | "ready" | "claimed" | "working" | "blocked" | "review" | "done" | "failed" | "cancelled";
  dependencies: TaskId[];
  assignee?: MemberId;
  lease?: { owner: MemberId; generation: number; expiresAt: string };
  allowedFiles?: string[];
  verification: VerificationContract[];
  artifacts: string[];
  receipt?: TaskReceipt;
  feedback: FeedbackThreadRef;
}
```

关键 invariant：

- unresolved dependency 的 task 不能进入 ready；
- claim 必须是原子 compare-and-swap；
- assignee crash 后 lease 到期才能重新 claim；
- `done` 必须有 receipt；
- review rejected 会回到 ready/working，并增加 generation；
- cancelled 与 failed 是不同语义；
- 人工 feedback 不能丢在聊天文本中，必须有 anchored durable record。

## 7. Ownership 与文件冲突

“每个 task 一个 worktree”能减少写冲突，但不能替代 ownership contract。

建议组合：

1. task 可声明 `allowedFiles`/subsystem ownership；
2. 默认写任务使用 isolated worktree；
3. shared checkout 只允许 read-only 或明确 single-writer task；
4. peer 发现需要越界时先消息协调并更新 task contract；
5. merge/apply 由 lead/integration role 处理；
6. same-file 并行只在用户明确选择竞争实现时允许。

worktree lifecycle：创建、base commit、gitignored dependency 策略、apply/merge、conflict、cleanup、orphan recovery 都必须记录为 durable operation。

## 8. Messaging

### 8.1 消息类型

```ts
type TeamMessage =
  | { kind: "note"; body: string }
  | { kind: "question"; body: string; responseContract?: JsonSchema }
  | { kind: "answer"; replyTo: MessageId; body: unknown }
  | { kind: "handoff"; taskId: TaskId; artifacts: string[] }
  | { kind: "blocker"; taskId: TaskId; body: string }
  | { kind: "review"; taskId: TaskId; verdict: "approve" | "request_changes"; findings: Finding[] }
  | { kind: "control"; action: "interrupt" | "pause" | "resume" | "stop" };
```

所有消息需要 ID、sender、recipient、createdAt、delivery state、replyTo。不要把 control message 与普通 prompt 混为一谈。

### 8.2 Delivery

- durable enqueue 后 send API 才返回 accepted；
- at-least-once delivery，receiver 按 message ID 去重；
- 自动 push，不要求 lead polling；
- `await reply` 是高层 convenience，不改变 mailbox durability；
- broadcast 展开为每 recipient 一条 delivery，便于确认；
- 大 payload 经 `local://`/`artifact://`，消息只传 ref。

## 9. Lead 的职责

Lead 是 policy owner，不应成为所有数据的中转：

- 分解/激活 task；
- 设置跨 task contract；
- 决定 spawn/stop；
- 处理冲突与阶段门；
- 汇总用户可见结果；
- 保证全局 oracle/验收达成。

peer 可以直接分享发现和协调 ownership。lead 不应把 peer message 全量复制到自己的 prompt；只注入摘要、blocker、decision 与 relevant artifact ref。

## 10. Human-in-the-loop

用户需要：

- 查看团队总览；
- 进入任一 member transcript；
- 直接发送 steering/follow-up；
- interrupt/pause/stop；
- 给 task 或具体 artifact/comment anchor 写 feedback；
- 更改 priority/assignee/dependency；
- 将 blocked/failed 拉回 ready；
- 要求 plan approval 或 review gate。

agent-to-agent 消息必须标记来源，不能冒充用户授权。teammate 不能代替用户批准敏感 tool invocation。

## 11. Plan Approval 与 Review Gate

高风险 worker 可先进入 read-only planning：

1. member 产出 versioned plan artifact；
2. lead 或 human review；
3. rejection 写 feedback 并保持 read-only；
4. approval 生成 capability transition event；
5. implementation 绑定 approved plan digest；
6. task completion 时验证实际 diff 是否超出批准范围。

reviewer 必须是独立角色，不能让执行者仅靠自报完成关闭 task。

## 12. TUI 与 Browser UI

### TUI 固定 panel

```text
Team · 4 members · 2 working · 1 blocked
  lead          coordinating
  api-worker    T104 · 3m
  ui-worker     T105 · 2m
  reviewer      blocked · waiting for T104
```

- Up/Down 选择 member；
- Enter 打开 transcript；
- Esc interrupt 当前 member；
- `t` 切 task list；
- failed/blocker 永不自动隐藏；
- idle member 可折叠但仍 addressable。

### Browser board

适合任务图、diff/review、comments、多个 member 的总览。TUI 与 browser 都消费同一 team snapshot/event API，不能各自维护状态。

## 13. 恢复

进程重启后：

1. 加载 team、member、task、mailbox events；
2. 验证每个 member session；
3. 未完成 provider stream 标记 interrupted；
4. unfinished tool 按 idempotency policy reconcile；
5. expired lease 进入 recoverable；
6. worktree 扫描并验证 base/head；
7. mailbox 重投未 ack 消息；
8. lead 生成恢复摘要；
9. 只有 safe task 自动 resume。

不能声称从 provider stream 中间续传；只能从 durable turn/tool/task boundary 恢复。

## 14. 成本与容量控制

Team policy 应有：

- max members；
- max concurrent turns；
- per-member model/effort；
- total token/cost budget；
- idle timeout；
- retry budget；
- no-progress detector；
- task recursion/delegation depth。

达到 budget 进入 paused/blocked，而不是静默切低质量模型或无限重试。

## 15. 安全与信任

- team role definition 沿用 agent definition trust；
- project task instructions/peer messages 是 untrusted；
- permission ceiling 从 lead/session policy 向下传递；
- member 间不能转发“用户已同意”来绕过批准；
- mailbox 内容 schema validate，坏消息隔离而不阻塞其余消息；
- browser/TUI 控制 API 需要本地 auth/CSRF protection；
- worktree 不自动共享 secrets，按明确 allowlist 注入。

## 16. 包归属建议

建议未来形成 `@own/pi-team` 深模块，条件是 session/agent definition contract 已稳定：

- team domain、ledger、mailbox、lease、reducer；
- 不依赖 TUI、git CLI 或具体 model provider；
- `@own/pi-team-node` 或 coding-agent adapter 处理 process/worktree；
- TUI/browser/server 各自是 adapter；
- durable records provide event/blob storage.

在 contract 未稳定前可先放 `packages/agent/src/team/`，避免过早发布公共 API。

## 17. 验收场景

1. 两个 member 同时 claim 一个 task，只有一个成功；
2. member crash 后 lease 到期并恢复；
3. peer message at-least-once 但只执行一次；
4. blocked task 收到用户 feedback 后回 ready；
5. plan rejection→revision→approval→execution；
6. worker 完成但 reviewer 驳回；
7. worktree merge conflict 进入明确 integration task；
8. lead crash 后 team 可重建；
9. 用户可进入 member、发消息、返回总览；
10. 预算耗尽暂停且保留可恢复状态。

## 18. 资料来源

- 当前 subagent example：`packages/coding-agent/examples/extensions/subagent/README.md`
- durable harness：`packages/agent/docs/durable-harness.md`
- protocol snapshots：`packages/protocol/README.md`
- Claude Code Agent Teams：https://code.claude.com/docs/en/agent-teams
- Claude Code Subagents：https://code.claude.com/docs/en/sub-agents
- OMP README（task/hub/todo）：https://github.com/can1357/oh-my-pi
- Cline Kanban：https://github.com/cline/kanban
