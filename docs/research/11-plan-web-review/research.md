# `/plan` 与网页评审工作流研究

- 状态：产品与架构方向建议
- 更新日期：2026-07-31
- 范围：plan mode、versioned plan artifact、Plannotator 式 browser review、anchored comments、approval/revision/execution handoff
- 核心结论：`/plan` 不是一个 prompt 模板，而是一种权限受限、可评审、可恢复的 session mode；网页是同一 plan domain 的 review adapter

## 1. 用户期望

期望流程：

1. 在 TUI 输入 `/plan`；
2. agent 调研并形成计划；
3. 自动或显式打开网页评审；
4. 用户在具体 section/line/block 上评论、修改、批准或要求重做；
5. feedback 回到 agent；
6. agent 修订 plan；
7. 获得批准后执行；
8. 执行过程与计划/acceptance 保持可追踪关系。

如果只把 plan 写到临时 Markdown 并打开 browser，无法可靠处理版本、comments、approval authority、session resume、多人/agent team 与执行偏离。

## 2. 当前 Pi 基线

当前 `plan-mode` extension example 已展示：

- `/plan` 切换 read-only tools；
- system prompt 注入 planning instructions；
- status indicator；
- exits plan mode 后把 plan 加回 prompt；
- `Ctrl+Alt+P`/`/plan` 等 toggle；
- read-only exploration 与 implementation mode 分离。

缺口：

- mode state 未形成正式 durable domain；
- 无 plan artifact/version；
- 无 structured acceptance/dependencies/risks；
- 无 browser review/comments；
- approval 不是 capability transition receipt；
- agent team/kanban 无共享 plan contract。

## 3. Plannotator 参考

Plannotator 的公开能力：

- intercept coding agent plan；
- browser 中渲染 plan；
- 对具体文本进行评论；
- approve 或 request changes；
- 支持编辑 plan；
- 本地运行；
- Pi extension、Claude Code hooks、OpenCode plugin 等 adapter；
- plan 可转 Obsidian/保存。

值得借鉴的是低摩擦网页评论与多 harness adapter。我们的设计应让 browser 通过 local protocol API 操作 plan domain，而不是让插件自己拥有第二套真相。

## 4. Plan Domain

```ts
interface PlanArtifact {
  id: PlanId;
  sessionId: SessionId;
  version: number;
  parentVersion?: number;
  status: "draft" | "in_review" | "changes_requested" | "approved" | "superseded" | "executing" | "completed" | "abandoned";
  objective: string;
  scope: PlanScope;
  assumptions: PlanAssumption[];
  steps: PlanStep[];
  acceptance: AcceptanceCriterion[];
  risks: PlanRisk[];
  decisionsNeeded: PlanDecision[];
  verification: VerificationPlan[];
  markdown: string;
  digest: string;
  createdBy: ActorRef;
  createdAt: string;
}
```

Markdown 是人类可读 projection；structured plan 是 runtime/kanban/team 可消费 contract。不要从任意 Markdown 每次反向猜 tasks/dependencies。

## 5. Plan Mode 状态机

```text
normal
  → planning.research
  → planning.drafting
  → planning.review
  → planning.revising
  → planning.approved
  → execution
```

控制事件：

- `/plan`：进入 planning；
- `submit_review`：冻结 draft version，进入 in_review；
- comment/request changes：进入 changes_requested；
- revise：生成新 version，旧版 immutable；
- approve：批准确切 digest/version；
- `/execute-plan`：只对 approved plan 进行 capability transition；
- abandon：明确结束，不误当完成。

## 6. 权限模型

Plan mode 不只是 prompt instruction。runtime 必须 enforcement：

- 默认允许 read/search/lsp/read-only shell；
- 禁止 workspace write/edit、git mutation、package install、external side effects；
- 允许写 plan/session-local review artifacts；
- command 分类由 permission policy，不通过字符串黑名单；
- user 可批准单次 research command，但不能让模型自行退出 plan mode；
- agent/subagent 同样继承 plan-mode ceiling；
- approved plan 只授权进入正常 execution policy，不自动批准每一个危险工具。

## 7. `/plan` Command UX

建议：

```text
/plan                     enter plan mode
/plan review              open/reopen browser review
/plan show                open TUI plan view
/plan versions            list versions
/plan abandon             abandon active plan
/execute-plan [version]   execute approved plan
```

当 agent 完成 draft：

```text
Plan v3 ready · 7 steps · 2 risks · 1 decision
[Review in TUI] [Open browser] [Ask agent to revise]
```

无 GUI/remote SSH 时 browser open 失败不应阻塞；输出 localhost URL/SSH port forwarding hint，并保留 TUI review fallback。

## 8. Browser Review Architecture

### Local review service

- bind loopback only by default；
- ephemeral random port；
- unguessable single-session capability token；
- token 不出现在 logs/model/session export；
- WebSocket/SSE 推 plan/comments/status；
- REST/typed RPC 提交 comment/edit/approval；
- CSRF/origin check；
- idle timeout 与 explicit stop；
- browser assets bundled，不依赖 cloud。

Browser 不直接读写 session files/SQLite。它调用 plan service；service append durable plan events。

### Remote/headless

支持：

- `--no-open` 输出 URL；
- 可配置 bind 但需 auth/TLS warning；
- SSH tunnel instructions；
- future server-hosted authenticated review；
- TUI 完整 fallback。

不要默认 `0.0.0.0`。

## 9. Anchored Comments

line number anchor 在 plan revision 后容易漂移。建议：

```ts
interface PlanAnchor {
  version: number;
  blockId: string;
  quote: string;
  startOffset?: number;
  endOffset?: number;
  contextBefore?: string;
  contextAfter?: string;
}
```

Plan structured blocks 有 stable block ID。评论主要 anchor block + quote；revision reanchor 算法：

1. 同 block ID；
2. exact quote；
3. context/fuzzy；
4. 无法匹配标 orphan，要求人工处理。

评论 thread 状态：open/resolved/outdated；resolution 需要 actor 与 note，不能 revision 后自动消失。

## 10. 编辑与版本

用户可：

- inline edit Markdown；
- structured edit step/acceptance/risk；
- add/delete/reorder step；
- leave comment without direct edit；
- approve with optional note；
- request changes。

每次提交产生新 immutable version 或 review event；不能 silent mutate approved plan。approval 绑定 digest。执行前发现版本已变则必须重新批准。

## 11. Feedback Loop

review feedback 通过 typed handoff 回 agent：

```ts
interface PlanReviewFeedback {
  planId: string;
  version: number;
  verdict: "changes_requested";
  threads: Array<{
    id: string;
    anchor: PlanAnchor;
    body: string;
  }>;
  directEdits: PlanPatch[];
}
```

只把 open/relevant threads 摘要注入 agent；完整 review 可通过 `plan://`/`local://` 读取。agent 修订时必须逐 thread 记录 resolution/reply，不能仅产一份新 plan 假装反馈已处理。

## 12. 从 Plan 到执行任务

批准 plan 后可 materialize tasks：

- 每个 `PlanStep` 可映射一或多个 task；
- dependencies 保留；
- acceptance/verification 附到 task；
- task 保存 source plan version/digest；
- plan revision 在 execution 开始后不自动改 active tasks；需要 change request/replan；
- Kanban 是 task projection，不是 plan store。

## 13. 执行偏离与 Change Request

执行中发现新信息：

1. worker/lead 创建 change request；
2. 说明 blocked assumption、影响 steps/acceptance；
3. minor change 可由 policy owner批准并生成 plan amendment；
4. major scope/architecture change 回 planning.review；
5. pending approval 时高风险 task blocked；
6. 所有 tasks/receipts 链接实际执行 plan/amendment。

不要让 plan 变成执行开始后无人维护的装饰文档。

## 14. Agent Team 集成

- lead 拥有 active plan；
- teammate 可提交 plan section/proposal；
- reviewer role 可 comment/request changes，但 human-only decision 不能冒充用户；
- high-risk teammate 可先提交 task-local plan 并等待 approval；
- team task ledger 从 approved version materialize；
- peer feedback 与 user review thread 来源分开。

## 15. TUI Plan View

除了 browser，TUI 需要：

- structured outline；
- version/status；
- open comment count；
- accept/reject/revise actions；
- keyboard navigation；
- selected step detail/acceptance/risk；
- open browser action；
- execution progress overlay；
- narrow-terminal sequential fallback。

首版 TUI 可以 read-only + approve/request changes；复杂 inline comments 由 browser 提供，但不能让 browser 成为唯一批准路径。

## 17. 安全

- plan body/comment 是 untrusted text；
- Markdown/HTML sanitize，CSP 禁 inline script；
- browser service loopback + random token + origin/CSRF；
- `file://`/OSC links 遵循 path policy；
- approve action需要当前 plan/version/digest，防 stale click；
- model/teammate 不能调用 human approval endpoint；
- user identity/actor source明确；
- direct edit 不能偷偷改变 permission policy；
- exported plan 去 session secrets/absolute auth refs。

## 18. 包归属建议

建议 `@own/pi-plan` 深模块：

- plan/version/review/approval/task materialization contracts；
- 不依赖 TUI/browser/process；
- coding-agent adapter 实现 `/plan` mode 与 tool permission；
- `@own/pi-plan-web` 或 server route 是 browser adapter；
- TUI adapter 在 coding-agent/tui；
- team/kanban 消费 plan contract。

只有 browser adapter足够独立且有第二 host 消费时才单独发布 package；首期可与 plan package workspace-internal。

## 19. 分阶段建议

1. Durable PlanArtifact/version/status；
2. runtime-enforced plan mode；
3. TUI review + approval；
4. loopback browser viewer；
5. anchored comments/direct edit；
6. revision feedback loop；
7. task materialization/kanban/team；
8. execution change request。

## 20. 验收场景

1. `/plan` 后 write tool 被 runtime deny；
2. draft version 可 resume；
3. browser unavailable 时 TUI 可批准；
4. stale plan tab 无法批准新 version；
5. comments 在 revision 后正确 reanchor/orphan；
6. request changes 后 agent 逐 thread回应；
7. approval 绑定 digest；
8. execute materializes dependencies/acceptance；
9. execution偏离产生 change request；
10. model/teammate 无法伪造 human approval；
11. loopback service 无 token拒绝请求；
12. browser close不丢 review state。

## 21. 资料来源

- 当前 Pi plan-mode example：`packages/coding-agent/examples/extensions/plan-mode/README.md`
- Plannotator：https://github.com/backnotprop/plannotator
- Plannotator Pi extension：https://github.com/backnotprop/plannotator/tree/main/apps/pi-extension
- Claude Code Agent Teams plan approval：https://code.claude.com/docs/en/agent-teams
- Agent Team 研究：`../05-agent-teams/research.md`
