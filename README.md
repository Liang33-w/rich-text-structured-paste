# rich-text-structured-paste

> Paste markdown or HTML into a web rich text editor — and keep the headings, tables and lists.

An Agent Skill (`SKILL.md` + `references/`) for Cangjie / ProseMirror / Quill / WangEditor style editors.

## 问题 / The problem

把一份结构化文档粘进网页富文本编辑器，结果常常是：字都进去了，但全并成一个段落；`## 标题` 原样显示成字面量；表格变成一串 `|`；或者新内容被塞进旧表格的某个单元格里。

You paste a structured document into a rich text editor and the text arrives — but flattened into a single paragraph, with `## Heading` shown literally and tables collapsed into a row of pipes.

## 结论 / The rule

> 在富文本编辑器里，**只有真实剪贴板 + 真实按键**能保住块结构。
> 所有"绕过剪贴板"的注入方式（`execCommand('insertHTML')`、合成 `ClipboardEvent('paste')`、直接改 `innerHTML`）在成熟编辑器上都会被 schema 过滤拍平或直接忽略。

Only a real clipboard write plus a real key press preserves block structure. Every shortcut that bypasses the clipboard is silently flattened or ignored by mature editors — usually without throwing an error, which is why the failure goes unnoticed.

## 内容 / What's inside

```
SKILL.md                       铁律 + 探测步骤 + 主路径 + 禁止事项
references/editor-probe.md     探测代码：定位编辑器、识别内核、块类型统计、焦点检查
references/failure-modes.md    9 种已复现失败模式的现象 / 根因 / 处置
```

## 覆盖的失败模式 / Failure modes covered

| # | 现象 | 根因 |
|---|---|---|
| 1 | 出现字面量 `##`、`**bold**`、`\| col \|` | 走了纯文本通道 |
| 2 | 字在，结构没了 | `execCommand` 被 schema 拍平 |
| 3 | 合成 paste 事件完全无反应 | 编辑器校验 `isTrusted` |
| 4 | `<h2>` 变成可见文字 | 逐字符输入 HTML |
| 5 | 旧表格还在，新表格套进去 | Ctrl+A 在单元格内只选中单元格 |
| 6 | 清空后标题/分类/标签也没了 | `reload()` 清掉全部表单状态 |
| 7 | `NotAllowedError` 写剪贴板失败 | 页面失焦或非安全上下文 |
| 8 | 点击报 `element not stable` | 元素有动画位移，改用 DOM click |
| 9 | `require is not defined` | 沙箱禁止模块加载，HTML 需内联 |

## 关键方法 / Key techniques

- **忠实验证**：不看文字、不看截图，只看块类型计数（`h2` 为 0 而 `p` 暴涨 = 被拍平）。
- **清空顺序**：`reload()` → 粘正文 → 最后填表单。反过来会白填。
- **内容侧**：只用语义标签（`h2`/`table`/`ul`/`pre`），内联样式一律会被编辑器丢弃。

## 安装 / Install

作为 Agent Skill 使用，把本仓库内容放进对应框架的 skills 目录：

```bash
git clone https://github.com/Liang33-w/rich-text-structured-paste.git
# QwenPaw:  $QWENPAW_WORKING_DIR/workspaces/{agent_id}/skills/
# Claude / compatible: ~/.claude/skills/ (or your framework's skills dir)
```

`SKILL.md` 遵循通用 Agent Skills 格式（YAML front matter + Markdown 正文），可直接在兼容框架中复用。

## 许可 / License

Apache License 2.0 — see [LICENSE](LICENSE).
