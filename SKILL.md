---
name: rich-text-structured-paste
description: "把 markdown 或 HTML 结构化内容无损写入网页富文本编辑器（Cangjie、飞书文档、腾讯文档、WangEditor、ProseMirror 一类），保住标题层级、表格、有序/无序列表、代码块。当用户要求「把这篇发到某个平台」「帮我在这写一篇」「粘贴进去格式全没了」「markdown 变纯文本了」「标题和表格要保留」时使用。Trigger especially when user mentions: paste markdown into rich text editor, formatting lost on paste, heading became plain text, table collapsed, Cangjie editor, ProseMirror, contenteditable, insertHTML, 富文本粘贴, 格式丢失, 表格粘不进去, 编辑器清空。"
metadata:
  requires:
    bins: []
---

# 富文本结构化粘贴

把一份结构化文档放进网页富文本编辑器，并保住它原有的块结构。

失败的样子很统一：字都进去了，但全并成一个段落；`## 标题` 原样显示成字面量；表格变成一串 `|`；贴完后旧内容还在，新内容被塞进旧表格的某个单元格里。

**这个 Skill 的价值不在于"会粘贴"，而在于不浪费时间试那三种根本不通的路。**

## 铁律

> 在富文本编辑器里，**只有真实剪贴板 + 真实按键**能保住块结构。
> 所有"绕过剪贴板"的注入方式（execCommand、合成 paste 事件、直接改 innerHTML）在成熟编辑器上都会被 schema 过滤拍平或直接忽略。

如果一个方法试了一次没成功，不要微调参数再试第二次 —— 换通道。同一通道调参是无效功。

## 第 0 步：前置检查

动手前确认三件事，任何一条不满足就先解决它：

1. **编辑器已就绪** —— 页面加载完，正文区可见可聚焦（`[contenteditable="true"]` 存在且 `isContentEditable` 为 true）。
2. **目标页已登录** —— 发布页需要登录态。未登录时粘贴会成功但提交失败，白忙一场。
3. **内容已渲染成 HTML 字符串** —— 不是在内存里想象的结构，而是一段可以直接喂给 `ClipboardItem` 的 HTML。标题用 `<h2>`/`<h3>`，表格用 `<table><thead><tr><th>`，列表用 `<ul>/<ol>`，代码用 `<pre><code>`。

## 第 1 步：探测编辑器（30 秒，别省）

不同编辑器的落点位置不同。探测顺序：

```
1. 有没有 iframe？    → 编辑器在 iframe 里，先 frameLocator 进去，否则一切选择器失效
2. 有没有 shadow DOM？→ 用 evaluate 穿透 shadowRoot 找 contenteditable
3. contenteditable 在哪？
   - [data-cangjie-editable]      → 字节 Cangjie 内核（飞书同源）
   - .ProseMirror / [data-pm-slice] → ProseMirror（Notion 类、腾讯文档新版）
   - .ql-editor                    → Quill
   - .w-e-text-container           → WangEditor
   - 纯 textarea                  → 不是富文本，直接填 value，本 Skill 不适用
```

探测手段：用浏览器自动化工具的 evaluate，一次性把这些信息捞出来，不要反复取快照。

需要捞的字段：
- `document.querySelectorAll('iframe').length`
- 所有 `[contenteditable]` 的 `tagName`、`className`、`data-*` 属性、`isContentEditable`
- 正文区当前的**块类型统计**（每种标签各多少个），作为粘贴前的基线

`references/editor-probe.md` 里有可直接复用的探测代码。

## 第 2 步：走唯一正确的主路径

### 2.1 把 HTML 写进系统剪贴板

```js
const html = '<h2>标题</h2><p>正文</p><table>…</table>';
const item = new ClipboardItem({
  'text/html':  new Blob([html], {type: 'text/html'}),
  'text/plain': new Blob([html.replace(/<[^>]+>/g, '')], {type: 'text/plain'}),
});
await navigator.clipboard.write([item]);
```

要点：
- **`text/html` 和 `text/plain` 两个 flavor 都要给。** 只给 html，部分编辑器会退化成按纯文本处理；只给 plain，结构必丢。
- `navigator.clipboard.write` 在浏览器扩展 / 真实用户手势上下文中**通常无需额外授权**即可成功。如果报 `NotAllowedError`，说明当前不是安全上下文或页面失焦，先点击正文区再重试一次。
- **不能靠读本地文件再拼**：某些自动化沙箱（如 Playwright 扩展的 code VM）里 `require` 和动态 `import()` 都不可用，HTML 必须作为字符串内联在待执行的代码里。

### 2.2 让编辑器获得焦点，再发真实按键

```js
await page.click('[contenteditable="true"]');   // 或 evaluate 里 element.focus()
await page.keyboard.press('Control+V');          // macOS: Meta+V
```

要点：
- **必须是真实按键事件**（走浏览器输入通道），不是 `dispatchEvent(new KeyboardEvent('keydown'))`。
- 粘贴前光标要落在**正文空白处**，不能落在已有表格的单元格里 —— 否则新表格会嵌进旧表格。见 2.3。
- 粘贴大表格（>50 行）后要等一秒再继续，编辑器是异步解析的，立刻读 DOM 会读到中间态。

### 2.3 先清空，再粘贴

这是最容易翻车的一步。

**不要用"点击正文 → Ctrl+A → Delete"。** 光标落在表格内时，Ctrl+A 只选中当前单元格，删掉的是单元格内容，旧表格壳还在，接着粘贴就变成表格套表格。

可用做法，按优先级：
1. **重新加载页面**（`page.reload()`）—— 最干净。代价是标题、分类、标签等表单字段也会被清空，需要重新填。如果正文很长、表单很多，先填表单再 reload 会白填，所以**顺序应该是：先 reload 清空 → 再粘正文 → 最后填表单**。
2. 找到编辑器的"清空/重置"按钮，如果有。
3. 用 evaluate 选中正文区所有子节点逐个 remove，再 `dispatchEvent(new InputEvent('input', {bubbles:true}))` 通知框架状态变更。仅在前两条不可行时用，部分编辑器会拒绝外部 DOM 变更。

### 2.4 粘完立刻验证块类型

不要靠肉眼看截图。读 DOM，对比第 1 步的基线：

```
粘前：h2=0, table=1(旧), p=3
粘后：h2=3, table=2, p=12    ← 数量对上才算成功
```

如果 `h2` 还是 0 而段落数暴涨 —— 说明被拍平了，回到"铁律"：换通道，别再调参。

`references/failure-modes.md` 里有完整的现象→根因→处置对照表。

## 第 3 步：提交并确认落地

1. 填完其余必填字段（标题、分类、标签、勾选框）。**注意 reload 会清空它们，所以放在粘贴之后填。**
2. 提交按钮常常是动态渲染的，且会因列表抖动导致点击超时。用 evaluate 按文案找元素并 `element.click()`，比坐标点击稳。
3. 提交后**不要只看有没有跳转**。回到列表页或详情页，读 `document.body.innerText`，确认刚写的特征句（一句只属于这篇的短语）出现在页面上。找不到就是没发出去。

## 禁止事项

- ❌ `document.execCommand('insertHTML', ...)` —— 在 Cangjie / ProseMirror 上会被拍平成单个段落，且不报错，最容易误判成功。
- ❌ 构造 `new ClipboardEvent('paste', ...)` 并 dispatch —— 编辑器普遍校验 `isTrusted`，非可信事件被静默忽略。
- ❌ 直接 `innerHTML = ...` —— 框架状态与 DOM 不同步，下一次输入就会被回滚。
- ❌ 用 `browser_type` / `fill` 往 contenteditable 里写整段 HTML —— 它逐字符输入，`<h2>` 会变成字面文字。
- ❌ 粘完不验证就提交 —— 拍平后的内容看起来"有字"，只有块类型统计能揭穿它。

## 内容侧注意

编辑器会做 schema 过滤，**只保留语义块，丢弃大部分样式**：

- 保留：`h1`-`h4`、`p`、`ul`/`ol`/`li`、`table`/`thead`/`tbody`/`tr`/`th`/`td`、`pre`/`code`、`strong`/`em`、`a`、`blockquote`、`hr`
- 丢弃：内联 `style`、`class`、`div` 嵌套、`font`、自定义属性、`<script>`
- 别指望靠 `<div style="font-size:24px">` 造标题 —— 一定会变成正文字号。要标题就用 `<h2>`。

所以：**生成 HTML 时只用语义标签，不要做视觉排版。** 视觉由编辑器主题决定。

## 参考

- `references/editor-probe.md` —— 可复用的探测代码（定位编辑器 + 统计块类型）
- `references/failure-modes.md` —— 9 种已复现失败模式的现象、根因、处置
