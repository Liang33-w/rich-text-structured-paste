# 失败模式对照表

以下 9 种都是**已复现**的失败，不是推测。按"试过一次就换路"的原则使用：同一行出现即按处置执行，不要微调参数重试。

## 速查

| # | 现象 | 根因 | 处置 |
|---|---|---|---|
| 1 | 粘完出现字面量 `## 标题`、`**加粗**`、`\| 列 \|` | 按纯文本通道粘贴，编辑器没做 markdown 解析 | 改用 `text/html` flavor，见主路径 2.1 |
| 2 | 字都在，但全并成一个段落，标题和表格没了 | `execCommand('insertHTML')` 被 schema 过滤拍平 | 弃用 execCommand，走真实剪贴板 |
| 3 | 合成 paste 事件完全没反应，也不报错 | 编辑器校验 `event.isTrusted === false` | 用真实按键事件（`keyboard.press`） |
| 4 | `<h2>` 变成文字 `<h2>` 显示出来 | 用逐字符输入（type/fill）往 contenteditable 写 HTML | 只能走剪贴板，不要 type |
| 5 | 清空后旧表格还在，新表格套在里面 | Ctrl+A 在表格内只选中当前单元格 | 重新加载页面清空，见主路径 2.3 |
| 6 | 清空后标题、分类、标签也空了 | `reload()` 清空全部表单状态 | 调整顺序：先清空 → 再粘正文 → 最后填表单 |
| 7 | `NotAllowedError: Write permission denied` | 页面失焦，或不在安全上下文 | 先点正文区让页面获得焦点，再重试一次 |
| 8 | 点击提交报 `element not stable` 超时 | 列表/卡片有动画，元素位置持续变化 | 用 evaluate 按文案定位并 `element.click()` |
| 9 | 自动化代码里 `require is not defined` | 沙箱禁止模块加载（Playwright 扩展 code VM） | HTML 内联为字符串；或用支持 `filename` 参数的接口加载本地 js，但 js 内部仍不能读文件 |

## 展开

### #1 纯文本粘贴 —— 最常见

把 markdown 原文直接喂给编辑器，结果 `##` 和 `|` 原样显示。原因很简单：编辑器拿到的是 `text/plain`，它没有理由去猜这是 markdown。

也有编辑器（如带 markdown 快捷输入的）会在**输入时**把 `## ` 转成标题，但**粘贴时不会**。这两条路径不同，别混。

**判定**：粘贴后 DOM 里 `h2` 计数为 0，而 `p` 数量等于原文段落数。

### #2 execCommand 拍平

`document.execCommand('insertHTML', false, html)` 在旧式编辑器上有效，在 Cangjie / ProseMirror 上表现为：内容进去了，但只剩一个 `<p>`，里面是纯文本。**不抛异常**，所以极容易误判成功。

判断依据同上：块类型统计。

### #3 合成 paste 事件被忽略

```js
// ❌ 一定不生效
const dt = new DataTransfer();
dt.setData('text/html', html);
el.dispatchEvent(new ClipboardEvent('paste', {clipboardData: dt, bubbles: true}));
```

成熟编辑器会检查 `isTrusted`。合成事件的 `isTrusted` 恒为 `false`。这条路没有任何绕过办法 —— 不要试图构造更"真"的事件对象。

### #4 逐字符输入

`type` / `fill` 类接口本质是逐字符敲键盘，`<` 就是 `<` 字符。HTML 永远不会被解析成标签。

**唯一例外**：`fill` 在某些实现里对 input/textarea 走的是直接赋值，但那不是富文本场景。

### #5 / #6 清空的顺序陷阱

这两个必须一起看，因为它们互为陷阱：

- 用 Ctrl+A 清空 → 表格场景下清不干净（#5）
- 用 reload 清空 → 干净了，但表单也空了（#6）

**正确顺序**：

```
1. 打开/刷新页面
2. 立刻粘正文（此时表单是空的，粘完不用管）
3. 填标题、选分类、加标签、勾选协议
4. 提交
```

反过来的顺序（先填表单再 reload）会白填一遍。

### #7 剪贴板权限

`navigator.clipboard.write` 需要：安全上下文（https 或 localhost）+ 文档处于焦点。

浏览器扩展模式下 `grantPermissions` 往往不可用（`Browser.grantPermissions not found`），但**这不妨碍 `clipboard.write` 成功** —— 它是一个独立的用户手势 API，不走 Playwright 的权限系统。

如果确实报 `NotAllowedError`：先 `click` 正文区（触发一次真实用户手势），再立刻写入。仍失败则说明页面在后台标签页，切到前台。

### #8 元素不稳定

列表页卡片的点击常报 `element not stable`（等待 5s 超时），因为它有 hover 动画或懒加载位移。

```js
// 稳定写法
() => {
  const el = [...document.querySelectorAll('a,button,div')]
    .find(x => (x.innerText || '').trim() === '目标文案');
  if (!el) return 'not found';
  el.click();
  return 'clicked';
}
```

`element.click()` 是同步 DOM 调用，不等待可操作性检查，因此不受动画影响。

### #9 沙箱里的模块加载

Playwright 扩展的 code VM 里：`require` 未定义、动态 `import()` 不可用、`fs` 不可达。所以**不能"先读本地 md 文件再转换"** —— HTML 必须已经作为字符串存在于待执行代码中。

可行的替代：
- 支持 `filename` 参数的接口，把长代码写进本地 `.js` 文件再加载，规避长字符串转义地狱。但注意：**js 文件内部依然不能读文件**，只能调用页面 API。
- HTML 生成放在 agent 侧完成（用文件系统工具读 md、转换），只把最终 HTML 字符串传给浏览器。

## 一条总原则

所有失败模式里，**只有 #1、#2、#3、#4 是"通道错误"**，其余都是操作顺序或环境问题。

通道错误的表现完全一致：**块结构丢失但文字完整**。

所以验证方式也应该只有一种：**看块类型计数，不看文字**。文字对而结构错，是这个领域最典型的假成功。
