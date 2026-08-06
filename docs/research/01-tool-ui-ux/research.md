# 工具调用 TUI UI/UX 研究

- 状态：方向建议
- 更新日期：2026-07-31
- 范围：工具调用在交互式 TUI 中的展示、展开、错误、重试、权限与多任务状态
- 非目标：本文件不决定主题色，也不直接修改现有组件
- 核心结论：在现有 `ToolExecutionComponent` 前建立跨工具共享的 `ToolPresentation` 与 `ToolRunState`；默认使用一至三行可扫读记录，复杂、失败、等待输入或并行调用再升级为可展开卡片，TUI/Web/RPC 共用语义而不共用 ANSI 渲染

## 1. 问题与目标

工具调用是 agent harness 的主要可观察执行面。用户需要在不阅读原始 JSON 的情况下快速回答：

1. agent 正在做什么；
2. 作用对象是什么；
3. 是否仍在运行、等待输入、重试或已失败；
4. 做了多少工作、产生了什么变化；
5. 是否需要用户批准或介入；
6. 如何展开原始参数、完整输出和诊断信息。

目标不是把每次工具调用做成一张很大的卡片，而是建立“默认可扫读、按需可诊断”的分层信息密度。

## 2. 当前 Pi 基线

当前仓库已经具备可复用基础：

- `ToolExecutionComponent` 支持 pending、success、error 背景状态、局部流式更新、折叠/展开、自定义 `renderCall`/`renderResult`、自绘 shell、图片输出和通用 fallback。
- 内置工具与 extension 工具可复用同一个 `ToolDefinition` 渲染入口。
- 工具结果可以通过 `details` 携带结构化数据，而不必全部塞进文本。
- `Ctrl+O` 已是全局工具详情折叠/展开操作。
- alternate-screen 规划已经定义滚动 transcript 与固定 editor/status dock，这为稳定展示长期运行状态提供了基础。

相关实现：

- `packages/coding-agent/src/modes/interactive/components/tool-execution.ts`
- `packages/coding-agent/src/core/extensions/types.ts`
- `packages/coding-agent/src/core/tools/`
- `packages/tui/src/TuiAltScreen.ts`
- `tui-plan.md`

主要缺口不是“能不能自定义渲染”，而是缺少跨工具一致的状态语义、信息层级、交互规范与并行任务总览。

## 3. 参考产品与可借鉴部分

### 3.1 OMP

OMP 把工具调用呈现为状态卡片，并强调：

- 简短的调用摘要；
- 展开后显示完整参数和结果；
- subagent 卡片显示进度、成本、耗时；
- staged edit 明确区分 proposed 与 accepted；
- `ask` 使用结构化选项而不是把交互混在普通输出中。

可借鉴的是“每类工具有领域摘要，但共享生命周期与视觉语法”，不是复制其颜色或符号。

### 3.2 OpenCode

OpenCode 提供全局 details 开关。这说明“默认简洁 + 全局切换详情”是用户已经理解的交互模式。我们可以保留 Pi 的 `Ctrl+O`，同时增加单卡展开和失败自动展开。

### 3.3 MCP

MCP 2026-07-28 工具规范要求客户端清楚展示暴露给模型的工具、调用状态与用户确认；长任务扩展还定义了 `working`、`input_required`、`completed`、`failed`、`cancelled`。这些状态应成为 UI 模型的一部分，而不只服务 MCP 工具。

## 4. 建议的统一状态模型

不要让每个 renderer 自己发明状态。建议在 TUI 层之前归一化为：

```ts
type ToolRunState =
  | { kind: "preparing" }
  | { kind: "awaiting_approval"; risk: "read" | "write" | "execute" | "external" }
  | { kind: "running"; startedAt: number; progress?: ToolProgress }
  | { kind: "input_required"; requestId: string }
  | { kind: "retry_wait"; attempt: number; nextAttemptAt: number; reason: string }
  | { kind: "succeeded"; finishedAt: number }
  | { kind: "failed"; finishedAt: number; recoverable: boolean }
  | { kind: "cancelled"; finishedAt: number }
  | { kind: "interrupted"; recoveryAction?: string };
```

其中 `ToolProgress` 应支持：

- 不确定进度：spinner + elapsed；
- 可确定进度：current/total；
- 阶段进度：phase label；
- 子任务：running/succeeded/failed counts；
- 速率或资源量：仅在工具确实提供时显示，不能伪造百分比。

## 5. 信息层级

### 5.1 折叠态：一至三行

折叠态只回答“动作、对象、状态、关键结果”：

```text
✓ Read  packages/agent/src/harness/agent-harness.ts · 506 lines
⋯ Bash  npm run check · 18s
↻ Model openai/gpt-x → anthropic/claude-y · retry 2 in 4s
! Edit  src/auth.ts · stale anchor · action required
```

规则：

- 动词优先，参数只保留主对象；
- path、query、command 使用语义截断，不能从中间丢掉关键后缀；
- 成功结果显示数量、变更规模或命中数；
- 错误显示原因类别，不在折叠态倾倒 stack trace；
- 颜色不是唯一状态编码，必须有符号与文本；
- 进行中必须显示 elapsed，避免用户误以为卡死。

### 5.2 展开态

建议固定顺序：

1. Summary；
2. Inputs；
3. Live output / progress；
4. Result；
5. Diagnostics；
6. Artifacts / links；
7. Usage（仅对 eval/subagent/model-backed tools）。

原始 JSON 属于 diagnostics 的最后一级，不应成为默认展示。

### 5.3 失败态

失败卡自动展开到可行动信息：

- 错误类别与原始错误；
- 已执行到的阶段；
- 是否有副作用；
- retry 次数与下一步；
- 完整输出 artifact 链接；
- `Retry`、`Open output`、`Copy diagnostic` 等动作。

## 6. 不同工具族的摘要规范

| 工具族 | 折叠态必须显示 | 展开态重点 |
|---|---|---|
| 文件读取 | path、selector、行数/类型 | anchor、截断、来源 |
| 搜索 | query、scope、匹配/文件数 | 分组结果、分页 |
| 编辑 | path、变更规模、proposal/applied | diff、冲突、验证 |
| shell | command、cwd、elapsed/exit | stdout/stderr、PTY、artifact |
| eval | language、cell title/status | notebook cells、结构化输出、kernel |
| subagent | name、任务、running counts | transcript tail、usage、result schema |
| MCP | server/tool、trust scope | server identity、schema、input request |
| ask/approval | 问题/风险与推荐项 | 选项、影响、超时 |
| long-running job | job name、state、elapsed | logs、readiness、restart policy |

## 7. 交互设计

建议按“全局、焦点、指针”三层：

- 全局：`Ctrl+O` 切换全部工具详情；
- 焦点：上下选择工具卡，Enter 展开当前卡；
- 指针：在 alt-screen 中点击标题展开，滚轮只滚动 pointer 下方区域；
- 失败：自动展开，但用户可以重新折叠；
- 流式输出：展开卡片内部滚动时保持位置；折叠时不抢 transcript follow-end；
- 复制：复制摘要、复制完整诊断、复制 artifact URI 分开；
- 链接：统一使用 OSC 8，可点击 `artifact://`、`agent://`、文件位置与 URL。

不建议把每张卡都做成永久边框盒子。高频工具会产生视觉噪音；短调用可使用一行“活动记录”，只有复杂、失败、需要输入的调用升级为卡片。

## 8. 并行与子代理展示

并行调用需要一张 group summary，而不是 N 张卡无结构堆叠：

```text
Task · 3 agents    1 running · 1 done · 1 failed    42s
  ✓ ScoutAPI       18s
  ⋯ ScoutUI        42s
  ! Reviewer       schema validation failed
```

展开 group 后才能进入成员 transcript。group 与 member 都要有稳定 ID，恢复 session 后保持对应关系。

## 9. renderer 架构建议

保留现有 `ToolDefinition.renderCall/renderResult`，但在其前面增加稳定的 presentation model：

```ts
interface ToolPresentation {
  identity: { runId: string; toolName: string; groupId?: string };
  summary: { verb: string; target?: string; meta?: string };
  state: ToolRunState;
  sections: ToolSection[];
  actions: ToolAction[];
  artifacts: ToolArtifactRef[];
}
```

renderer 应优先产出结构化 presentation；只有需要特殊绘制的工具才提供自定义 Component。这样可让 TUI、RPC UI、browser board 和 session export 共享语义，而不是复用 ANSI 字符串。

## 10. 无障碍与兼容性

- 状态必须同时有文字、符号和颜色；
- 提供 ASCII symbol preset；
- 终端宽度小于 60 列时隐藏次要 metadata，不能横向截断主错误；
- 所有鼠标动作必须有键盘等价路径；
- 不依赖动画表达唯一状态；
- 尊重 reduced motion，spinner 可降级为稳定 elapsed 文本；
- CJK、宽字符和组合字符宽度必须继续走 `visibleWidth`/ANSI-aware utilities；
- main-screen 保持线性记录，复杂焦点交互只在 alt-screen 启用。

## 11. 验收场景

原型必须至少演示：

1. 100ms 成功 read 不造成大卡片闪烁；
2. 30 秒 shell 持续输出且 elapsed 更新；
3. tool 等待用户批准；
4. transient error 倒计时重试；
5. partial output 后失败，明确保留已产生输出；
6. 三个 subagent 并行、其中一个失败；
7. MCP tool 进入 `input_required`；
8. 窄终端、ASCII、无颜色模式；
9. session resume 后卡片状态与 artifact 链接仍正确；
10. main-screen final document 仍可读。

## 12. 包归属建议

- 状态与 presentation 类型：未来可放在深模块 `@own/pi-tool-presentation`，但只有当 TUI、RPC/browser 或 client 至少两个 adapter 同时消费时才值得独立成包。
- 终端 Component：保留在 `packages/tui`。
- coding-agent 到 presentation 的 adapter：保留在 `packages/coding-agent`。
- 不建议为每一种工具卡建立 npm package。

## 13. 建议研究顺序

1. 定义状态与 presentation contract；
2. 用 read、bash、task、ask 四类工具做静态 fixture；
3. 在 40/80/140 列终端做视觉原型；
4. 加入 keyboard/mouse 行为；
5. 接入 retry、MCP task 与 session resume；
6. 用录制的 event fixtures 做稳定渲染测试。

## 14. 资料来源

- 当前 Pi：`packages/coding-agent/src/modes/interactive/components/tool-execution.ts`
- 当前 Pi：`packages/coding-agent/docs/tui.md`
- 当前 Pi：`packages/coding-agent/docs/keybindings.md`
- 当前 TUI 规划：`tui-plan.md`
- OMP README：https://github.com/can1357/oh-my-pi
- OpenCode TUI：https://opencode.ai/docs/tui/
- MCP Tools：https://modelcontextprotocol.io/specification/2026-07-28/server/tools
- MCP Tasks：https://modelcontextprotocol.io/extensions/tasks/overview
