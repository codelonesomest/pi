# Settings Menu UI/UX 研究

- 状态：信息架构与交互方向建议
- 更新日期：2026-07-31
- 范围：TUI settings、schema-driven config、scope/source、搜索、诊断与复杂子配置
- 核心结论：从“平铺可循环值列表”升级为 schema-driven settings workspace；简单值可内联修改，复杂配置进入专用页面

## 1. 用户任务

设置界面必须让用户完成四类任务：

1. 发现有哪些能力与配置；
2. 看懂当前值来自哪里、是否被更高层覆盖；
3. 安全修改并立即理解影响；
4. 诊断“为什么我的设置没有生效”。

一个长平铺列表即使可搜索，也不能可靠处理 model roles、fallback chains、MCP servers、tool permissions、compaction strategies、agent/team policies 等结构。

## 2. 当前 Pi 基线

当前 Pi 已有：

- global 与 project JSON settings，project override global；
- `/settings` common options UI；
- `SettingsList` 支持 search、选中、description、循环 values、submenu；
- `SettingsSelector` 已包含 model/thinking、theme、images、compaction、skills commands、queue、transport、terminal、trust 等多类 callback；
- theme submenu 支持 preview；
- project trust 保护 project-local config/resources；
- settings manager 有完整 typed defaults/merge/persist 逻辑。

相关实现：

- `packages/tui/src/components/settings-list.ts`
- `packages/coding-agent/src/modes/interactive/components/settings-selector.ts`
- `packages/coding-agent/src/core/settings-manager.ts`
- `packages/coding-agent/docs/settings.md`

主要问题：UI 与 settings fields 通过大型 `SettingsConfig`/`SettingsCallbacks` 手工同步；分类不够显式；用户看不到 effective value 的来源；复杂数组/records 不能友好编辑；新增 settings 会扩大 selector 文件与 callback surface。

## 3. OMP 参考

OMP settings 使用 schema 作为 type/default/enum/description 的来源，并提供：

- YAML global/project/CLI/runtime layers；
- `omp config list/get/set/reset`；
- `/settings` 分 tab 展示；
- dotted path；
- source disabling、path-scoped settings、fallback chains 等复杂配置；
- schema-driven parse/validation；
- effective values 与 clear precedence。

值得借鉴的是“schema 是 CLI、TUI、docs 的共同真相”，但 project settings 只能手改和 reset 写默认等具体选择不必照搬。

## 4. 设计原则

1. **先显示 effective state，再编辑 source**。
2. **来源可见**：default/global/project/CLI/runtime/managed。
3. **作用域明确**：修改 global 还是 project，不能暗中写错文件。
4. **简单设置一步改，复杂设置专页改**。
5. **搜索匹配 label、path、description、enum、tags**。
6. **错误就地解释，不靠启动 warning 滚走**。
7. **不可编辑 override 显示原因与来源**。
8. **安全相关设置不使用模糊 toggle 文案**。
9. **支持键盘完整操作，鼠标只是等价入口**。
10. **schema 不等于 UI**：schema 提供事实，curated presentation 提供信息架构。

## 5. Schema 建议

```ts
interface SettingDefinition<T> {
  path: SettingPath;
  type: SettingType<T>;
  defaultValue: T;
  title: string;
  description: string;
  group: SettingGroupId;
  tags?: string[];
  enum?: SettingChoice<T>[];
  scope: { global: boolean; project: boolean; session: boolean };
  restart: "none" | "reload" | "session" | "process";
  sensitivity?: "normal" | "secret";
  experimental?: boolean;
  visibleWhen?: SettingPredicate;
  validate?: (value: T, context: ValidationContext) => ValidationIssue[];
  editor?: SettingEditorKind;
}
```

resolve 输出：

```ts
interface ResolvedSetting<T> {
  definition: SettingDefinition<T>;
  effectiveValue: T;
  source: SettingSource;
  layers: SettingLayerValue<T>[];
  editableAt: SettingScope[];
  issues: ValidationIssue[];
}
```

## 6. 信息架构

建议一级分类：

- Models & Reasoning
- Tools & Permissions
- Agents & Teams
- MCP & Resources
- Context & Memory
- Retry & Reliability
- Sessions & Storage
- Editor & Input
- TUI & Appearance
- Network & Providers
- Trust & Privacy
- Advanced / Experimental

每组显示 count、warning badge、modified count。不要把“全部设置”作为唯一视图。

### 主界面布局

宽终端：

```text
Settings  [Global ▼]  Search: retry
Groups                Results
Models                 Retry enabled        on      global
Tools                  Max attempts         3       default
> Retry & Reliability  Fallback chain       2 models project
Sessions
...
                       Description / source / restart note
```

窄终端使用 group → list → detail 三层 push navigation。

## 7. Scope 与来源

顶部必须选择 edit scope：

- Global；
- Project；
- Session override。

每一项显示 effective source：

```text
Theme        dark          global
Retry        on            project overrides global
Model        gpt-x         CLI override · read-only
```

详情页显示完整层：

```text
Default: dark
Global:  titanium
Project: —
CLI:     light  ← effective, read-only
```

操作：

- Set at selected scope；
- Remove override（不是“写回默认”）；
- Copy effective config path；
- Open source file；
- Explain precedence。

这能解决“reset 写 default 反而继续覆盖 lower layer”的常见混乱。

## 8. 编辑器类型

| 类型 | 交互 |
|---|---|
| boolean | explicit On/Off，不用真假字符串循环 |
| enum | searchable selector + description |
| number | input + min/max/unit + presets |
| duration | 人类单位输入 + canonical preview |
| string | text editor；secret mask |
| path | path autocomplete/validation |
| string array | chips/list reorder |
| record | key/value table |
| model selector | provider/model search、availability/auth 状态 |
| permission | allow/ask/deny matrix + pattern editor |
| fallback chain | ordered model list、drag/reorder、test resolve |
| MCP server | dedicated connection/editor/status page |
| compaction strategy | strategy cards + thresholds + simulation |

不要为了 schema-driven 而用 generic JSON textarea 编辑所有复杂值。

## 9. Preview 与副作用

- theme：实时 preview，cancel 恢复；
- editor padding/output padding：preview 当前 transcript/editor fixture；
- tool UI density/symbol preset：fixture preview；
- compaction：显示以当前 model/context 何时 trigger 的 simulation；
- retry：显示 delay sequence 与 fallback resolution；
- model role：即时验证 provider/auth/context；
- MCP：单独“Test connection”，不在每次光标移动时连接；
- restart/reload requirement 在保存前显示。

## 10. 搜索与命令面板

搜索结果必须可直接到 setting，不要求先选组。支持：

- `retry`；
- `path:compaction.*`；
- `source:project`；
- `modified:true`；
- `issue:error`；
- `experimental:true`。

首版可以只做普通 fuzzy search + filters chips，不需要完整 query language；但数据模型应允许。

## 11. Validation 与 diagnostics

三层：

1. field validation：类型、范围、enum；
2. cross-field validation：例如 fallback enabled 但 chain 空；
3. runtime diagnostics：provider unavailable、MCP auth fail、skill path missing。

保存 invalid config 应默认拒绝；允许保留未知 future keys，但 UI 标记“unknown to this version”，不能删除它们。

提供 `/doctor` 或 Settings → Diagnostics：

- config parse/schema；
- source/precedence；
- duplicate resources；
- unavailable models；
- MCP connection；
- conflicting context managers；
- package/extension version compatibility。

## 12. 权限与危险设置

安全配置要展示行为而非内部枚举：

```text
Workspace writes
  Ask every time
  Allow inside workspace
  Deny

Shell commands
  Default: Ask
  Rules: git diff * → Allow
         rm -rf *   → Deny
```

- broad allow 显示 warning；
- project scope 不能提升 managed/user ceiling；
- secret fields 不显示/copy raw value；
- destructive reset/clear requires confirmation；
- disabled project trust 时 project settings 清楚显示未加载。

## 13. 可扩展性

extension/package 可注册 settings definition 与专用 editor，但必须：

- namespaced path；
- stable owner/source；
- schema version；
- unload 后保留 config unknown key；
- UI group 不得无限增加一级 tab，优先挂到现有 group/Extensions 子页；
- extension editor crash 回退为 read-only JSON view，不拖垮 settings UI。

## 14. TUI 交互

- `/` 或直接 typing 搜索；
- Tab 在 group/list/detail 间切换；
- Up/Down navigation；
- Enter edit/open；
- Space toggle 仅 boolean；
- Ctrl+S save；
- Esc 返回/cancel；
- `?` contextual help；
- mouse click/scroll 等价；
- unsaved changes 离开时 prompt；
- status line 显示 scope、dirty、validation count。

## 15. 与 CLI/docs 共享

同一 registry 生成：

- `pi config list/get/set/unset/explain`；
- `/settings`；
- JSON Schema；
- docs table；
- migration validation；
- autocomplete。

建议 `unset` 与 `reset-to-default` 分开，避免覆盖语义不清。

## 16. 包归属建议

- setting definition/resolution/validation：`packages/coding-agent` 现阶段；
- generic schema-driven settings components：`packages/tui`；
- 若 server/client 也编辑同一 config，再提取 `@own/pi-config-schema`；
- 不建议独立“settings UI package”，因为 TUI 与 browser adapter 不共享渲染。

## 17. 验收场景

1. 查到 retry setting 并看清 project override；
2. 删除 project override 后恢复 global value；
3. CLI runtime override 显示 read-only；
4. invalid number 阻止保存；
5. fallback chain reorder 并模拟 resolution；
6. MCP connection test failure 有 actionable error；
7. theme preview cancel 后恢复；
8. unknown future key round-trip 不丢失；
9. 40 列窄终端完整操作；
10. 全程键盘可达且颜色非唯一状态。

## 18. 资料来源

- 当前 settings docs：`packages/coding-agent/docs/settings.md`
- 当前 selector：`packages/coding-agent/src/modes/interactive/components/settings-selector.ts`
- 当前 generic list：`packages/tui/src/components/settings-list.ts`
- 当前 manager：`packages/coding-agent/src/core/settings-manager.ts`
- OMP settings：https://github.com/can1357/oh-my-pi/blob/main/docs/settings.md
- OpenCode config：https://opencode.ai/docs/config/
- Codex config reference：https://developers.openai.com/codex/config-reference
