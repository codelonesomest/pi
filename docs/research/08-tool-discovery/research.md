# Tool、MCP 与 Skill 按需发现研究

- 状态：架构方向建议
- 更新日期：2026-07-31
- 范围：启动时减少 tool/schema/skill/MCP context，占用时再发现和加载
- 核心结论：建立统一 capability catalog 与分层 materialization；“发现”与“授权、连接、执行”必须分开

## 1. 问题

Harness 的能力来源会持续增加：

- built-in tools；
- optional/experimental tools；
- extension tools；
- MCP server tools/resources/prompts；
- skills；
- agents；
- internal URI handlers。

如果启动时把所有 JSON Schema、description、skill body、MCP catalog 都塞进 system prompt，会同时造成：

- 固定 context 成本；
- provider prompt cache 体积增大；
- tool 数超过模型稳定选择范围；
- 不相关描述干扰选择；
- MCP server 冷启动与失败拖慢 session；
- config/permission 变化导致缓存前缀不稳定。

Anthropic 的官方 tool search 文档给出的典型多 server catalog 可达到约 55k tokens，并指出超过约 30–50 个可见 tools 后选择准确度下降。其 server-side tool search 通常只加载 3–5 个匹配工具，context 减少超过 85%。这些数字是 Anthropic 对其支持模型/API 的公开说明，不应直接当作所有 provider 的保证，但足以证明问题存在。

## 2. 当前 Pi 基线

Pi skills 已经做 progressive disclosure：

1. startup 扫描 name/description；
2. system prompt 只列 metadata；
3. 匹配任务时读取完整 `SKILL.md`；
4. references/scripts/assets 再按需读取。

但当前仓库明确不内置 MCP；built-in/extension tool definitions 通常仍作为 active tools 直接进入 provider request。当前缺少一个跨 tool/MCP/skill/agent 的统一 catalog、搜索与 load 生命周期。

## 3. 参考实现

### 3.1 Agent Skills

Agent Skills 规范本身定义三层 disclosure：

- metadata 约 100 tokens；
- activated `SKILL.md` 建议 < 5000 tokens；
- resources 按需。

这应继续保持，不要为了统一 discovery 把 skill body 转成 tool schema 常驻。

### 3.2 Anthropic Tool Search

官方机制：

- provider request 仍包含 deferred tool definitions，但标 `defer_loading: true`；
- context 初始只出现 search tool 与 non-deferred tools；
- regex/BM25 search 返回 `tool_reference`；
- API 展开完整 schema 后模型再调用。

优点：provider-native、模型理解引用。限制：只适用于支持该功能的 Anthropic models/transport；不能成为 harness 唯一策略。

### 3.3 OMP xd://

OMP 将少用工具放在 `xd://` devices 后，通过 `read xd://` 发现、`write xd://<tool>` 执行。优点是 provider-agnostic、启动 context 小；代价是工具调用变成 read/write 的二阶段协议，typed schema 与 approval UI 需由 host 补回。

## 4. 统一 Capability Catalog

建议所有可发现能力先归一化为 catalog item：

```ts
type CapabilityKind = "tool" | "skill" | "agent" | "mcp-tool" | "mcp-resource" | "mcp-prompt" | "uri";

interface CapabilityDescriptor {
  id: string;
  kind: CapabilityKind;
  name: string;
  title?: string;
  description: string;
  keywords: string[];
  source: CapabilitySource;
  trust: TrustLevel;
  cost: {
    contextTokensEstimate: number;
    coldStart?: boolean;
    network?: boolean;
  };
  risk: "read" | "write" | "execute" | "external";
  availability: "ready" | "disabled" | "needs-auth" | "offline" | "error";
  digest: string;
}
```

完整 payload 分开：

```ts
interface CapabilityMaterialization {
  descriptor: CapabilityDescriptor;
  toolSchema?: JsonSchemaTool;
  skillRoot?: string;
  agentDefinition?: AgentDefinitionRef;
  connection?: McpConnectionRef;
}
```

Catalog 是 metadata index；materialization 才连接 server、读取 body 或生成 provider tool schema。

## 5. 分层加载模型

### Tier 0：Essential

始终可见、极小且高频：

- read/search/glob；
- edit/write；
- shell/process；
- capability discovery；
- ask/approval；
- task/todo（按 session mode）。

数量应保持小，实际集合由 mode/permission 决定。

### Tier 1：Metadata catalog

只保留 compact descriptor，通常不直接给模型完整 catalog；search index 在 host。本层覆盖所有 optional capabilities。

### Tier 2：Loaded for turn/session

匹配当前 intent 后加载完整 schema/body：

- 一次 turn；
- 固定 N turns；
- 当前 session pinned；
- explicit user pin。

### Tier 3：Runtime activation

真正发生：

- MCP connect/auth；
- executable skill script permission；
- optional tool backend startup；
- agent spawn。

发现或 materialize 不等于执行许可。

## 6. Discovery Tool

建议提供 provider-neutral primitive：

```ts
interface DiscoverInput {
  query: string;
  kinds?: CapabilityKind[];
  source?: string[];
  risk?: string[];
  limit?: number;
}

interface DiscoverResult {
  matches: Array<{
    id: string;
    kind: CapabilityKind;
    name: string;
    description: string;
    source: string;
    availability: string;
    score: number;
  }>;
}
```

然后：

```ts
load_capabilities({ ids: [...], scope: "turn" | "session" })
```

可把 discover/load 合成一个工具减少 round-trip，但内部仍保持两阶段语义。对于 Anthropic provider，adapter 可映射到 native tool search；其他 provider 使用 client-side BM25/regex/embedding search。

## 7. 搜索策略

第一版使用 deterministic hybrid：

1. exact/name/prefix；
2. keyword/alias；
3. BM25 over name/description/schema property names；
4. optional embedding rerank；
5. policy/availability filter。

不要一开始依赖 embedding：catalog 多数是短技术文本，BM25 易诊断、可离线、稳定。需要记录 query→matches→loaded→called telemetry，之后用真实数据优化 description 与 ranking。

## 8. Tool 生命周期

```text
discovered → materialized → exposed → called → retained/evicted
```

规则：

- descriptor digest 变化会使 materialization cache invalid；
- provider tool list 顺序 deterministic；
- 同一 cached prefix 中已经 exposed 的 tool 不应频繁重排；
- session load 有 LRU/上限；
- 用户可 pin/unpin；
- permission/config 变化可立即 hide/deny；
- failed backend 不从 catalog 消失，标 availability/error 以便诊断。

## 9. Skills 的特殊处理

Skill 不是 function tool：

- discovery 返回 metadata；
- activation 读取 `SKILL.md`；
- skill body进入 instructions/context，而不是 tools array；
- references 再通过 `skill://` 读取；
- skill script 运行仍走 shell/eval permission；
- `disable-model-invocation` 的 skill 不进入 model discovery，只在用户显式命令可见；
- skill 的 `allowed-tools` 不能扩大当前 permission ceiling。

## 10. MCP 的特殊处理

- server descriptor 可在不连接时存在；
- tool catalog 可有 TTL/ETag/cache；
- `tools/listChanged` 使 cache invalid；
- server tools 可先只索引 metadata，选中后再建立/复用 connection；
- remote auth 应在用户真正使用或显式 test 时触发；
- connection failure 不阻塞 unrelated startup；
- malicious server description 视为 untrusted metadata；
- 同名 tools 用 namespaced ID，例如 `mcp.github.create_issue`，display title 可简化。

## 11. Provider Adapter

```ts
interface ToolExposureStrategy {
  prepareTurn(input: {
    essential: MaterializedTool[];
    catalog: CapabilityCatalog;
    loaded: LoadedCapabilitySet;
    model: ModelCapabilities;
  }): Promise<ProviderToolExposure>;
}
```

策略：

- `native-deferred`：Anthropic tool search 等；
- `client-search`：只给 discover/load primitive；
- `small-catalog`：tool 数低于阈值时全量；
- `manual-only`：严格环境只允许用户 pin。

不要在 domain 层写 provider if/else。

## 12. Context 与 Prompt Cache

- descriptors 的排序与序列化 deterministic；
- loaded set 只在 turn boundary 改变；
- 不在 streaming response 中途修改当前 provider tool list；
- listChanged 先标 stale，下一 safe request 重建；
- materialized schema digest 可复用；
- 记录 tool schema token estimate；
- compaction 不应丢失 session-pinned capabilities，但可重新由 durable loaded-set event 恢复。

## 13. TUI/Settings

需要：

- Capability browser：Tools / Skills / Agents / MCP；
- status：loaded、deferred、disabled、auth、error；
- source/trust/risk；
- pin for session/default；
- token estimate；
- search test：输入 query，显示 ranking；
- MCP connection test；
- duplicate/collision diagnostics。

status line 可显示 `tools 8 loaded / 142 discoverable`，但默认不要制造噪音。

## 14. 安全

- description/schema 是 untrusted；
- discovery 结果不能自动获得执行 authority；
- project/package/MCP source 保留 provenance；
- hidden capability 不可通过猜 ID 绕过；
- server/tool list changes 重新评估 permission；
- executable skill/MCP local command 需 project trust/approval；
- catalog search 不泄露未授权 tool names；
- model 不能自行启用 disabled server 或读取 credentials。

## 15. 性能指标

- startup 不启动 disabled/deferred MCP processes；
- 1000 capability local search p95 < 20ms；
- essential definitions预算可配置，默认目标 < 5k tokens；
- 每 turn loaded tool schema 目标 < 10k tokens，超限 warning；
- catalog update 不阻塞 TUI；
- cold MCP activation 有明确 progress/timeout；
- search recall/precision 用 recorded intent fixtures 评测。

## 16. 包归属建议

建议独立深模块 `@own/pi-capabilities`，前提是至少 tool、skill、MCP 三种 source 共同使用：

- descriptor/catalog/index/load policy；
- 不依赖 TUI、provider 或具体 filesystem；
- coding-agent adapters：built-in/extension/skills；
- MCP adapter；
- provider exposure adapters；
- TUI browser adapter。

这个 package 比“每类 discovery 一个包”更深、更能减少重复。

## 17. 分阶段建议

1. 统一 catalog + diagnostics，不改变 provider exposure；
2. optional built-in tools 走 client-side discover/load；
3. skills catalog 复用；
4. MCP tool metadata 接入；
5. Anthropic native deferred adapter；
6. ranking eval 与 auto-pin/eviction。

## 18. 验收场景

1. 500 optional tools 启动 context 不线性增长；
2. query 能找对跨 source capability；
3. disabled/unauthorized tool 不出现在结果；
4. load 后下一 turn 才暴露 schema；
5. MCP listChanged 正确 invalid cache；
6. provider 不支持 native search 时可用 client strategy；
7. skill activation 不混入 tools array；
8. session resume 恢复 pinned set；
9. description injection 不提升权限；
10. catalog ordering stable，prompt cache 不因 nondeterminism 失效。

## 19. 资料来源

- 当前 Pi skills：`packages/coding-agent/docs/skills.md`
- 当前 Pi extensions/tools：`packages/coding-agent/docs/extensions.md`
- Agent Skills specification：https://agentskills.io/specification
- Anthropic Tool Search：https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool
- MCP tools：https://modelcontextprotocol.io/specification/2026-07-28/server/tools
- OpenCode MCP caveat：https://opencode.ai/docs/mcp-servers/
- OMP README（discoverable xd tools）：https://github.com/can1357/oh-my-pi
