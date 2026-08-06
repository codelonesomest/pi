# 重试、故障恢复与长期无人值守运行研究

- 状态：可靠性架构方向建议
- 更新日期：2026-07-31
- 范围：provider request/stream、agent turn、tool、MCP、subagent/team、process/session 的 retry、fallback、watchdog、resume 与 24 小时运行
- 核心结论：不能“不管什么 error 都自动 retry”；必须按 operation durability、错误类别、是否已产生副作用和幂等性决定 retry/reconcile/block/failover。24 小时运行依赖 durable supervisor，不是更大的 retry loop

## 1. 用户关心的具体问题

- 请求在 provider 建立连接前失败，能否自动 retry？
- 已经收到部分 stream 后失败，为什么有时不能 retry？
- provider/model 故障时，能否按 fallback chain 切换？
- tool、subagent、MCP、网络或进程崩溃后如何处理？
- 主进程重启后，能否发现 session 最终是 error 并自动 resume？
- 如何让任务连续运行 24 小时，而不是因 unknown error 永久停下？

这些问题跨越至少五层。用一个 `maxRetries` 不能正确回答。

## 2. 当前 Pi 基线

当前仓库已经有三类重试/恢复能力：

### 2.1 Provider request retry

`packages/ai/src/utils/provider-retry.ts`：

- 处理建立/获取 provider response 时的 408、409、429、5xx、连接错误及 `x-should-retry`；
- 读取 `retry-after-ms` / `retry-after`；
- exponential backoff + jitter；
- sleep 可被 AbortSignal 中断；
- 默认 `maxRetries` 为 0，由 caller/settings 决定；
- provider 请求延迟默认最多接受 60 秒，可配置。

这层通常发生在 stream 成功建立之前；不能假设所有 SDK/transport 都能在 stream 中途从 byte offset 恢复。

### 2.2 Agent turn auto-retry

`packages/coding-agent/src/core/agent-session.ts`：

- 识别 overloaded、rate limit、server errors；
- context overflow 走 compaction，不混入普通 retry；
- failed assistant message 先持久化到 session，retry 时从 live agent state 移除；
- exponential backoff、UI events、可取消；
- 成功后重置 attempt；
- compaction/branch summary 也可用 retry policy；
- context overflow 只有一次 compact-and-retry guard，避免循环。

### 2.3 Durable recovery 设计

`packages/agent/docs/durable-harness.md` 已明确：

- session 是 durable append-only state；
- provider stream 不能从中间 resume；
- 恢复从 durable boundary 重新开始；
- unfinished turn/provider request 默认 mark interrupted；
- unfinished tool 只有声明 retry-safe/idempotent 才自动重试；
- queue consumption、pending write、operation/turn/tool start/finish 需要 durable entries。

这已经给出了正确原则，但尚需成为完整 supervisor/product contract。

## 3. 为什么“任何 error 自动 retry”危险

### 可安全重试

- DNS/connection reset before request accepted；
- provider 429/503 且无外部副作用；
- read-only filesystem/search；
- deterministic local computation；
- remote task 查询；
- content summary request。

### 需要 reconcile 后再决定

- stream 中途断开；
- tool request 已发送但 result 丢失；
- MCP remote task 可能仍运行；
- GitHub issue/create PR/payment/deploy 等外部 mutation；
- shell command 部分执行；
- file edit 落盘但 completion event 未写；
- worktree merge/apply 中断。

### 不应自动重试

- auth invalid/permission denied；
- malformed request/schema；
- unsupported model/tool；
- billing/quota exhausted；
- safety refusal/policy denial；
- deterministic compiler/test failure；
- user cancellation；
- prompt/tool definition bug；
- unknown side-effect state 且无幂等键/查询能力。

无限 retry 会烧钱、重复副作用、放大 outage、锁死 session，并让真正的代码/config 错误难以发现。

## 4. 统一 Operation Model

所有长或可失败行为应有 durable operation identity：

```ts
interface DurableOperation {
  id: OperationId;
  kind: "provider_request" | "agent_turn" | "tool_call" | "mcp_task" | "subagent" | "team_task" | "compaction" | "process";
  state: "scheduled" | "running" | "waiting_retry" | "input_required" | "succeeded" | "failed" | "cancelled" | "interrupted" | "uncertain";
  attempt: number;
  generation: number;
  idempotency: "pure" | "idempotent" | "queryable" | "non_idempotent" | "unknown";
  sideEffects: "none" | "possible" | "confirmed" | "unknown";
  requestDigest: string;
  startedAt?: string;
  heartbeatAt?: string;
  resultRef?: string;
  error?: ClassifiedFailure;
}
```

状态转移 append 到 session journal。公开 API 只有在 operation accepted/durable 后返回。

## 5. 错误分类

不要靠 error message regex 贯穿系统。Provider adapter 可在边界将 SDK/HTTP error normalize：

```ts
type FailureClass =
  | "network"
  | "timeout"
  | "rate_limit"
  | "overloaded"
  | "server"
  | "auth"
  | "permission"
  | "quota"
  | "invalid_request"
  | "context_overflow"
  | "content_policy"
  | "protocol"
  | "tool_error"
  | "process_exit"
  | "resource_exhausted"
  | "cancelled"
  | "unknown";

interface ClassifiedFailure {
  class: FailureClass;
  transient: boolean;
  retryAfterMs?: number;
  providerCode?: string;
  message: string;
  diagnosticsRef?: string;
}
```

regex 只能留在 legacy adapter；domain policy 消费 structured class。

## 6. Retry Policy

```ts
interface RetryPolicy {
  enabled: boolean;
  maxAttempts: number;
  maxElapsedMs: number;
  backoff: "fixed" | "exponential" | "decorrelated-jitter";
  baseDelayMs: number;
  maxDelayMs: number;
  respectRetryAfter: boolean;
  retryClasses: FailureClass[];
  circuitBreaker?: CircuitBreakerPolicy;
}
```

规则：

- attempt 包含初次执行，UI/设置不要混用 maxRetries/maxAttempts；
- server `Retry-After` 受用户最大等待与 operation deadline 限制；
- jitter 防止并行 agents 同时重试；
- 总 elapsed/budget 与 attempt count 双上限；
- user cancel 立即终止 sleep/current attempt；
- retry policy 可以 per operation/provider/tool override，但 higher-level ceiling wins；
- retry schedule durable，进程重启后可恢复 remaining delay，而不是从零重置。

## 7. Stream 中途失败

### 7.1 不能真正续传的原因

多数 provider stream 没有通用的可恢复 cursor。中途断线后：

- 已生成 token 可能计费；
- provider 内部状态/response ID 未必支持 continue；
- tool-call JSON 可能只收到一半；
- 重发同一请求会生成不同输出；
- partial assistant text 不能作为可信 completed turn。

因此正确术语是“重跑 assistant turn”，不是“resume stream”。

### 7.2 安全策略

Provider stream 阶段尚未执行 tool side effect 时：

1. 持久化 failed attempt 与 partial output artifact；
2. live context 排除 incomplete assistant message；
3. 从上一 durable user/tool-result boundary 重发；
4. 新 attempt 有独立 ID，并链接 previous attempt；
5. UI 可显示 previous partial output，但不把它当模型历史；
6. usage/cost 两次都记录；
7. 超过 policy 后 fail/fallback。

如果 provider 支持 server-side response retrieval/continuation，可由专属 adapter 声明 capability；不能把 provider-specific 能力假装成统一保证。

### 7.3 Tool call 边界

只有完整且 schema-valid 的 assistant tool call message durable 后才执行 tool。partial JSON 不能执行。若 tool 已经开始，后续恢复由 tool operation policy 决定，不能简单重跑整个 turn。

## 8. Tool Retry 与幂等性

每个 tool definition 增加：

```ts
interface ToolReliability {
  idempotency: "pure" | "idempotent" | "queryable" | "non_idempotent" | "unknown";
  retryPolicy?: RetryPolicyRef;
  reconcile?: (operation: InterruptedToolOperation) => Promise<ReconcileResult>;
  idempotencyKey?: "supported" | "required" | "none";
  compensation?: CompensationDescriptor;
}
```

示例：

- read/grep/glob：pure，可 retry；
- hash/format/check：通常 idempotent；
- edit with stable anchors：query target state 后可判断 applied/not applied；
- write entire file：需要 digest/atomic write 才可 idempotent；
- shell：默认 unknown，不自动 retry；
- create issue/payment/deploy：需要 external idempotency key 或 query/reconcile；
- ask：input_required，不是 failure；
- long process：由 process supervisor 重新 attach/query，而非重启命令。

## 9. Model/Provider Fallback

Fallback 不是 retry 的同义词。建议 ordered routing policy：

```ts
interface ModelFallbackPolicy {
  candidates: ModelSelector[];
  on: FailureClass[];
  preserveProvider?: boolean;
  maxSwitches: number;
  contextCompatibility: "require_fit" | "compact_if_needed" | "skip";
  toolCompatibility: "require_all" | "reduce_optional" | "skip";
  outputCompatibility: "require" | "best_effort";
}
```

切换前验证：

- context fit；
- image/tool/structured output support；
- reasoning/model role；
- auth/quota；
- loaded tool schema compatibility；
- privacy/data residency policy；
- cost ceiling。

### 建议流程

1. same endpoint transport retries；
2. same model endpoint/credential failover（若配置）；
3. same provider compatible model；
4. cross-provider candidate；
5. compact/reduce tools if policy 允许；
6. block/fail with diagnostics。

不要遇到 invalid request 就换十个模型；同一 payload bug 会反复失败。

### Conversation portability

跨 provider transform 必须：

- 排除 failed/aborted assistant messages；
- 处理 provider-specific reasoning/signatures；
- validate tool call/result pairing；
- preserve system/user/tool semantics；
- 记录 transformation warnings；
- 若不能安全转换则 skip candidate。

## 10. Circuit Breaker 与 Outage

多 agent 24h 运行时，单独 exponential retry 会产生 thundering herd。每 provider/endpoint/account 需要 shared circuit breaker：

```text
closed → open → half_open → closed/open
```

- failure rate/连续 transient error 触发 open；
- open 时 operation 等待或使用 fallback，不实际请求；
- half-open 只允许少量 probe；
- auth/quota 可进入更长 open/blocked；
- server Retry-After 影响 reopen；
- breaker state 有 telemetry，但敏感 credential 不入 session；
- 用户可手动 reset/test。

## 11. Watchdog 与 Durable Supervisor

24h 运行需要独立 supervisor loop：

```ts
interface Supervisor {
  recover(): Promise<RecoveryReport>;
  schedule(operation: OperationRequest): Promise<OperationRef>;
  cancel(id: OperationId): Promise<void>;
  pause(scope: ScopeRef): Promise<void>;
  resume(scope: ScopeRef): Promise<void>;
  reconcile(): Promise<void>;
}
```

watchdog 检测：

- heartbeat 超时；
- provider stream 无数据/无 progress；
- child process 退出；
- subagent/teammate session 终止；
- retry timer 到期；
- task lease 过期；
- MCP remote task 可查询；
- session 有 unfinished operation；
- no-progress loop（反复同样错误/tool）；
- budget/credential/disk/resource alarm。

Supervisor 只能在明确 policy 下采取动作：retry、fallback、restart child、requeue task、mark uncertain、block 并通知。它不能把所有 terminal error message 转成“继续”。

## 12. Session Final Error 自动 Resume

不要通过“最后一条 assistant stopReason=error”单独判断。最后 error 可能已经被用户看见、取消或不再是 active operation。

恢复判定应 reduce durable journal：

```ts
interface RecoveryCandidate {
  operationId: string;
  terminalStateMissing: boolean;
  lastDurableBoundary: string;
  failure?: ClassifiedFailure;
  sideEffectState: string;
  policyDecision: "resume" | "retry" | "reconcile" | "block" | "ignore";
}
```

Startup watcher：

1. 找 operation_started 但无 terminal event；
2. 找 waiting_retry timer；
3. 找 retryable failed goal/task 且 policy 允许；
4. 验证 runtime dependencies/model/tool versions；
5. reconcile tool/MCP/process side effects；
6. append recovery decision；
7. safe operation 重新入队；
8. uncertain/blocker 通知用户。

已明确 terminal `failed` 的普通交互 turn 不默认自动续写；只有 goal/automation policy 声明 `keep_running` 才创建新的 recovery attempt。

## 13. Unattended Mode

```yaml
reliability:
  mode: supervised
  unattended:
    enabled: true
    max-runtime: 24h
    max-cost: 20 USD
    max-turns: 500
    max-no-progress-cycles: 3
    on-input-required: notify-and-block
    on-permission: block
    on-unknown-side-effect: block
```

可配置行为：

- notification channels；
- quiet hours；
- safe auto approvals（默认无）；
- fallback models；
- disk/log retention；
- checkpoint interval；
- shutdown deadline；
- maintenance/restart window。

硬边界：

- 不自动回答需要用户选择的问题；
- 不越过权限；
- 不重复 unknown/non-idempotent side effect；
- 不无限消耗 token/cost；
- 不把 test/compiler failure 当 transient；
- 不在没有 acceptance proof 时把 goal 标 done。

## 14. No-progress 检测

长运行常见失败不是异常，而是 agent 循环：

- 相同 tool + args 反复失败；
- 相同 error fingerprint；
- 相同 diff 被反复改回；
- compaction/fallback 循环；
- reviewer 与 worker 无新证据来回驳回；
- output 无实质进展。

记录 fingerprint：

```text
(operation kind, normalized error, tool/model, target, state digest)
```

连续 N 次无 state change：

1. 停止自动 retry；
2. 尝试一次不同策略/diagnostic agent（policy 允许）；
3. 仍无进展→blocked；
4. 保存 concise blocker、attempts、logs、推荐动作。

## 15. Process 与 Service Supervision

长运行进程使用 stable launch spec，而不是 PID：

- application/argv/cwd/env refs；
- readiness（log/port/health）；
- restart policy：no/on-failure/always；
- bounded backoff；
- process-tree termination；
- stdout/stderr artifact；
- heartbeat/last output；
- crash count；
- persistent/detached 语义；
- restart 后重新发现/launch，不信任旧 PID。

watcher、MCP stdio、browser review、team workers 都可复用 process supervisor adapter。

## 16. UI/UX

Retry 状态必须清楚：

```text
↻ Provider overloaded · retry 2/4 in 8s   [Retry now] [Switch model] [Cancel]
```

详情：

- error class/raw diagnostic；
- current attempt/elapsed；
- backoff/Retry-After；
- selected fallback chain；
- partial output artifact；
- cost so far；
- side-effect certainty；
- recovery decision。

长期运行 dashboard：

- goal/task/member/process 状态；
- last progress；
- retry/fallback/circuit breaker；
- budget；
- blocked/input required；
- next scheduled action；
- notification status。

不要用 spinner 掩盖 hours-long blocked state。

## 17. Settings

分层设置：

- Provider request retry；
- Agent turn retry；
- Summarization retry；
- Tool policies；
- Fallback chains；
- Circuit breakers；
- Recovery on startup；
- Unattended goal policy；
- Notifications；
- Budgets/limits。

UI 提供 delay sequence simulation，例如 `1s → 2s → 4s → 8s`；明确 attempt 语义与总等待上限。

## 18. Observability

每 attempt 记录：

- operation/attempt ID；
- failure class/provider code；
- scheduled/actual delay；
- Retry-After；
- model/provider route；
- request/response ID（无 secret）；
- first token/stream duration；
- partial bytes/tokens；
- usage/cost；
- side-effect/reconcile result；
- terminal decision。

关键指标：retry success rate、fallback success、duplicate-prevented、uncertain operations、circuit-open time、no-progress blocks、MTTR、unattended completion rate。

## 19. 包归属建议

建议深模块：

- `@own/pi-reliability-core`：failure taxonomy、policy、operation/recovery decisions；
- provider adapters 留在 `packages/ai`；
- durable supervisor/reducer 可在 `packages/agent` 或 `@own/pi-runtime`；
- Node process supervisor 是 node adapter；
- coding-agent 负责 settings/TUI/notifications；
- team/MCP/eval 实现其 `reconcile` adapter。

不要建立 `pi-retry`、`pi-fallback`、`pi-watchdog` 三个互相循环依赖的小包；它们是同一 reliability domain。

## 20. 分阶段建议

1. 统一 structured failure taxonomy；
2. durable operation/attempt journal；
3. current turn retry 迁移到 policy；
4. tool idempotency/reconcile metadata；
5. startup recovery reducer；
6. model fallback router + compatibility checks；
7. circuit breaker；
8. goal/team supervisor 与 unattended budgets；
9. notifications/no-progress；
10. chaos/restart verification。

## 21. 验收与故障注入

1. request 建立前 429 遵守 Retry-After；
2. stream 收到部分 text 后断线，从上一 durable boundary 重跑且 partial 保留；
3. partial tool JSON 不执行；
4. tool side effect 后 crash 不重复执行；
5. idempotency key 允许 safe retry；
6. auth/quota 不无限 retry；
7. context overflow 只 compact/retry 一次；
8. fallback model 不支持 tools 时被 skip；
9. provider outage 触发 shared circuit breaker；
10. process crash 后 supervisor 按 policy 重启；
11. host 在 waiting_retry 中重启后恢复剩余计划；
12. unfinished MCP task 重新 query；
13. unknown side effect 进入 uncertain/blocked；
14. 相同错误循环触发 no-progress；
15. cost/runtime budget 耗尽安全暂停；
16. 24h soak 中 session/process 重启仍能恢复并保留 audit。

## 22. 资料来源

- Provider retry：`packages/ai/src/utils/provider-retry.ts`
- Agent auto-retry：`packages/coding-agent/src/core/agent-session.ts`
- Durable recovery design：`packages/agent/docs/durable-harness.md`
- Agent Team 研究：`../05-agent-teams/research.md`
- MCP 研究：`../09-built-in-mcp/research.md`
- Context compaction 研究：`../10-context-compaction/research.md`
