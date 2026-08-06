# 内置 MCP 研究

- 状态：架构与安全方向建议
- 更新日期：2026-07-31
- 范围：Harness 内置 MCP client、server lifecycle、tools/resources/prompts、auth、tasks 与 UI；并区分未来将 Harness 暴露为 MCP server 的能力
- 核心结论：内置 MCP 应是独立 integration subsystem，由 capability discovery 按需暴露；不能只是启动几个 stdio process 然后把所有 tools 塞进 context

## 1. “内置 MCP”需要拆成两种角色

### 1.1 MCP Client（首要）

Harness 连接第三方 local/remote MCP servers，把它们的 tools、resources、prompts 等能力接入 agent。

### 1.2 MCP Server（后续、独立）

Harness 自己对 IDE、其他 agents 或 automation 暴露 sessions/tools/resources。它的 auth、multi-tenant、permission 与生命周期完全不同。

两者可以共用 protocol types/transport，但不能用一个 manager 混在一起。首发范围应是 client；server mode 需单独威胁模型和 product contract。

## 2. 当前 Pi 基线

当前 Pi 官方定位明确：不内置 MCP，用户通过 extensions/packages 自行实现。这让核心保持小，但无法自然提供：

- 统一 server config、auth 与 connection UI；
- lifecycle/restart/health；
- tool discovery 与 context budget；
- resource URI；
- session durability；
- trust/permission；
- cross-provider behavior；
- MCP tasks/elicitation 的标准 UI。

本 fork 若以强 harness 为目标，MCP 值得成为 first-class subsystem，但仍应模块化，避免侵入 upstream-sensitive agent loop。

## 3. 协议范围

依据 MCP 2026-07-28 规范，client 至少要理解：

- initialization 与 capability negotiation；
- tools：list/call/listChanged；
- resources：list/read/subscribe/templates；
- prompts：list/get；
- logging/progress/cancellation；
- roots；
- client-side capabilities，如 sampling、elicitation；
- tasks 扩展（仍需 capability negotiation 和 feature flag）；
- stdio 与 Streamable HTTP transports；
- remote authorization/OAuth 相关流程。

不能看到 server 宣称能力就默认启用；每项需协商、policy 与 UI adapter。

## 4. 建议模块

```text
mcp-core/
  protocol types
  capability negotiation
  client state machine
  request correlation
  cancellation/progress
mcp-transport-stdio/
mcp-transport-http/
mcp-auth/
mcp-host/
  config resolution
  connection pool
  catalog adapter
  permission/trust
  session/resource integration
```

核心接口：

```ts
interface McpClientConnection {
  readonly server: McpServerIdentity;
  readonly capabilities: McpServerCapabilities;
  state(): McpConnectionState;
  listTools(options?: CachePolicy): Promise<McpTool[]>;
  callTool(name: string, input: JsonValue, options: CallOptions): Promise<McpCallResult>;
  listResources(options?: CachePolicy): Promise<McpResource[]>;
  readResource(uri: string): Promise<McpResourceContents>;
  listPrompts(options?: CachePolicy): Promise<McpPrompt[]>;
  getPrompt(name: string, args: JsonValue): Promise<McpPromptResult>;
  close(): Promise<void>;
}
```

MCP SDK types 不应泄漏到 agent-core tool interfaces；host adapter 将其 normalize 为 capability/resource/task contracts。

## 5. 配置模型

```yaml
mcp:
  servers:
    github:
      transport: http
      url: https://example/mcp
      auth: oauth
      enabled: true
      startup: lazy
      trust: external
      expose: deferred
      timeout: 30s
    local-docs:
      transport: stdio
      command: [node, ./server.mjs]
      cwd: ${workspace}
      env:
        DOCS_ROOT: ${workspace}/docs
      enabled: project-trusted
```

必须支持：

- user/project/managed scope 与 precedence；
- enable/disable；
- eager/lazy/manual startup；
- tool/resource/prompt allow/deny patterns；
- per-server timeouts/restart；
- secret references，不能把 raw token 写进普通 config；
- server identity pinning/version metadata；
- project-local command 需 trust。

数组 `command` 优于 shell string，避免 quoting/injection。

## 6. Lifecycle state machine

```text
disabled
  → configured
  → starting/authenticating
  → initializing
  → ready
  → degraded
  → reconnect_wait
  → stopped
  → failed
```

每次 transition 有 durable/observable event。规则：

- startup lazy，不让坏 server 阻塞整个 TUI；
- stdio child 必须 process-tree supervision；
- HTTP reconnect 使用 backoff/jitter/Retry-After；
- protocol/auth/config errors 不应无限 retry；
- health 与 catalog cache 分开；
- shutdown 先取消 requests、close transport、再终止 child；
- server stderr 进入 bounded diagnostic artifact，不混入 protocol stream。

## 7. Tools

MCP tool 转成统一 capability：

```text
mcp.<server>.<tool>
```

用户展示可用友好 title，但 stable ID 必须 namespaced。

调用前：

1. tool 仍在 current catalog；
2. input schema validate；
3. permission/risk/approval；
4. server connection ready；
5. request ID 与 session operation durable record；
6. progress/status 映射到 tool UI；
7. result content normalize；
8. large/binary output spill artifact；
9. error 分类为 transport/protocol/tool/auth/policy。

server 的 annotations/risk hints 只能作为提示，不能当可信 permission declaration。

## 8. Resources 与 `mcp://`

MCP resources 不应伪装成 tool output。建议 internal URI：

```text
mcp://<server>/<encoded-resource-uri>
```

或以 lookup ID 避免双重 URI 解析。router 需要保留远端原始 URI 字节，用 exact identifier 发回 server。

支持：

- list/template autocomplete 只使用 bounded local cache；
- read 时 lazy connect；
- MIME-aware content；
- subscriptions → cache invalidation/event，不自动把更新内容推入 model context；
- remote resource provenance；
- size limits/artifact spill；
- resources 默认 immutable，除非未来协议和 handler 明确提供 write contract。

## 9. Prompts

MCP prompt 是模板化内容，不是系统级可信 instructions：

- 用户显式选择或 agent 经 permission 调用；
- result 标记 server/source；
- prompt messages 进入普通 untrusted conversation/context layer；
- resource attachments 仍受 size/policy；
- argument autocomplete 不连接每次 keystroke，使用 cached schema/options；
- prompt 不能修改 host system prompt、tool permission 或 trust ceiling。

## 10. Sampling、Elicitation 与 Roots

### Sampling

server 请求 client 代表它调用 model。风险是成本与 prompt injection：

- 默认关闭；
- per-server allow/ask/deny；
- model role、token/cost budget、tools 明确限制；
- nested depth 与 loop detection；
- 用户可查看 server 提交的 request；
- usage 归属到 initiating MCP operation。

### Elicitation

server 要求用户输入：

- 转成统一 `input_required`/`ask` UI；
- secret field 使用 secure input 且不进入 transcript；
- URL/open-browser 有 host allowlist；
- unattended mode 根据 policy fail/block，而不是填写虚假默认。

### Roots

只暴露当前允许 workspace roots：

- path canonicalize；
- 不自动暴露 home、session store、secrets；
- root change notifications；
- per-server scope；
- project trust gate。

## 11. MCP Tasks

Tasks 适合长运行 tool call，但协议扩展仍应 feature-gated。映射到统一 operation model：

| MCP task state | Harness state |
|---|---|
| working | running |
| input_required | input_required/blocked |
| completed | succeeded |
| failed | failed |
| cancelled | cancelled |

需持久化 task ID、server identity、poll/reconnect token、expiry。进程重启后可查询 remote task；若 server 不支持 task durability，就标 interrupted，不能伪称 resume。

## 12. Auth 与秘密

Remote MCP auth：

- OAuth authorization code + PKCE 等流程优先；
- Dynamic Client Registration 只在 policy 允许时；
- redirect listener 有随机 state、loopback binding、timeout；
- token 存 OS keychain/credential store；
- config 只保存 credential reference；
- refresh 串行化；
- 401 触发一次 refresh/re-auth，不无限循环；
- token、Authorization header、client secret 不进入 logs/session/model；
- remote server origin/change 使 credential binding 失效。

Static headers 可以引用 env/vault，但 UI 不显示 resolved secret。

## 13. Local stdio 安全

Local MCP server 是可执行代码：

- project config 未 trust 不启动；
- 显示 exact argv、cwd、env names、package source；
- 不默认 `npx -y latest`，dependency/version 需要 pin/review；
- lifecycle scripts/package download 与 repo 安全规则一致；
- sandbox/workspace filesystem/network policy；
- env allowlist；
- stdout 只允许 MCP frames，stderr bounded；
- child process tree 可可靠停止。

## 14. Discovery 与 Context

所有 MCP tools 必须接入 `08-tool-discovery`：

- configured server metadata 可在 disconnected 状态入 catalog；
- tool list cache 可从上次成功连接恢复，但标 stale；
- selected tool 才 materialize schema/connection；
- `tools/listChanged` invalidates catalog；
- server 数和 tool schema tokens 有预算；
- 用户可 pin 某些 MCP tools 为 always-visible；
- startup 不全量连接所有 servers。

OpenCode 官方文档明确提醒 MCP tools 会快速增加 context，这也是不能使用 eager-all 策略的直接证据。

## 15. UI/UX

Settings → MCP 页面：

```text
MCP Servers
  github       ready       OAuth       42 tools · deferred
  local-docs   stopped     project     8 tools
  jira         auth needed remote      —
```

详情：

- source/config scope；
- transport/endpoint（去 secret）；
- trust/auth/health；
- capabilities；
- cached catalog；
- loaded tools/token estimate；
- Test / Connect / Authenticate / Restart / View logs；
- allow/deny patterns。

Tool card 显示 server identity；errors 区分 server tool error 与 connection error。

## 16. Durability

Session journal 保存：

- server config identity/digest（非 secret）；
- connection and auth-required events；
- tool request/result/task refs；
- resource refs；
- elicitation/sampling decisions；
- catalog digest used for call。

不保存 socket/process handle。resume 时 manager 重新建 connection，并 reconcile remote tasks。

## 17. Observability

per server：

- connection state/restarts；
- initialization latency；
- list cache hit/staleness；
- call count/error/latency；
- bytes/token estimate；
- auth refresh；
- task duration；
- sampling cost。

日志默认 redaction，能够导出 server-specific diagnostic artifact。

## 18. 兼容与 SDK 策略

MCP 协议仍演进。建议：

- pin exact SDK version；
- protocol adapter 隔离 SDK types；
- capability negotiation 而不是 version if/else 到处散落；
- fixture server 覆盖 stdio/HTTP/listChanged/progress/cancel/auth/tasks；
- unknown content/type 保留并安全降级；
- experimental features 独立 flag；
- upgrade SDK 前 replay protocol corpus。

## 19. MCP Server mode（后续）

若将 Harness 暴露为 MCP server，首批可提供：

- sessions as resources；
- start/send/cancel/wait as tools；
- plans/tasks as resources/tools；
- capability catalog as resource。

但必须先解决：

- local/remote auth；
- tenant/session boundary；
- user approval routing；
- tool authority delegation；
- rate/cost limits；
- streaming/long tasks；
- server shutdown/recovery。

不要简单把所有内部工具无条件转发。

## 20. 包归属建议

建议独立包边界：

- `@own/pi-mcp-core`：client state、normalized protocol contracts；
- `@own/pi-mcp-node`：stdio/HTTP/process/OAuth credential adapters；
- `@own/pi-capabilities`：catalog integration；
- coding-agent：config、trust、permission、TUI/session adapter；
- server mode 未来另建 adapter，不污染 client manager。

最初可只建 core + node 两层，避免一 capability 一 package。

## 21. 验收场景

1. disabled server 不启动；
2. project stdio config 未 trust 不执行；
3. lazy tool load 后才连接；
4. listChanged invalid cache；
5. malformed frame/timeout 只使该 server degraded；
6. OAuth token 不进 logs/session；
7. tool input/result schema 与 artifact 正确；
8. elicitation secret 不进入 transcript；
9. sampling 受预算和 permission；
10. remote task 在 restart 后 reconcile；
11. roots 不越过 workspace policy；
12. unrelated MCP failure 不阻塞 TUI startup。

## 22. 资料来源

- 当前 Pi 明确无 built-in MCP：`packages/coding-agent/docs/usage.md`
- MCP 2026-07-28 specification：https://modelcontextprotocol.io/specification/2026-07-28
- MCP tools：https://modelcontextprotocol.io/specification/2026-07-28/server/tools
- MCP tasks extension：https://modelcontextprotocol.io/extensions/tasks/overview
- OpenCode MCP servers：https://opencode.ai/docs/mcp-servers/
- Codex config reference（MCP settings）：https://developers.openai.com/codex/config-reference
- Tool discovery 研究：`../08-tool-discovery/research.md`
