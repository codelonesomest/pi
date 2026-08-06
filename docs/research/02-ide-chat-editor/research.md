# IDE 级聊天编辑器研究

- 状态：可行性与架构建议
- 更新日期：2026-07-31
- 范围：TUI prompt editor 的选择、光标、鼠标、编辑命令、Unicode 与终端能力降级
- 核心结论：可以接近 IDE 编辑器体验，但不能承诺所有终端都完整支持 Command/Super 组合键；必须做 capability-tiered 设计

## 1. 期望体验

用户希望聊天输入框不再只是多行 prompt input，而是一个完整编辑器：

- Shift+方向键扩展选择；
- Command/Ctrl+Shift+方向键按行、词或文档扩展选择；
- 鼠标点击改变光标；
- 鼠标拖拽选择；
- 双击选词、三击选行；
- copy/cut/paste、undo/redo；
- word、line、document navigation；
- CJK/emoji/组合字符正确；
- 软换行下的视觉行移动；
- 与 autocomplete、IME、外部编辑器兼容。

## 2. 当前 Pi 基线

当前 `packages/tui` 已有大量正确基础：

- `Editor` 使用 grapheme 和 word segmenter；
- 支持多行、软换行、undo stack、kill ring、yank、word navigation、autocomplete；
- `Focusable` 与 `CURSOR_MARKER` 为 IME 候选窗口定位硬件光标；
- key parser 同时支持 legacy terminal sequences 与 Kitty keyboard protocol；
- macOS 有 native modifier helper，可读取 Shift/Command/Control/Option 状态；
- alt-screen 已启用 SGR mouse tracking，支持 transcript hyperlink、拖选与滚动；
- keybindings 已使用 namespaced action ID，可配置 cursor、delete、undo、copy 等操作。

相关实现：

- `packages/tui/src/components/editor.ts`
- `packages/tui/src/keys.ts`
- `packages/tui/src/keybindings.ts`
- `packages/tui/src/native-modifiers.ts`
- `packages/tui/src/TuiAltScreen.ts`
- `packages/coding-agent/docs/keybindings.md`

当前公开按键动作覆盖 cursor/deletion/undo/yank，但没有形成完整的 editor selection command surface。alt-screen 的鼠标拖选目前主要服务渲染后的 transcript，不等于可编辑文本中的选择模型。

## 3. 终端现实约束

### 3.1 键盘协议

传统终端输入无法可靠区分多种组合：

- Shift+Enter 与 Enter；
- Ctrl+Shift 与 Ctrl；
- Command/Super/Hyper；
- key press、repeat 与 release。

Kitty keyboard protocol/CSI-u 能报告完整 modifier bitmask，并减少 legacy 控制字符歧义。当前 Pi 已支持 Kitty protocol，这是实现高级组合键的首选路径。

但是：

- 并非所有 terminal、tmux 版本、SSH 链路都完整透传；
- macOS Command 通常先被 terminal application 消费；
- Windows/Linux 的 Super 也可能被桌面环境消费；
- native modifier polling 可以辅助鼠标事件，但不应作为跨平台键盘协议替代品。

因此 UI 必须显示实际 capability，而不能文档里统一承诺 `Command+Shift+Arrow`。

### 3.2 建议兼容等级

| 等级 | 条件 | 能力 |
|---|---|---|
| A | Kitty keyboard + SGR mouse | 完整 modifier、press/repeat/release、鼠标编辑 |
| B | CSI-u/modifyOtherKeys + SGR mouse | 大部分 Ctrl/Alt/Shift 组合与鼠标编辑 |
| C | legacy input + SGR mouse | 基础 Shift 选择可能依赖终端映射，鼠标可用 |
| D | legacy/no mouse | 纯键盘基础编辑，提供可配置替代绑定 |

启动后可在 `/diagnostics` 或 editor help 中显示当前等级和缺失能力。

## 4. 编辑器状态模型

不要在每个 command 内直接拼字符串和改 cursor。建议明确：

```ts
interface EditorState {
  document: TextDocument;
  selection: Selection;
  preferredVisualColumn: number | null;
  viewport: EditorViewport;
  composition?: ImeComposition;
}

interface Selection {
  anchor: TextOffset;
  active: TextOffset;
  affinity: "forward" | "backward";
}
```

关键规则：

- 无选择时 `anchor === active`；
- 所有 move command 接受 `extend: boolean`；
- `Shift` 只改变 active，保留 anchor；
- 非 extend navigation 折叠已有选择到对应边缘；
- insert/delete 先替换 selection；
- undo transaction 同时恢复 document、selection 与 viewport intent；
- offset 必须落在 grapheme boundary；
- logical offset 与 wrapped visual row/column 由 layout projection 转换。

## 5. 文本存储选择

不建议第一版直接引入 rope/piece table。聊天 prompt 通常小于代码文件，当前字符串模型更简单。

建议分层：

1. `TextDocument`：字符串、line starts、grapheme boundary cache；
2. `EditorSelectionModel`：anchor/active 与 edit transaction；
3. `EditorLayout`：宽度、软换行、tab、wide grapheme 到 screen cells 的映射；
4. `EditorController`：commands；
5. `EditorView`：渲染、光标和选择样式。

只有 profile 证明大 paste 或长 prompt 的 edit latency 不可接受，才替换 `TextDocument` 内部结构。接口不应暴露 rope 实现。

## 6. Command 体系

所有行为注册为 namespaced editor action，禁止在组件里硬编码按键：

```text
tui.editor.moveLeft
 t ui.editor.moveWordLeft
 t ui.editor.moveLineStart
 t ui.editor.moveVisualLineStart
 t ui.editor.moveDocumentStart
 t ui.editor.selectLeft
 t ui.editor.selectWordLeft
 t ui.editor.selectLineStart
 t ui.editor.selectDocumentStart
 t ui.editor.selectAll
 t ui.editor.copy
 t ui.editor.cut
 t ui.editor.paste
 t ui.editor.redo
```

实际命名需清理上面的展示空格，并沿用现有 `tui.editor.*` namespace。

更优做法是一个 command + `extend` 参数，不在实现内部复制移动算法；keybinding registry 可以把 action ID 映射为参数化 command。

### 推荐默认映射

| 行为 | macOS/现代终端 | 通用替代 |
|---|---|---|
| 逐字符选择 | Shift+Left/Right | 可配置 |
| 逐词选择 | Option+Shift+Left/Right | Ctrl+Shift+Left/Right |
| 行首/尾选择 | Command+Shift+Left/Right | Shift+Home/End |
| 文档首/尾选择 | Command+Shift+Up/Down | Ctrl+Shift+Home/End |
| 全选 | Command+A | Ctrl+A 可能与 Emacs 行首冲突，需 profile |
| copy/cut/paste | Command/Ctrl+C/X/V | terminal clipboard fallback |
| undo/redo | Command/Ctrl+Z、Shift+Command/Ctrl+Z | 现有 undo binding 保留迁移 |

默认值应按 OS + capability profile 生成，而不是单一常量覆盖所有平台。

## 7. 鼠标编辑

### 7.1 事件路由

alt-screen 已有 mouse tracking。需要在布局 hit-test 后按优先级路由：

1. overlay；
2. focused editor；
3. scroll view/transcript；
4. terminal fallback。

editor 获得：

- press：设置 anchor/active；
- drag：扩展 active；
- release：完成 selection；
- double click：按 word segmenter 选词；
- triple click：选 logical line；
- Shift+click：从现有 anchor 扩展；
- edge drag：滚动 editor viewport，而不是 transcript。

### 7.2 screen coordinate → text offset

必须共享渲染所用的 layout projection：

```ts
interface EditorLayoutFrame {
  rows: VisualRow[];
  pointToOffset(x: number, y: number, bias: "nearest" | "left"): TextOffset;
  offsetToPoint(offset: TextOffset): { x: number; y: number };
}
```

不能根据原始字符串 length 猜列；wide CJK、emoji、ZWJ sequence、combining marks 都会出错。

## 8. 选择渲染

- 使用独立 selection style，不反转整行；
- active cursor 在 selection 边缘仍可见；
- inactive/focus-lost selection 使用弱化样式；
- selection 跨软换行时按视觉 segment 绘制；
- 空行选择需要至少一个 cell 的反馈；
- 复制输出使用原始 logical text，不包含软换行；
- bracketed paste marker 是内部显示策略时，复制/选择必须有清楚规则，不能泄露不可逆 placeholder。

## 9. Clipboard 与 terminal 交互

建议 clipboard adapter 顺序：

1. native clipboard package；
2. OSC 52（有安全/长度配置）；
3. platform command adapter；
4. internal clipboard fallback + status notice。

不要假设 Ctrl+C 总是 copy：无选择时它目前还承担 clear/interrupt 语义。建议：

- editor 有选择：copy；
- 无选择且 agent running：interrupt；
- 无选择且 idle、editor 非空：现有 clear 行为；
- 这条 precedence 必须写入 keybinding help。

## 10. IME 与 composition

第一版仍可以沿用 terminal 自己完成 composition、应用只接收 committed text 的模式。必须保持：

- hardware cursor 与 fake cursor 坐标一致；
- overlay/container 正确转发 focus；
- selection replacement 在 committed text 到达时是一个 undo transaction；
- composition 期间不因 background render 抢 focus。

如果未来 terminal protocol 能提供 preedit，再新增 `ImeComposition`，不要先模拟一个不可靠的 preedit layer。

## 11. Autocomplete、历史与外部编辑器

- autocomplete popup 的选择不能覆盖 editor selection；
- Up/Down 在 popup 开启、multi-line visual movement、history navigation 三者间要有明确 precedence；
- external editor 返回时作为一个 replace-document undo transaction；
- history item 放入 editor 后默认无选择，cursor 在尾部；
- file mention/command completion 应针对 selection replacement 正常工作。

## 12. 性能目标

建议以用户可感知指标定义：

- 10k grapheme prompt 的字符插入 p95 < 8ms；
- resize + rewrap p95 < 16ms（80×40 测试窗口）；
- drag selection render 不低于 30fps；
- 100k 字符 paste 可以完成且 UI 不冻结超过 100ms；
- 每次 keypress 不扫描全部 document，除非 profile 证明规模足够小且更简单实现更快。

## 13. 验收矩阵

必须覆盖：

- ASCII、CJK、emoji、ZWJ family emoji、combining accent；
- hard newline 与 soft wrap；
- tab 与宽字符点击；
- Shift 选择、word/line/document selection；
- backward selection；
- replace selection、undo、redo；
- mouse click/drag/double/triple click；
- terminal resize during selection；
- IME candidate placement；
- Kitty、legacy、tmux、SSH 降级；
- macOS、Linux、Windows Terminal。

## 14. 包归属建议

第一版应留在 `@earendil-works/pi-tui`（未来 fork scope 对应的 TUI package）：

- editor 是 TUI 的深模块；
- 它与 terminal input、layout、Unicode width、focus 强耦合；
- 单独 `pi-editor` package 会暴露大量浅接口并增加同步成本。

只有当 browser/desktop editor 也要共享纯文本 state machine，才提取一个无 terminal 依赖的 `editor-core`；此时 TUI 是第二个真实 adapter，独立 seam 才成立。

## 15. 建议实现顺序

1. selection state + command semantics；
2. keyboard selection；
3. selection rendering；
4. clipboard precedence；
5. editor hit-test + click；
6. drag/double/triple click；
7. capability help 与降级 profile；
8. performance profile 与大文本优化。

## 16. 资料来源

- 当前 Editor：`packages/tui/src/components/editor.ts`
- 键盘解析：`packages/tui/src/keys.ts`
- modifier helper：`packages/tui/src/native-modifiers.ts`
- alt-screen mouse：`packages/tui/src/TuiAltScreen.ts`
- 当前 keybindings：`packages/coding-agent/docs/keybindings.md`
- Kitty keyboard protocol：https://sw.kovidgoyal.net/kitty/keyboard-protocol/
- Pi TUI 文档：`packages/coding-agent/docs/tui.md`
