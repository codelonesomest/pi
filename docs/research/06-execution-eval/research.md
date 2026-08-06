# 内置 Eval 执行能力研究

- 状态：架构与安全方向建议
- 更新日期：2026-07-31
- 范围：类似 OMP 的持久 Python/JavaScript cell runtime、结构化展示、工具回调、子代理编排
- 注意：本专题中的 `eval` 是 agent 可调用的代码执行工具，不是 `packages/evals` 中的行为评测套件
- 核心结论：以持久但可丢失进程状态的 `EvalRuntime` seam 统一 JavaScript/Python adapters；cell、输出、artifact、nested tool/agent operation 必须 durable，runtime state 默认不在恢复时自动重放，权限与取消继续经过 harness policy

## 1. 产品价值

普通 shell 适合运行项目命令，不适合增量数据探索与编排：

- 每次 one-shot script 都重复 import、解析和加载；
- 复杂 quoting 和临时脚本增加失败面；
- 数据帧、图像、JSON、Markdown 只能降级成 stdout；
- 很难在同一运行时内直接调用 harness 的 `read`、`task` 等工具；
- 无法自然表达 notebook 式“定义 → 检查 → 修正 → 使用”。

内置 eval 应提供持久、可观察、可取消的 cell runtime，用于：

- 数据分析与转换；
- 快速实验算法/API；
- 结构化提取；
- 并行/流水线 orchestration；
- 在 runtime 内组合已有 harness tools；
- 生成图像、JSON、Markdown artifact。

## 2. 当前仓库基线

当前仓库有 `packages/evals`，它提供的是 model-backed behavioral evaluation：

- `createPiCodingAgentHarness(...)`；
- isolated temp projects；
- comparative harness tables；
- judges、session artifacts、tokens/latency/cost。

它不提供 interactive session 中给模型使用的 Python/JS runtime。两者应共享 telemetry/artifact concepts，但不能共用同一个“eval”领域接口。

建议命名区分：

- **Execution Eval**：tool/runtime；
- **Harness Evals**：离线或 CI 行为评测。

CLI 与文档可分别叫 `eval tool` 和 `eval suite`。

## 3. OMP 参考能力

OMP 的 eval 提供：

- persistent JavaScript VM 与 Python/IPython-style kernel；
- 按 language 独立 state 与 reset；
- top-level await；
- cell title、timeout、streaming、structured details；
- `display()` 输出 JSON/image/Markdown/text；
- `read`、`write`、`env`、`output`；
- `tool.<name>(args)` 回调当前 session tools；
- `completion()` 做 stateless one-shot model call；
- `agent()`、`parallel()`、`pipeline()` 编排 subagents；
- artifact spill、取消、kernel cleanup；
- parent/subagent 可选择共享 executor state。

这是很强的参考，但完整复制会一次引入 worker protocol、Python subprocess kernel、tool bridge、artifact、concurrency、security 等多个复杂系统。建议分阶段建立深接口。

## 4. 建议对外工具 contract

第一版使用单 cell call，之后可以兼容 batch；工具参数：

```ts
interface EvalInput {
  language: "js" | "py";
  code: string;
  title?: string;
  timeout?: number;
  reset?: boolean;
}
```

返回：

```ts
interface EvalResult {
  executionId: string;
  language: "js" | "py";
  status: "completed" | "failed" | "cancelled" | "timed_out";
  outputs: EvalOutput[];
  durationMs: number;
  artifactRefs: string[];
  diagnostics: EvalDiagnostic[];
}

type EvalOutput =
  | { kind: "text"; text: string }
  | { kind: "markdown"; markdown: string }
  | { kind: "json"; value: JsonValue }
  | { kind: "image"; artifact: string; mimeType: string }
  | { kind: "table"; columns: string[]; rows: JsonValue[][] };
```

不要把所有 rich output 先 stringify，再要求 TUI 猜内容类型。

## 5. Runtime seam

```ts
interface EvalRuntime {
  readonly language: "js" | "py";
  execute(request: EvalExecutionRequest): Promise<EvalExecution>;
  reset(session: EvalSessionKey): Promise<void>;
  dispose(session: EvalSessionKey): Promise<void>;
  health(): Promise<EvalRuntimeHealth>;
}
```

host runtime 只依赖该接口。JavaScript worker 与 Python kernel 是两个 adapter。

### Session key

应由以下组合生成：

- owning agent/session ID；
- language；
- cwd identity；
- optional sharing group。

默认每个 agent 独立 kernel。共享 state 必须显式：

- `isolated`（默认）：避免 sibling race 和秘密泄漏；
- `inherit-read`：child 启动时 snapshot/seed；
- `shared`：高级 orchestration 才允许，并明确取消/reset 的破坏性影响。

## 6. JavaScript runtime

建议独立 Worker/process，不能在主 TUI event loop 直接运行任意 JS：

- top-level await 通过 async wrapper；
- persistent global scope；
- local module cache policy明确；
- stdout/stderr/display 通过 typed frames；
- AbortSignal 先 cooperative，再 terminate worker；
- runaway sync loop 只能靠 worker termination；
- reset 终止旧 worker 并新建，而不是尝试清空所有 global state。

不要默认暴露 host `process`、raw filesystem 与 unrestricted network。权限通过 capability bridge 注入。

## 7. Python runtime

建议 subprocess + framed NDJSON/CBOR protocol：

- persistent asyncio loop；
- top-level await；
- stdout/stderr/rich display frames；
- interrupt：SIGINT/Windows equivalent → grace → terminate → kill；
- reset：销毁 kernel，不尝试逆转 imported modules/global mutation；
- stdin request 明确失败或转成 `input_required`，不能永久挂起；
- interpreter selection、venv 与 cwd 是 config，不由 cell 随意更换 host runtime。

第一阶段可以只支持 Python 3 标准库；第三方 package 安装必须走普通 tool/approval，不在 eval 内静默安装。

## 8. Prelude 与 tool bridge

建议最小 prelude：

```text
display(value)
read(path, selector?)
write(path, content)
output(ref, selector?)
env(name?)
tool.call(name, args)
```

之后再加：

```text
completion(prompt, options)
agent(prompt, options)
parallel(thunks)
pipeline(items, stages)
```

### Tool bridge 安全规则

- 只暴露当前 agent capability catalog 中可见工具；
- 每次调用仍执行正常 permission/approval；
- bridge 请求带 parent eval execution ID，便于 trace；
- recursion/depth/concurrency 有硬上限；
- eval 不能通过 bridge 绕过 plan mode、sandbox 或 project trust；
- result 经过原工具的 size/artifact policy；
- tool call 事件进入 session journal，不能藏在 eval stdout 里。

## 9. `completion()` 与 `agent()`

### completion

适合轻量 map/classify/judge：

- stateless；
- 无 agent tools；
- 可指定 model role 与 JSON Schema；
- 失败直接抛 structured error；
- 使用量计入 parent eval nested usage。

### agent

复用正式 subagent runtime：

- 不能建立第二套 agent discovery；
- 可返回 structured object 或 `agent://` handle；
- background node 有 durable ID；
- `parallel`/`pipeline` 遵守 team/task concurrency budget；
- large shared context 写入 `local://`，不要重复 inline。

## 10. Timeout 与取消

区分：

- wall timeout；
- idle timeout（无 output/progress）；
- CPU budget（未来 sandbox）；
- nested tool call time；
- user cancellation。

若 nested `agent()` 正常长时间运行，不应消耗 eval cell 的“无进展”预算；但整个 operation 仍受总 wall deadline/用户取消控制。

取消一个 shared runtime 可能破坏其他 execution，因此默认禁止同 kernel 并行，或为每 execution 提供隔离 worker。

## 11. Artifact 与输出限制

- stdout/stderr 使用 ring/tail buffer；
- inline output 超过阈值 spill 到 `artifact://`；
- 大 image/table 不复制进 session JSONL；
- `display` JSON 做深度、节点数、循环引用检查；
- 每行与总 byte 有硬限制；
- binary output 只通过 artifact；
- TUI 保留每 cell summary、duration、status，并可展开 artifact。

## 12. TUI 设计

折叠：

```text
✓ Eval py · analyze CSV · 1.8s · 2 outputs
```

展开：

```text
[py] analyze CSV                          completed 1.8s
code
  df = ...
outputs
  table 10×4
  image artifact://...
```

运行中显示当前 cell、elapsed 与最近 output tail。错误自动展开 traceback 的末端，并提供 full artifact；不要倾倒数千行 stack。

## 13. Durability

kernel process 不可持久化，session 必须持久化：

- execution requested/started/finished/interrupted；
- code digest 或完整 code（按隐私设置）；
- runtime version/interpreter；
- output refs；
- nested tool/agent IDs；
- reset event。

resume 后：

- 默认 kernel state 为 lost；
- UI 明确显示“runtime state not restored”；
- 可选 notebook replay 只重跑被标记 pure/replay-safe 的 cells；
- 永不自动重跑带 tool/write/network 副作用的 cell。

未来可支持 checkpoint serializer，但不应作为首发承诺。

## 14. Sandbox 与安全

任意代码执行是高风险面：

- 默认继承当前 process 权限是不够的；
- 推荐子进程 sandbox、workspace root、network policy、resource limits；
- secrets 默认不注入 env；
- `env()` 只返回 allowlisted/脱敏值；
- filesystem 优先走 tool bridge；
- raw JS/Python runtime 是否允许 direct FS/network 由 profile 控制；
- package imports、native modules、subprocess spawn 进入 permission model；
- project-supplied code 与 skill script 保留 provenance。

第一版若没有强 sandbox，必须清楚标注“与当前用户 process 同权限”，不能制造安全错觉。

## 15. 性能与资源

- kernel lazy start；
- idle TTL + LRU cleanup；
- 每 session 每 language 最多一个默认 kernel；
- 全局 kernel/worker 上限；
- memory/CPU telemetry；
- kernel crash 不影响主 TUI；
- JS/Python adapter health check；
- cold start 与 first output latency 进入 eval suite。

## 16. 与 Harness Evals 的结合

`packages/evals` 应覆盖 execution eval contract：

- state persistence across cells；
- JS/Python isolation；
- structured JSON/image output；
- tool bridge permission ceiling；
- timeout/cancel；
- kernel crash recovery；
- artifact spill；
- prompt/model 在不同 tool descriptions 下的使用正确率。

但 execution runtime 本身不依赖 `vitest-evals`。

## 17. 分阶段建议

### Phase A

- JavaScript worker；
- persistent state、reset、text/JSON display；
- read/output/tool bridge；
- timeout/cancel/artifact。

### Phase B

- Python kernel；
- rich display；
- interpreter/config/health。

### Phase C

- completion/agent/parallel/pipeline；
- structured output；
- shared execution DAG。

### Phase D

- sandbox profiles、resource quotas；
- replay-safe notebook/session recovery；
- UI refinements。

## 18. 包归属建议

建议独立 package，但按 adapter 拆分而不是按功能碎拆：

- `@own/pi-eval-core`：types、runtime contract、frames、output model；
- `@own/pi-eval-js`：JS worker adapter；
- `@own/pi-eval-python`：Python kernel adapter；
- coding-agent：tool definition、permission bridge、TUI adapter；
- `packages/evals`：behavioral verification。

若首版只有 JS 且只有 coding-agent 消费，可先放 coding-agent 内部；Python/其他 host 出现后再提取，避免假 seam。

## 19. 验收场景

1. 两次 JS call 复用变量；
2. reset 后变量消失；
3. Python/JS state 不串；
4. infinite loop 可取消且主 TUI 继续工作；
5. structured JSON/image 正确展示；
6. 大 output spill 到 artifact；
7. tool bridge 不能绕过 deny/approval；
8. kernel crash 后本次失败、后续可新建；
9. session resume 不假装恢复内存 state；
10. nested agent usage 与 artifact 可追踪。

## 20. 资料来源

- 当前 harness evals：`packages/evals/README.md`
- 当前 eval scripts：`package.json`
- OMP eval 文档：https://github.com/can1357/oh-my-pi/blob/main/docs/tools/eval.md
- OMP README：https://github.com/can1357/oh-my-pi
- Subagent 研究：`../04-subagents-and-frontmatter/research.md`
