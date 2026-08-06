# Subagent 与统一 Frontmatter 研究

- 状态：格式与运行时方向建议
- 更新日期：2026-07-31
- 范围：可复用 agent definition、frontmatter、scope、trust、tool/model/skill/MCP/hook 与 output contract
- 核心结论：建立“可移植核心 + namespaced harness 扩展”的版本化格式；不要把所有运行时状态都塞进 YAML frontmatter

## 1. 目标

用户希望融合 Claude Code subagent 的成熟能力并建立自己的强大 agent definition 格式。它需要同时支持：

- 自动/显式委派；
- 独立 context；
- model/effort/turn budget；
- tools 与 permissions；
- skills 预载或按需使用；
- MCP server scope；
- hooks；
- background 与 worktree isolation；
- persistent memory policy；
- typed structured output；
- 可作为 agent team teammate role；
- user/project/package/managed scope；
- hot reload、diagnostics 与 trust。

## 2. 当前 Pi 基线

`packages/coding-agent/examples/extensions/subagent/` 已提供：

- 独立 `pi` process/context；
- single、parallel、chain；
- streaming、abort、usage；
- user agent 与 project agent discovery；
- markdown + YAML frontmatter；
- fields：`name`、`description`、`tools`、`model`；
- project agent trust confirmation；
- workflow prompt presets。

这是有效原型，但仍是 example extension，而不是稳定的 harness contract。它缺少 versioned schema、permission policy、output schema、hooks/skills/MCP、team role、isolation 生命周期和清楚的 override/precedence。

## 3. Claude Code 参考

Claude Code 的 custom subagent frontmatter 截至 2026-07 支持：

- required `name`、`description`；
- `tools`、`disallowedTools`；
- `model`、`effort`、`maxTurns`；
- `permissionMode`；
- `skills` preload；
- `mcpServers`；
- scoped `hooks`；
- `memory`；
- `background`；
- `isolation: worktree`；
- `color`、`initialPrompt`。

其 scope precedence 为 managed、CLI session、project、user、plugin。definition 也可复用为 agent team teammate，但部分 fields 在 team path 不适用。

值得借鉴：功能完整、scope 清楚、description 驱动自动委派。需要改进：多个字段的 team/subagent 语义不同，格式本身没有跨 harness portable core。

## 4. OpenCode 参考

OpenCode 区分 primary 与 subagent，并支持：

- JSON 或 Markdown definition；
- model、temperature/top_p/max steps；
- `permission` 的 allow/ask/deny 和 pattern；
- task permission 控制某 agent 能调用哪些 subagent；
- hidden/color；
- parent/child session navigation。

值得借鉴的是 permission-first，而不是只给 tool allowlist。

## 5. Agent Skills 标准的边界

Agent Skills 规范定义的是“可按需加载的能力包”，不是运行中的 agent identity。其 portable fields 很小：

- `name`、`description`；
- `license`、`compatibility`、`metadata`；
- experimental `allowed-tools`；
- body + scripts/references/assets progressive disclosure。

因此不能把 skill frontmatter 与 agent frontmatter 合并成同一种东西。Agent definition 可以引用/preload skills，但两者生命周期、权限与系统 prompt 角色不同。

## 6. 建议格式原则

1. **可移植核心小而稳定**：identity、description、mode、model intent、capabilities。
2. **运行时扩展 namespaced**：本 fork 特有字段放 `runtime` 或 `x-own-pi`，避免与未来标准冲突。
3. **配置与 prompt 分离**：frontmatter 是 declarative config，Markdown body 是 role prompt。
4. **引用优于复制**：复杂 permission/hook/MCP 可引用 profile。
5. **deny wins**：所有 merge 后 permission deny 优先。
6. **trust provenance 保留**：definition 从哪里加载必须进入 runtime metadata。
7. **schema version 明确**：拒绝未知 major version，不静默猜测。
8. **定义不是运行状态**：task、mailbox、turn count、PID、worktree path 不写回 definition。

## 7. 建议 frontmatter v1

```yaml
---
schema: own-pi-agent/v1
name: security-reviewer
description: Reviews authentication and authorization changes. Use after auth-related edits.
mode: subagent

model:
  role: slow
  selectors:
    - anthropic/claude-opus-5
    - openai/gpt-5.6
  effort: high
  max-turns: 12

capabilities:
  tools:
    allow: [read, grep, glob, lsp, bash]
    deny: [write, edit]
  delegates:
    allow: [scout]
    max-depth: 1

permissions:
  profile: read-only-review
  overrides:
    bash:
      "*": deny
      "git diff *": allow

skills:
  preload: [secure-design]
  discover: true

mcp:
  servers: [docs]

execution:
  background: auto
  isolation: none
  timeout: 20m
  concurrency: shared

output:
  format: structured
  schema: ./schemas/review-result.schema.json
  mode: strict

memory:
  scope: none

ui:
  color: warning
---

You are a skeptical security reviewer...
```

字段名只是提案，必须通过 fixtures 验证后冻结。

## 8. 字段语义

### 8.1 Identity

- `schema`：required；
- `name`：lowercase kebab case、全 scope stable；
- `description`：同时说明 what + when，作为 discovery text；
- `mode`：`primary | subagent | teammate | any | hidden`；`teammate` 可以由 subagent role 复用，不代表常驻实例。

### 8.2 Model

不要只接受一个 model ID：

- `role` 表达 intent（smol/default/slow/plan）；
- `selectors` 是允许的 ordered fallback；
- `effort` 是抽象等级，由 provider adapter 映射；
- `max-turns` 是 agent loop budget，不是 provider request retry。

resolve 后要把确切 provider/model/version 写入 run receipt。

### 8.3 Capabilities 与 permissions

`capabilities.tools` 决定模型看得到什么；`permissions` 决定调用后是否 allow/ask/deny。两层不能混淆。

- tool 不可见时模型不应尝试；
- tool visible 但 `ask` 时 UI 可确认；
- deny 永远胜过 allow；
- parent permission 是 ceiling，child definition 不能提升权限；
- project definition 不能通过 profile ref 绕过 trust。

### 8.4 Skills

- `preload`：把完整 SKILL.md 放入 child initial context，仅用于确实每次需要的技能；
- `discover`：只给 metadata/catalog，运行时按需加载；
- skills 自己的 `allowed-tools` 不能提升 agent permission。

### 8.5 MCP

- 只引用已经信任/configured 的 server name；
- inline server definition 只允许 managed/user scope，project scope 需 trust；
- server tool catalog 默认按需发现；
- credential 不进入 agent file。

### 8.6 Hooks

建议避免在 frontmatter 内嵌巨大 hook map。使用：

```yaml
hooks:
  profiles: [review-quality-gates]
  local:
    before-finish: ./hooks/validate-review.mjs
```

project hook 是 executable code，必须进入 trust/approval 模型。plugin/package definition 默认不得偷偷注册 host-level hooks。

### 8.7 Output

structured output 是本 fork 可以超越简单 prose handoff 的关键：

- inline JSON Schema 或相对 path；
- permissive/strict；
- max output bytes；
- artifact spill；
- validation failure 进入明确 failed 状态，不把无效 JSON 当成功。

## 9. 标准化后的内部模型

所有来源先 parse 成 source-specific AST，再 normalize：

```ts
interface AgentDefinition {
  identity: AgentIdentity;
  source: AgentDefinitionSource;
  prompt: string;
  modelPolicy: ModelPolicy;
  capabilityPolicy: CapabilityPolicy;
  permissionPolicy: PermissionPolicy;
  resourcePolicy: ResourcePolicy;
  executionPolicy: ExecutionPolicy;
  outputContract: OutputContract;
}
```

runtime 只消费 normalized model，不在执行路径中反复判断 Claude/OpenCode/Pi field 名。

## 10. Scope 与 precedence

建议从高到低：

1. managed/org；
2. one-shot CLI/session；
3. project-local（closest cwd wins）；
4. user；
5. package/plugin。

规则：

- 同 scope 同名是 error/warning，不依赖 filesystem order；
- higher scope override 整个 definition，默认不 field-merge；
- 可以显式 `extends` 一个 base definition，但必须防 cycle，并在 diagnostics 显示 resolved chain；
- project definition 只有 trust 后加载；
- hot reload 原子替换 catalog，新 run 使用新版本，正在运行的 child 保持 snapshot。

## 11. Discovery 与自动委派

catalog 常驻 context 只提供：

- name；
- description；
- mode；
- cost/permission hints；
- output type。

完整 prompt、schema、skills、MCP 仅在 spawn 时加载。自动委派排序可基于 lexical/BM25/embedding，但最终必须满足 parent delegation policy 和权限 ceiling。

需要支持显式调用：

```text
@security-reviewer review auth changes
```

以及 model tool call：

```json
{ "agent": "security-reviewer", "task": "..." }
```

## 12. Subagent run contract

每个 run 需要 stable receipt：

```ts
interface AgentRunReceipt {
  runId: string;
  definition: { name: string; digest: string; source: string };
  model: { provider: string; id: string; effort?: string };
  status: "completed" | "failed" | "cancelled" | "interrupted";
  startedAt: string;
  finishedAt?: string;
  usage?: Usage;
  output?: unknown;
  artifacts: string[];
  sessionRef: string;
  worktreeRef?: string;
}
```

parent 只接收有界 summary/structured output；完整 transcript 通过 `agent://<runId>` 读取。

## 13. Isolation

`execution.isolation` 建议：

- `none`：read-only/research；
- `worktree`：写任务；
- `auto`：runtime 根据 tool/write scope 决定；
- 未来 filesystem snapshot adapter。

需要定义：base ref、untracked/gitignored 文件策略、cleanup、changes apply/merge、crash 后 orphan recovery。frontmatter 只表达 policy，不存实际 worktree path。

## 14. Team 复用

同一 definition 可作为 teammate role，但 normalize 时应用 team adapter：

- 保留 prompt、model、tool/permission ceiling、color；
- team coordination tools 由 runtime 注入；
- skills/MCP 是否继承必须明确，不能像某些参考实现一样静默忽略；
- teammate 是可多轮接收消息的 session，而普通 subagent run 默认完成即返回；
- output schema 对“完成某 task 的 receipt”生效，不限制每次 peer message。

## 15. 安全

- definition body 是 untrusted instructions；
- project/package agent 必须显示来源；
- inline command hooks 与 MCP server definitions 是 executable/integration authority；
- `extends`、relative schema、skill refs 必须限制在允许 root；
- 不在 diagnostics/export 中显示 secrets；
- parent/child tool escalation 必须有测试；
- 任何 `bypass` 类 permission 只允许 user/managed 明示，不能来自 project file。

## 16. 包归属建议

建议形成一个深模块，但先放在 `packages/agent`：

- schema、parse、normalize、resolution 属于 agent runtime domain；
- coding-agent 提供 filesystem discovery/trust/UI adapter；
- team runtime 消费同一 normalized definition；
- 当 server、coding-agent 和其他 host 都稳定消费后，可提取 `@own/pi-agent-definition`。

不要把每个 agent definition 发布成 npm package；definitions 应由 project/user/package resource bundle 分发。

## 17. 验收条件

1. 每个 field 有 schema、merge、trust、runtime test；
2. 同名冲突 deterministic；
3. unknown major version fail loudly；
4. parent deny 无法被 child 提升；
5. hot reload 不改变 active run snapshot；
6. output schema strict failure 正确传播；
7. project hook/MCP 未 trust 时不执行；
8. definition 可同时用作 subagent 与 teammate，差异被 diagnostics 清楚展示；
9. full prompt 不在主 context 常驻；
10. Claude/Pi simple agent definitions 可导入并产生 migration report。

## 18. 资料来源

- 当前 Pi subagent example：`packages/coding-agent/examples/extensions/subagent/README.md`
- Claude Code subagents：https://code.claude.com/docs/en/sub-agents
- Claude Code agent teams：https://code.claude.com/docs/en/agent-teams
- OpenCode agents：https://opencode.ai/docs/agents/
- Agent Skills specification：https://agentskills.io/specification
