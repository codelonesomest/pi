# Primary Agent Profiles 研究

- 状态：方向建议，尚非实施承诺
- 更新日期：2026-08-04
- 范围：为 primary agent 提供可发现、可切换、可复用的 identity/profile；包括 `/profile` 命令、定义格式、prompt 注入、模型/工具策略边界、scope、trust 与 session 语义。
- 非范围：不把 profile 做成 subagent/team/workflow 的替代物；不实现一个新的 permission engine；不复制 OMP 的 process-wide configuration profile。

## 1. 问题与结论

当前 harness 已有 subagent、agent team 与 workflow 的研究方向，但 primary agent 只有单一默认 prompt 加 project context、skills、settings 与 session model。用户不能把“当前主助手应以什么身份、工作方式、模型意图与可见工具集合工作”保存为一个可命名、可选择、可切换的对象。

**结论：应引入 `PrimaryAgentProfile`，它是 `AgentDefinition` 的 primary-only projection，而不是一个自由文本 prompt preset。**

推荐入口是用户提出的 **`/profile`**：

```text
/profile                 # 打开可搜索 profile picker，并显示当前项
/profile current         # 显示有效 profile、来源、definition digest 与生效范围
/profile <name>          # 精确选择；非精确输入作为 picker 初始查询
/profile none            # 回到 built-in baseline profile
```

`/profile` 的 picker 应支持输入搜索、描述、来源、可见权限/工具摘要和当前项标记。它是 **当前 session 的选择**；选择后在下一次 agent turn 生效，不改写全局或 project 配置。持久化默认值、定义编辑和 session-restart recovery 必须在后续 work package 中明确加入，不能由首版隐式猜测。

## 2. 为什么不是照搬 OMP `/switch`

### 2.1 OMP 的实际语义

截至 2026-08-04，OMP `/switch`（以及 `alt+p`）是一个 **session-only model picker**：选择 concrete model 或 quick role，并明确提示“role models stay unchanged”。当选择的模型 context window 小于当前 session 时，host 先 compact 再切换。它不选择 primary-agent identity，也不切换 system prompt、tools 或 permissions。

OMP 另有 `--profile <name>` / `OMP_PROFILE`：它把用户级 `.omp` 路径重定向到 `~/.omp/profiles/<name>/agent/`，从而隔离 settings、skills、rules、hooks、tools、extensions、MCP、sessions 与 state。这是 **process-wide configuration home**，不是 session 内的 primary agent 角色；它不能作为本功能的语义基础。

因此可以借鉴 `/switch` 的紧凑、可搜索、session-local picker 体验，但不能把 model switching 或配置根隔离误称为“identity/profile switching”。

### 2.2 更接近的外部参考：OpenCode primary agents

OpenCode 明确区分 primary agent 与 subagent。primary agent 是用户直接对话的主助手；其 definition 可组合系统 prompt、model preference、permissions 与 UI metadata，并在 session 中切换。其 Build（所有工具）与 Plan（edit/bash 为 ask）说明“角色”可以同时表达工作目标与执行边界。

这一概念与需求相符，但不能直接复制其配置语义：本仓库当前只有 tool visibility preset，尚没有可被 profile 使用的统一 runtime permission enforcement。因此本项目首版必须把 visibility 与 authorization 明确分开，禁止 profile 把 prompt 中的“只读”伪装成安全隔离。

## 3. 当前 Pi 基线

### 3.1 命令与交互

- `packages/coding-agent/src/core/slash-commands.ts` 只有静态 built-in command catalog；没有 `/profile`。
- `packages/coding-agent/src/modes/interactive/interactive-mode.ts` 在 editor submit path 中直接分发 built-ins；`/model` 使用 selector，适合复用为 profile picker 的交互范式。
- prompt template、skill command 与 extension command 都是额外 command source，但它们是用户输入模板或扩展行为，不是 primary identity catalog。

### 3.2 Prompt 组装

- `packages/coding-agent/src/core/system-prompt.ts` 构造 default/custom system prompt、append text、project context files、skills 与 cwd。
- `AgentSession._rebuildSystemPrompt()` 从 `ResourceLoader` 重新读取 `SYSTEM.md`、`APPEND_SYSTEM.md`、context files 与 skills，建立 `_baseSystemPrompt`；extension 的 per-turn override 在 turn 完成后会清除。
- 当前 `SYSTEM.md` 是 custom base prompt，`APPEND_SYSTEM.md` 只是追加内容；二者都是 discovery/CLI 输入，不是多 profile catalog，也不携带 name、description、model/tool policy 或 source diagnostics。

因此不能只在 `/profile` handler 中临时给当次 turn 加一段 text：下一 turn、reload、session replacement 和 extension prompt lifecycle 都会丢失或覆盖它。profile 需要成为 `AgentSession` 的已解析 base-prompt input。

### 3.3 Settings、trust 与 capabilities

- global `~/.pi/agent/settings.json` 与 trusted project `.pi/settings.json` 由 `SettingsManager` deep merge；当前 `Settings` 没有 primary profile catalog 或 selection field。
- project-local settings/resources 受 project trust 保护；project profile definition 也必须沿用同一 admission，而不能通过 profile 绕过 trust。

### 3.4 已有研究的关系

- `04-subagents-and-frontmatter` 已建议小而稳定的 `AgentDefinition` 核心，定义/配置与 Markdown prompt body 分离，且 `mode` 区分 `primary | subagent | teammate | any | hidden`。
- 该研究同样要求 prompt body、model intent、tool/capability policy、source provenance、schema version 与 trust 进入 normalized model；definition 不是 task/PID/mailbox 等 runtime state。
- `07-settings-ui-ux` 要求 effective value、source、scope 与 reload requirement 可见，并坚持 `unset` 删除 override，而非写回 default。

## 4. 建议领域模型

### 4.1 名称与边界

在 UI 和用户文档中使用 **Primary Agent Profile**；内部以 `PrimaryAgentProfile` 或 `AgentDefinition`（`mode: "primary"`）表达。这样既满足“给主 agent 一个 identity”的认知模型，也避免再创建一套与 subagent definition 平行但无法互用的 schema。

一个 profile 表达：

1. **identity**：稳定名称、标题、描述、source、版本/digest；
2. **instruction**：Markdown body 作为 role prompt；
3. **model intent**：可选 role/selector/thinking preference；
4. **capability intent**：可选可见工具集合或已存在 capability preset；
5. **display metadata**：颜色、短标签等非行为信息。

一个 profile **不**拥有：session history、subagent roster、team mailbox、workflow graph、PID、secret、MCP credential、运行中 tool permission grant，或新的安全 override authority。

### 4.2 最小格式提案

首版应为 user/project 可发现的 Markdown definition，frontmatter declarative、body 为 prompt：

```md
---
schema: pi-primary-profile/v1
name: pragmatic-engineer
description: Implements scoped product changes with evidence and minimal abstraction.
mode: primary
model:
  role: default
  thinking: high
capabilities:
  preset: code-edit
ui:
  color: accent
---

You are the primary implementation engineer. Prefer a small verified change,
state evidence precisely, and preserve existing repository conventions.
```

`description` 是 picker/autocomplete 的 discovery text，不能依赖完整 body 常驻主 prompt。未知 major schema version 必须 fail closed，并在 `/profile` diagnostics 中显示；未知 future fields 必须 round-trip preservation 的具体策略后才能允许编辑器写回。

首版应固定 **append-to-host** 语义：profile body 被放入 host-built prompt 的专用 `<primary_agent_profile>` section，保留 default system policy、tools、skills、project context、AGENTS instructions、trust 与 completion contract。`SYSTEM.md` 风格的“replace base prompt”不应与 identity profile 绑定，因为它会让 profile 不可预期地删除 host guidance。

### 4.3 Profile resolution

建议声明式 scope（高到低）：

```text
managed policy (future)
→ session/CLI selected profile
→ trusted project definition/default
→ user definition/default
→ built-in baseline: none
→ runtime safety ceiling
```

同 scope 同名是 diagnostic error；高 scope 以 **whole-definition override** 覆盖低 scope，不做隐式 field merge。每次 resolution 生成：

```ts
interface ResolvedPrimaryAgentProfile {
  definition: PrimaryAgentProfile;
  source: "managed" | "project" | "user" | "built-in";
  digest: string;
  selection: "session" | "project-default" | "user-default" | "baseline";
  pendingEffect: "none" | "next-turn" | "new-session";
}
```

此对象是 `AgentSession` 与 picker 的共同输入。session 保留 selected profile name 和 resolved digest；definition source 可在 UI 中打开/诊断。是否把完整 resolved body snapshot 写入 legacy session、如何在 restart 后处理 definition 被删除或 digest 不匹配，是 durable session work 的明确门禁，不能在实现时默默 fallback 到同名新定义。

## 5. 命令、autocomplete 与 UI

### 5.1 推荐交互

`/profile` 打开专用 `ProfilePicker`，而不是生成上百个顶层 `/profile:<name>` commands：

```text
Primary profile                         Current session
> pragmatic-engineer  Product changes · code-edit       user
  reviewer             Read-only review · read-only      project
  planner              Plans before changes · read-only  user
  none                 Standard host behavior            built-in

Type to filter · Enter switch for next turn · Esc close
```

每一项详情至少显示：description、source path/scope、prompt digest、model intent、tool visibility request、是否因为缺少 adapter 而 degraded。该设计实现用户想要的 autocomplete/search，又不会把动态 catalog 误塞进静态 slash completion。

`/profile <name>` 可处理 scriptable 精确选择；不存在或歧义时不静默成功，应给出可操作错误并打开/建议 picker。用户在 agent run streaming 时的选择应拒绝或排队到 run settled；不得 mid-turn 改写 provider prompt。

### 5.2 生效与可见性

- 切换只影响后续 primary-agent request；已发出的 provider request、subagent run 与 workflow node 保持创建时的 profile/model/policy snapshot。
- picker/status line 应始终显示 profile name；`/profile current` 展示 source、digest 与 selection source，便于排查“为什么这个身份生效”。
- 若 profile 的 model intent 不能解析或当前无 auth，profile 可被选择但必须显示 degraded/blocked；是否切换模型、是否 compact-first，复用现有 model runtime contract，不由 profile picker 自行实现。
- 若 profile 请求不可用的 tool visibility preset，resolver 给出 diagnostics；不要悄悄把 profile 解释成另一个工具集合。

## 6. 权限与安全边界

profile body 是 instructions，不是 policy。以下不变量必须成立：

1. profile 不能提升 host、managed、project-trust 或 parent/runtime permission ceiling；
2. profile 只能请求 tool **visibility**，实际 allow/ask/deny 必须由未来 runtime permission engine 执行；
3. project profile 与 project-local extension/skill 一样，未经 trust 不加载、不显示为可选项；
4. profile 不能内嵌 raw credential、MCP server、shell hook 或 executable action；复杂配置必须引用已经受 trust/permission 管控的 object；
5. session selection、definition source/digest 和 authorization diagnostics 是可审计状态；不记录 secrets；
6. profile prompt 可以要求“只读”，但 UI 必须标明这只是 behavior guidance，除非已绑定真实 read-only permission policy。

## 7. 备选方案比较

| 方案 | 优点 | 主要问题 | 结论 |
|---|---|---|---|
| 每次 `/profile` 追加一段临时 system text | 最快 | 不能安全跨 turn/reload；缺 source/schema/diagnostics；易被 per-turn override 清除 | 拒绝 |
| 利用已有 prompt templates | 已有文件与 command discovery | template 是用户消息快捷输入，不是 primary agent runtime identity | 拒绝 |
| 让用户直接切 `SYSTEM.md` / `APPEND_SYSTEM.md` | 无新 domain | 文件是单个 discovery winner，不支持 catalog、session selection 或 profile metadata；修改也不适合 session safety | 拒绝 |
| process-wide config root profiles（OMP `--profile`） | 强隔离所有配置 | 改变 sessions、credentials、extensions 等整个 home；远大于换主 agent 身份 | 不作为本功能 |
| 独立 `PrimaryAgentProfile`，归一化到 `AgentDefinition` | 语义完整、可复用、可发现；后续可共享 schema | 需要 profile resolver、session snapshot 和 permission boundary work | 推荐 |

## 8. 分阶段建议与门禁

### Phase A：只读 discovery spike

先验证以下未知，不修改 production command 或 prompt behavior：

- 当前 resource discovery 是否能以 user/project + trust 载入 profile Markdown；
- `AgentSession` 能否在 idle boundary 原子切换 resolved base prompt，而不被 extension per-turn override/reload 覆盖；
- session persistence 当前可否安全表达 selected name + immutable definition digest；
- profile model intent 与 active session context-size/compaction behavior 的准确接点；

该 spike 的 observable output 是 discovery matrix、prompt ordering fixture 与 restart/missing-definition result；临时代码不得进入 production path。

### Phase B：最小可用 vertical slice

通过 Phase A 后，限定为：

- user-scope definitions + built-in `none`；
- `/profile` searchable picker 与 `/profile current`；
- next-turn identity-prompt switching；
- profile name/source/digest status；
- no model/tool/permission mutation；
- targeted UI/prompt tests与真实 interactive smoke test。

先不引入 project definitions、default profile、definition editing、model switching、tool changes或 session persistence migration。这样可以证明用户可观察的 identity selection，同时不伪称有安全或 durability 语义。

### Phase C：受信 project definitions 与 session recovery

只有 session/storage research 支持稳定 selection event 与 exact snapshot/recovery policy 后，才加入：

- trusted project definition discovery；
- project/user precedence diagnostics；
- new-session default resolution；
- resume after deletion/rename/digest mismatch 的明确 fail-closed UX；
- persisted selection 的 migration/rollback contract。

### Phase D：统一 agent definition/policy

待 `04-subagents-and-frontmatter` 的 normalized `AgentDefinition` 与 runtime permission enforcement 可用后，才收敛 primary/subagent/team 共用 schema，并允许 profile 声明 model/tool/permission policy。不得在此之前把 prompt guidance 当作工具权限。

## 9. 首个 implementation work package 的验收合同

开始 production implementation 前至少需要：

1. **选择**：在一份 user profile catalog 中输入 `/profile`，键盘搜索并选择 profile；下一次 agent request 带有 profile section，标准 host prompt、skills 与 project context 仍存在。
2. **隔离**：选择不会改变正在运行的 request、已启动 subagent 或当前 provider request 的 system prompt。
3. **恢复边界**：reload 后 profile selection 的行为符合明确 contract；若 restart durability尚未实现，UI 必须明确提示它是 session-memory-only，而不能伪称永久保存。
4. **安全**：profile body 要求 write 时，host tool/call-time policy 不因 profile 改变；untrusted project profile 不进入 picker。
5. **诊断**：同名冲突、未知 schema major、不可解析 model/preset 都显示 source-linked actionable diagnostic。
6. **验证**：新增的行为测试、目标 interactive command test、一次真实 TUI smoke test，以及仓库规定的 `npm run check`。

任何需要修改 durable session format、public AgentDefinition format、trust/authorization policy 或跨多个 runtime layer 的工作，均须先满足 `90-research-governance/implementation-research-gate.md` 的 current-state、failure、migration 与独立 review 门禁。

## 10. 开放问题

1. profile definition 应放进新 `profiles/` root，还是等 `AgentDefinition` catalog 后由 `agents/` 以 `mode: primary` 统一发现？建议 Phase A 以两者的 discovery collision 为实验问题，避免永久双目录。
2. “切换 profile”是否应同时切换 model/thinking？建议 Phase B 不切；Phase D 才由 model policy 统一处理。
3. primary profile 是否允许 profile-specific skills/MCP？建议首版不允许；需确保 preload 成本、trust 和 permission ceiling 明确后再加。
4. 默认 profile 应是 global user setting、project setting，还是每个 session prompt 决定？建议先只有 explicit session selection，避免在 session durable contract 未冻结时创造第二个真相。
5. 是否需为 profile selection 建独立 keybinding？先交付 `/profile`；有重复使用数据和 keybinding action registry 后再决定，不能硬编码快捷键。

## 11. Sources

### Repository evidence

- `packages/coding-agent/src/core/slash-commands.ts` — current static built-in command catalog.
- `packages/coding-agent/src/modes/interactive/interactive-mode.ts` — direct built-in dispatch and `/model` selector pattern.
- `packages/coding-agent/src/core/system-prompt.ts` — default/custom/append prompt assembly and ordering.
- `packages/coding-agent/src/core/agent-session.ts` — base system prompt rebuild and per-turn extension override lifecycle.
- `packages/coding-agent/src/core/settings-manager.ts` and `packages/coding-agent/docs/settings.md` — layered settings, trust and tool visibility preset boundary.
- `docs/research/04-subagents-and-frontmatter/research.md` — unified AgentDefinition, provenance, scope, trust and policy direction.
- `docs/research/07-settings-ui-ux/research.md` — effective value/source/scope/reload UX direction.
- `docs/research/90-research-governance/implementation-research-gate.md` — pre-production research gate.

### External primary sources

- [OpenCode Agents](https://opencode.ai/docs/agents/) — primary vs subagent semantics, agent definition inputs, primary switching and permission model. Accessed 2026-08-04.
- [OMP model picker source](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/modes/components/model-picker.ts) — `/switch` is a session model picker. Accessed 2026-08-04.
- [OMP Configuration Discovery and Resolution](https://github.com/can1357/oh-my-pi/blob/main/docs/config-usage.md) — named profile relocates OMP user configuration roots. Accessed 2026-08-04.
- [OMP System Prompt Customization](https://github.com/can1357/oh-my-pi/blob/main/docs/system-prompt-customization.md) — `SYSTEM.md`/`APPEND_SYSTEM.md` assembly and profile-scoped config roots. Accessed 2026-08-04.
