# 多策略 Context Compaction 研究

- 状态：架构方向建议
- 更新日期：2026-07-31
- 范围：普通阈值摘要、连续 compartment compaction、选择性 pruning、recall 与长期 memory 的边界
- 核心结论：保留两条正式策略——可靠的 `classic-summary` 与高级的 `continuous-compartments`；两者共享同一 context ledger、budget 和 recovery contract，但同一 session 只能有一个 primary context manager

## 1. 先区分四个概念

1. **Raw history**：不可丢失的 session journal/message/tool events。
2. **Prompt view**：本轮真正发送给 model 的派生 context。
3. **Compaction**：把旧 prompt segments 替换成更小 representation。
4. **Long-term memory**：跨 session 保存的决策、约束、惯例等知识。

Compaction 不应删除 raw history；memory 也不能代替 session chronology。设计失败常来自把四者混成一个 summary 字段。

## 2. 当前 Pi 基线

当前 Pi 已支持经典 compaction：

- context 接近 model limit 时自动触发；
- `reserveTokens` 为下一轮和输出保留空间；
- `keepRecentTokens` 保留近期消息；
- summary 覆盖较旧 branch history；
- compaction entry 保存 `summary`、`firstKeptEntryId`、`tokensBefore` 与 details；
- extension 可提供 custom compaction；
- branch navigation 有 branch summary；
- `/compact` 可手动触发并附加指令；
- session JSONL 保留原始 tree entries，compaction 是追加的派生 entry。

这是很好的 fallback 基线。主要缺点：compaction 是显式停顿；一次大 summary 容易过度压缩；旧信息只能靠手动读 raw session；cache 与多轮连续整理不是核心模型。

相关资料：

- `packages/coding-agent/docs/compaction.md`
- `packages/coding-agent/docs/session-format.md`
- `packages/coding-agent/src/core/compaction/`

## 3. Magic Context 可借鉴的机制

Magic Context 的公开设计包括：

- 后台 historian 把旧 raw history 压成分层 chronological compartments；
- deterministic decay rendering 根据年龄/重要度选择 compartment fidelity；
- agent 可用 `ctx_reduce` 标记 stale tool outputs，实际删除延迟到 cache-safe boundary；
- cache-stable prompt layout；
- `ctx_search` 跨 memory/raw conversation/git 等检索；
- `ctx_expand` 将 compacted range 恢复为原始 transcript；
- historian 同一 pass 提取长期 project memories；
- optional dreamer 做 verify/curate/classify 等离线维护；
- temporal markers 保留时间感；
- 内置 compaction 必须禁用以避免 double-compress。

应研究并重建这些原理，而不是复制其数据库、命令或“无限 context”宣传。任何有限 model window 都只能获得一个有损 prompt view；真正的保证是 raw history 可检索、压缩可追溯、恢复不中断。

## 4. 统一 Context Ledger

建议把 prompt 组成建模为 typed segments：

```ts
type ContextSegment =
  | SystemSegment
  | RuleSegment
  | MemorySegment
  | RawTurnSegment
  | ToolOutputSegment
  | CompartmentSegment
  | RetrievalSegment
  | ActiveTaskSegment;

interface ContextSegmentMeta {
  id: string;
  sourceRef: string;
  tokenEstimate: number;
  createdAt: string;
  importance: number;
  cacheClass: "stable" | "session-stable" | "turn-volatile";
  retention: "required" | "recent" | "summarizable" | "droppable";
  sensitivity?: string;
}
```

Prompt assembly 只消费 ledger view，不直接遍历 session JSONL 拼字符串。所有策略都输出一个 `ContextPlan`：

```ts
interface ContextPlan {
  strategy: string;
  modelWindow: number;
  inputBudget: number;
  outputReserve: number;
  segments: RenderedContextSegment[];
  omitted: OmissionReceipt[];
  totalTokens: number;
}
```

每轮可用 `/context explain` 查看“为什么这段被保留/压缩/省略”。

## 5. Token Budget

不要只用单一 percentage threshold：

```text
model context window
- provider/system overhead
- expected output reserve
- tool schema budget
- safety margin
= input history budget
```

需要 model-aware 预算：

- model 最大 context 与最大 output；
- provider 对 image/tool schema/cache 的计算差异；
- 当前 loaded tool catalog；
- next action 可能产生的大 tool result；
- agent/team/eval nested context policy。

Token count 优先使用 provider tokenizer；无 tokenizer 时 conservative estimate，并在 telemetry 比较 actual usage 修正误差。

## 6. 策略 A：Classic Summary

这是默认可靠路径和 emergency fallback。

### 触发

```ts
trigger = estimatedNextInput > inputBudget * threshold
```

建议设置：

- threshold（例如 0.80–0.90，可按 model profile）；
- reserve tokens；
- keep recent tokens/turns；
- summary model role；
- max summary tokens；
- manual only/automatic。

### 算法

1. 选择从当前 branch root/上一 compaction 到 cut point 的完整 semantic units；
2. tool call + result 不能拆开；
3. 保留最后 N recent tokens；
4. 生成 structured summary；
5. 校验 summary 包含 active goal、decisions、constraints、modified files、verification、open blockers、artifact refs；
6. append compaction entry；
7. 下一 turn 使用 summary + recent raw history；
8. failure 时不破坏 current context，按 emergency pruning 再试或提示。

### Summary schema

建议内部结构化后再 render：

```ts
interface ClassicSummary {
  objective: string;
  completed: string[];
  decisions: string[];
  constraints: string[];
  artifacts: string[];
  codeState: Array<{ path: string; state: string }>;
  verification: string[];
  openWork: string[];
  unresolved: string[];
  chronology: string[];
}
```

这样比一段自由 prose 更易检查与迁移。

## 7. 策略 B：Continuous Compartments

### 7.1 Compartment

```ts
interface HistoryCompartment {
  id: string;
  sessionId: string;
  range: { fromEvent: string; toEvent: string };
  rawDigest: string;
  generation: number;
  summaryTiers: {
    headline: string;
    compact: string;
    detailed?: string;
  };
  importance: number;
  topics: string[];
  entities: string[];
  createdAt: string;
  historian: ModelReceipt;
}
```

范围必须 contiguous、不可重叠（同 generation）、可验证 digest。修改 summary 产生新 generation，不原地覆盖历史。

### 7.2 后台 historian

- 在 raw history 达到最小 batch 后异步生成；
- 不阻塞 primary model；
- 只处理已经 durable 的 closed range；
- 使用便宜/本地 model 可以，但输出必须 schema validate；
- 失败只延迟 compartment，不影响主 session；
- primary context 临近上限而 historian 落后时，立即执行 classic fallback；
- historian 的输入/output/usage 与 model version 有 receipt。

### 7.3 Deterministic rendering

不让 LLM 每轮决定历史保留程度。根据：

- age；
- importance；
- current query relevance；
- active task linkage；
- available budget；
- recency floor；

选择 detailed/compact/headline/omitted。相同 ledger、query fingerprint、settings 与 model budget应产生相同计划，保护 cache 与可调试性。

### 7.4 Context layout

建议稳定顺序：

1. system/rules；
2. durable project memories；
3. oldest→newest compartments；
4. recent raw history；
5. retrieval additions；
6. active task/turn input。

只在 cache-safe boundary materialize newly completed compartments。后台 historian 完成不能在 provider stream 或 cache TTL 中间立即改 prefix。

## 8. Selective Reduction

长 tool outputs 是最适合先减的 context：

- raw artifact 已 durable；
- inline output 可替换为摘要 + `artifact://` ref；
- agent 可标 stale，但不能直接物理删除 session event；
- required errors、unresolved decisions、current diff/verification 不能只因年龄被丢；
- drop queued 到 cache-safe boundary；
- operation 生成 omission receipt。

建议工具：

```text
context_reduce(ids/ranges, reason)
context_keep(ids/ranges, reason)
```

model-visible tag 只用于 UI/agent reference；内部仍以 stable segment/event ID 为真相。

## 9. Emergency Recovery

即使高级策略正常，也需要 emergency path：

1. 立即移除可重建的大 tool previews，只保留 artifact refs；
2. 缩短 loaded optional tool schemas；
3. 使用最新 valid compartments；
4. 同步 classic summarize 尚未覆盖的 oldest range；
5. 若仍超限，换 larger-context fallback model（仅设置允许）；
6. 最后暂停并给出诊断，不循环压缩同一内容。

Provider 返回 context overflow 时最多进行一次重新估算/compaction retry；持续 overflow 是 estimator/config error，不能无限 retry。

## 10. Recall 与 Expand

Advanced compaction 只有配合 recall 才可靠：

- `context_search(query, scope)` 搜 raw session、compartments、memories、git index；
- `context_expand(range/id)` 返回 raw transcript 的精确片段；
- retrieval 内容是 turn-volatile segment，有 token budget与引用；
- model 必须能区分 raw quote、historian summary、inferred memory；
- retrieval 不应永久 pin，除非 agent/user 明确 promotion。

搜索第一版可用 SQLite FTS/BM25；embedding 是可选 adapter。不能因 embedding provider不可用而失去 raw history搜索。

## 11. Memory 提取

Historian 可顺手提出 memory candidates，但写入长期 memory 应独立：

```text
candidate → validated → active → stale/archived
```

- categories：architecture、constraint、config、convention、decision、user preference；
- project identity 不只绑定 cwd；
- file-backed memory 记录证据 path/digest；
- 高风险 config/secrets 不自动保存；
- correction 应产生 supersede 关系；
- 后台 verify/curate 是后续能力，不是 compaction 正确性的前提。

## 12. Strategy Ownership 与冲突

同一 session 只能有一个 `primaryContextManager`：

- `classic-summary`；
- `continuous-compartments`；
- third-party extension。

启动时检测：

- 多个 manager；
- built-in auto compaction + extension compaction；
- duplicate pruning hooks；
- incompatible session format/version。

冲突时 fail safe：禁用高级 manager，保留 classic emergency，并显示 actionable diagnostics。绝不能 double summarize 同一 range。

## 14. TUI/Settings

Status 显示：

```text
Context 61% · raw 18k · compacted 42k→8k · tools 4k · reserve 12k
```

Settings：

- Strategy：Classic / Continuous / Manual；
- Trigger threshold；
- output reserve；
- recent raw budget；
- historian model/fallback；
- agent-driven reduction；
- cache-safe delay；
- memory/retrieval/embedding；
- emergency fallback。

Diagnostics：

- prompt segment breakdown；
- current compartments/ranges；
- pending reduction；
- estimated vs provider usage；
- cache bust reason；
- historian lag/error；
- conflicting manager。

## 15. 质量评测

不能只测“token 变少”。需要 replay corpus，评测：

- active task continuation；
- constraint retention；
- exact value/path recall；
- chronology；
- unresolved blocker；
- tool result provenance；
- hallucinated completion；
- prompt cache hit rate；
- main model latency/cost；
- historian cost/lag；
- context overflow recovery。

每个 compaction 算法变更对相同 session corpus 做 A/B，judge + deterministic assertions 结合。

## 16. 包归属建议

建议深模块边界：

- `@own/pi-context-core`：ledger、budget、strategy contract、context plan；
- `@own/pi-context-classic`：现有 summary adapter；
- `@own/pi-context-continuous`：compartment/historian/reduction；
- `@own/pi-memory`：只有跨 session memory 稳定后再分离；
- SQLite/embedding/provider/TUI 是 adapters。

首期可在 `packages/agent` 实现 core contract，把现有 coding-agent compaction 包成 adapter；不要先重写现有可靠路径。

## 17. 分阶段建议

1. Context ledger + explain，不改行为；
2. classic strategy 迁移到统一 contract；
3. tool output artifact pruning；
4. durable compartments + background historian；
5. deterministic decay renderer；
6. search/expand；
7. memory capture/maintenance；
8. cache-aware tuning 与 eval corpus。

## 18. 验收场景

1. classic threshold trigger 可预测；
2. summary failure 不损坏 session；
3. tool call/result 不被 cut point 拆开；
4. continuous historian 落后时 fallback；
5. background completion 不随机破坏 cached prefix；
6. raw range 可 exact expand；
7. dropped output 仍可从 artifact 找回；
8. session resume 恢复 strategy/compartment state；
9. 两个 managers 冲突时 fail safe；
10. model context overflow 不进入 infinite retry；
11. 旧 decision 可经 search 找回且有 provenance；
12. 删除 index 后可从 journal重建。

## 19. 资料来源

- 当前 Pi compaction：`packages/coding-agent/docs/compaction.md`
- 当前 session format：`packages/coding-agent/docs/session-format.md`
- Magic Context README：https://github.com/cortexkit/magic-context
- Magic Context configuration：https://github.com/cortexkit/magic-context/blob/master/CONFIGURATION.md
