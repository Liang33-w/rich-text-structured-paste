# 编辑器探测代码

以下代码在浏览器页面上下文执行（evaluate 或等价能力）。分三段，各自独立可用。

## 1. 一次性摸清编辑器在哪

```js
() => {
  const ce = [...document.querySelectorAll('[contenteditable]')].map(el => ({
    tag: el.tagName,
    cls: (el.className || '').toString().slice(0, 120),
    dataAttrs: [...el.attributes]
      .filter(a => a.name.startsWith('data-'))
      .map(a => `${a.name}=${a.value.slice(0, 40)}`),
    editable: el.isContentEditable,
    textLen: (el.innerText || '').length,
    childCount: el.children.length,
  }));

  return JSON.stringify({
    url: location.href,
    iframes: document.querySelectorAll('iframe').length,
    shadowHosts: [...document.querySelectorAll('*')]
      .filter(el => el.shadowRoot).length,
    textareas: document.querySelectorAll('textarea').length,
    contentEditables: ce,
  }, null, 1);
}
```

判读：

| 结果 | 含义 |
|---|---|
| `iframes > 0` 且 `contentEditables` 为空 | 编辑器在 iframe 里。后续所有选择器都要先进入 frame，否则永远选不中 |
| `shadowHosts > 0` 且选不到 contenteditable | 编辑器在 shadow DOM 里。需要从 host 的 `shadowRoot` 往下找 |
| `textareas > 0` 且 contentEditables 为空 | 不是富文本编辑器。直接赋值 `value` 并触发 `input` 事件即可，本 Skill 不适用 |
| contentEditables 有多条，且 `textLen` 都很大 | 可能同时存在标题框和正文框。用 `textLen` 最大的那个当正文区，不要用第一个 |

## 2. 认编辑器内核

拿到 `dataAttrs` / `cls` 后对照：

| 特征 | 内核 | 行为特点 |
|---|---|---|
| `data-cangjie-editable` | 字节 Cangjie（飞书同源） | 严格 schema 过滤；`insertHTML` 会被拍平；忽略非可信 paste 事件 |
| `ProseMirror` / `[data-pm-slice]` | ProseMirror | 基于事务；外部 DOM 改动会被下一次事务回滚 |
| `ql-editor` | Quill | 支持部分粘贴解析，但对嵌套表格敏感 |
| `w-e-text-container` | WangEditor | 对 `insertHTML` 宽容度较高，仍建议走剪贴板 |
| Slate 特征（`data-slate-editor`） | Slate | 完全虚拟 DOM；任何外部 DOM 操作都会被覆盖 |

**只有 Cangjie 和 ProseMirror 需要严格按主路径走。** 其余可以尝试 `insertHTML`，但如果第一下没成，立刻回主路径 —— 别调参。

## 3. 块类型统计（粘贴前后对比用）

```js
(selector = '[contenteditable="true"]') => {
  const root = document.querySelector(selector);
  if (!root) return JSON.stringify({error: 'editor not found'});
  const tags = ['h1','h2','h3','h4','p','ul','ol','li','table','tr','th','td','pre','code','blockquote','hr','strong','a','img'];
  const count = {};
  tags.forEach(t => count[t] = root.querySelectorAll(t).length);
  // 顶层块序列，用来判断结构是否被打平
  const topLevel = [...root.children].map(c => c.tagName.toLowerCase()).slice(0, 60);
  return JSON.stringify({
    count,
    topLevelCount: root.children.length,
    topLevel,
    textLen: (root.innerText || '').length,
  }, null, 1);
}
```

判读要点：

- **`h2`/`h3` 计数为 0，而 `p` 暴涨** → 标题被拍平，粘贴通道错了。
- **`topLevelCount === 1`** → 全部内容挤进一个容器，几乎必然失败。
- **`table` 计数比预期多 1** → 旧内容没清干净，新表格粘进了旧表格（检查 `th`/`td` 总数是否溢出）。
- **`textLen` 对但块结构不对** → 典型的"看起来成功了"，只有块统计能抓到。

## 4. 焦点状态检查

粘贴前跑一次，确认光标真的在正文里：

```js
() => {
  const el = document.activeElement;
  return JSON.stringify({
    active: el ? el.tagName + '.' + (el.className || '').toString().slice(0, 60) : null,
    isEditable: el ? el.isContentEditable : false,
    inEditor: el ? !!el.closest('[contenteditable="true"]') : false,
    inCell: el ? !!el.closest('td, th') : false,   // true = 会嵌进旧表格
  }, null, 1);
}
```

`inCell: true` 时**不要粘贴**。先把光标移到正文末尾（`Ctrl+End`）或重新加载页面。
